# Pragmatic Email Validator

Validates an email address meets pragmatic format and length expectations. By design, rejects RFC-valid emails that are unlikely to be appear in the wild. 

Does not attempt to verify associated inbox.

## 🧐 How this differs from RFC compliance

Strict RFC 5322 compliance is practically useless for modern web applications. 

The official email specifications (RFC 5322 & 5321) are extremely permissive, allowing formats that no major mail provider actually supports. If you use a strict RFC validator, you will accept garbage data that will hard-bounce when you try to send mail.

**This library is "Pragmatic"**: it validates what *actual* email providers accept, not what a text file from 1982 says is theoretically possible.

| Feature | RFC 5322 Strict | This Library (Pragmatic) |
| :--- | :--- | :--- |
| **Quoted Local Parts** | Allowed (`" "@example.com`) | ❌ Rejected |
| **IP Address Domains** | Allowed (`user@[127.0.0.1]`) | ❌ Rejected |
| **Comments** | Allowed (`user(comment)@example.com`) | ❌ Rejected |
| **Consecutive Dots** | Allowed in quotes | ❌ Rejected |
| **Local Part Length** | Up to 64 chars | ✅ Checks widely (usually 64) |
| **Total Length** | Up to 320 chars | ⚠️ Warns > 254 (Path limit) |
| **TLD Requirement** | Optional (`user@localhost`) | ✅ Required (e.g. `.com`) |

**Example:**
The address `"very.(),:;<>[]\".VERY.\"very@\ \"very\".unusual"@strange.example.com` is **valid** according to the RFC, but rejected by Gmail, Outlook, and effectively every sending service on earth. This library returns `false`.

## Pragmatic Email Validation by Vendors

Major providers (Gmail, Microsoft, Apple), sending services (SendGrid, AWS SES), and authentication systems that rely on email (Auth0, Okta) all enforce rules stricter than the RFCs.

### 1. Consumer Email Services

  * **Gmail**:
      * **Strictness**: Ultra-pragmatic.
      * **Dots**: Ignores them in the local part (`jane.doe` == `janedoe`).
      * **Case**: Insensitive.
      * **Blocklist**: Rejects consecutive dots (`..`), leading/trailing dots, and `&`, `=`, `_`, `'`, `-`, `+`, `,` at the *start* of a username.
      * **Source**: [Gmail Username Rules](https://support.google.com/mail/answer/9211434)
  * **Microsoft (Outlook/Hotmail/Live)**:
      * **Strictness**: High.
      * **Format**: Must start with a letter.
      * **Restricted**: Does not allow underscores `_` in the domain (violates strict RFC but common in pragmatic checks).
      * **Aliases**: Supports `+` addressing (e.g. `name+tag@outlook.com`) but implementation is spotty in older Exchange servers.
  * **Apple (iCloud/Me.com)**:
      * **Strictness**: High.
      * **Local Part**: Must start with a letter.
      * **Quoted Strings**: Hard rejection.

### 2. Enterprise Identity Services

These services sit in front of millions of corporate logins. If they reject it, the user can't sign up.

  * **Auth0**:
      * **Length**: Enforces a total length of **254 characters** (not 320).
      * **Validation**: explicitly uses a "pragmatic" regex that forbids IP domains and quoted local parts.
      * **Source**: [Auth0 User Profile Rules](https://www.google.com/search?q=https://auth0.com/docs/get-started/auth0-overview/create-users)
  * **Okta**:
      * **Restricted Chars**: Explicitly forbids most special characters in the local part except `.` `_` `-` `+`.
      * **Source**: [Okta Attribute Rules](https://www.google.com/search?q=https://help.okta.com/en-us/Content/Topics/users-groups-profiles/usgp-user-profiles-attributes.htm)
  * **Microsoft Entra ID (Azure AD)**:
      * **Blocklist**: Rejects nearly all "special" characters allowed by RFC (like `! # $ % * / ? ^ { } | ~`) in the UserPrincipalName (email) creation flow.

### 3. Transactional Email Delivery Services

If your user inputs an email, you likely send it via one of these. If they reject it, your app breaks.

  * **AWS SES**:
      * **Format**: Supports only `ASCII` characters by default (UTF-8 support is opt-in and tricky).
      * **Quoted strings**: Generally unsupported in `From` addresses.
  * **SendGrid**:
      * **Validation**: Their inbound parse API and address validation features will flag "Valid RFC" emails like `test@example` (no TLD) as risky or invalid.

### 4. Standard Libraries

Your repo is in good company. Here is what the world's most popular toolkits do:

  * **HTML5 `<input type="email">`**: The browser standard. It is **pragmatic**, not strict. It requires an `@` and at least one char on either side. It does *not* support quoted strings or comments.
  * **Validator.js (Node)**: The `isEmail` function defaults to `allow_utf8_local_part: true` but `allow_ip_domain: false`. It has flags to force-lowercase specific domains (Gmail/Outlook) because they know those providers are case-insensitive.
  * **Laravel (PHP)**: The `email` validation rule uses a "loose" check by default (`filter_var`) which rejects many edge-case RFC formats for security reasons.
