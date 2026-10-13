# Session 4 — Authentication, Authorization, and Application Security

## 1. What does application security protect?

Expense Tracker contains personal financial information. Even if the data is not a bank balance, it can reveal a person's spending habits, travel, health purchases, or living situation. Security therefore protects confidentiality, integrity, and availability.

- **Confidentiality:** only authorised people can read the information.
- **Integrity:** records and calculations are not changed by an unauthorised person or an invalid operation.
- **Availability:** legitimate users can use the application when needed.

Security is not one library or one login page. It is a set of controls across the browser, API, database, deployment, credentials, and operational process.

## 2. Authentication versus authorization

Authentication answers: **Who are you?** Authorization answers: **What are you allowed to do?**

When a user logs in, the backend checks the supplied credentials. If they are valid, it issues an access token. The frontend can send that token on later requests. Flask verifies the token and retrieves the identity from its claims.

But a valid token is not permission to read every expense. For a request to list expenses, the backend must filter by the authenticated user's identity. For an update or delete, it must verify that the target record belongs to that user or that the user has another explicitly defined permission. The same ownership rules must cover summaries and exports, not just the main list screen.

A common serious bug is an insecure direct object reference: the API accepts an expense ID and returns or changes that record without verifying ownership. The ID may be hard to guess, but that is not a security control. Authorisation must be enforced on the server for each operation.

## 3. Passwords and password hashing

Passwords should never be stored as readable text. A password hash is produced by a one-way password-hashing algorithm with a salt and work factor. When a user logs in, the server checks the submitted password against the stored hash.

Hashing is not the same as encryption. Encryption is designed to be reversible with a key; password hashing is designed so the original password is not directly recovered from the stored value. A slow password-hashing algorithm makes large-scale guessing more expensive.

The project uses Werkzeug password-hashing helpers in the authentication route. The team should be ready to show where the hash is created during registration and where it is verified during login. Never show a real user's hash in a presentation or log.

## 4. What is a JWT?

JWT means JSON Web Token. It is a signed token containing claims, such as the subject identity and expiry information. The server signs the token with a secret key; later the server validates the signature and claims.

A JWT is normally encoded, not encrypted. Anyone who obtains the token may be able to decode its payload, so sensitive information such as passwords must never be placed in the claims. The signature protects integrity; it does not hide the contents.

A typical flow is:

1. User submits email and password over HTTPS.
2. Flask verifies the credentials against the stored password hash.
3. Flask creates a signed access token.
4. The frontend stores and sends the token according to the chosen client design.
5. Protected routes verify the token and derive the user identity.
6. Expired or invalid tokens are rejected.

A token should expire. Otherwise a stolen token could remain useful for too long. Token revocation is a separate design problem because stateless tokens are not automatically invalidated just because a user logs out in the browser.

## 5. Token storage and browser risks

The current frontend API helper stores the access token in browser local storage. That is simple and persists through reloads, but JavaScript running in the same origin can read it. A successful XSS vulnerability could therefore expose the token.

An alternative is a server-managed session or an appropriately configured HttpOnly, Secure, SameSite cookie. HttpOnly prevents ordinary JavaScript from reading the cookie, but cookie-based authentication requires careful CSRF consideration and correct same-site/secure settings. There is no universal choice that is secure without correct configuration.

In defence, explain the actual design and its trade-off. Do not say the token is “encrypted by local storage” or that local storage is automatically secure.

## 6. Secrets and configuration

Secrets include JWT signing keys, Flask secret keys, database passwords, SSH private keys, and deployment credentials. They must not be committed to Git, pasted into screenshots, printed in logs, or embedded in a public frontend build.

Use environment variables or a managed secret store, restrict access, rotate credentials if exposure is suspected, and separate development/test values from production values. A value being called a secret does not make it secret if it is stored in a public repository.

Important source-review note: the reviewed main branch has backend configuration that reads SECRET_KEY for both Flask's secret key and the JWT signing key. The CI configuration also defines JWT_SECRET, but a variable that the application does not read has no effect. The reviewed docker-compose.yml also contains fixed PostgreSQL credentials. These are concrete items to reconcile before describing production secret management as complete. Use the current correction branch and deployed configuration as the evidence source; do not claim a proposed fix is already merged or deployed.

Production configuration should fail closed when required secrets are missing or still set to known placeholder values. A service should not quietly start with a predictable signing key. Test the startup behaviour explicitly.

## 7. Input validation and injection

Every request is untrusted, even if it came from the application's own frontend. A caller can edit requests, send unexpected JSON types, omit fields, or send extremely large input.

Validate type, range, format, and allowed values. Use SQLAlchemy's parameterised query mechanisms instead of concatenating untrusted strings into SQL. Parameterisation keeps data separate from SQL instructions and helps prevent SQL injection.

Input validation must also consider the meaning of the value. A date should be a valid date in the agreed format; an amount should be positive and have acceptable precision; a category reference should exist. Validation should return safe and consistent error messages.

## 8. Rate limiting and abuse resistance

Rate limiting restricts the number of requests a client may make in a time window. It can slow repeated login attempts and protect endpoints from excessive requests. It does not replace authentication or authorisation.

The application configures Flask-Limiter and maps a rate-limit exception to HTTP 429. The team should check the actual scope and storage backend. In-memory limits can reset on restart and may not be shared across multiple workers or server instances. A multi-instance production setup generally needs a shared limiter store and an explicit plan for identifying clients safely behind a reverse proxy.

## 9. Nginx, network exposure, and TLS

Nginx is the public-facing reverse proxy in the container design. It can serve the frontend and forward /api requests to Flask. Ideally, only the required public port is exposed; PostgreSQL and Flask should not be unnecessarily open to the internet.

AWS security groups should allow only required inbound traffic. If the application is publicly accessible, HTTPS should be configured so login credentials and tokens are not sent in cleartext over the network. The live demonstration should use the real deployment URL and confirm the TLS behaviour rather than assume it.

The reverse proxy is not a substitute for application security. A request can still be malicious after it passes through Nginx. Flask must authenticate and validate it, and PostgreSQL must enforce data integrity.

## 10. Least privilege and AWS IAM

Least privilege means each account, process, and service receives only the permissions it needs. The backend database user should not be a PostgreSQL superuser for normal application operations. The AWS identity used for deployment should have only the necessary permissions, and the EC2 security group should expose only required ports.

SSH keys and deployment secrets should be protected. Never put a private key in a repository or bake it into a Docker image. The EC2 instance should be patched and managed, and access should be restricted to authorised team members.

A strong answer distinguishes the cloud IAM identity from the application's end-user identity. AWS IAM controls access to AWS resources. JWT-based application authentication controls access to Expense Tracker API resources. They solve different problems.

## 11. Error messages, logs, and privacy

A public error should say enough to help a user, but not reveal internal implementation details. A database failure can return a generic message to the client while protected logs record the error category and request identifier.

Do not log passwords, bearer tokens, database URLs, SSH keys, or full personal financial records unnecessarily. Screenshots for the defence should use test accounts and fake expenses. If a secret has been exposed, removing it from the latest commit is not enough: it may remain in Git history or logs, so rotate it.

## 12. Threat scenarios the team should understand

**Threat: one user reads another user's expense.**  
Control: derive the user identity from the verified token and scope queries and mutations to that user. Test with two accounts.

**Threat: attacker guesses a weak JWT secret.**  
Control: use a strong, random, private signing key; reject missing/placeholder keys; restrict secret access; rotate if exposed.

**Threat: attacker submits a negative amount or invalid date.**  
Control: validate on the server and use database constraints for core invariants.

**Threat: two requests create duplicate budgets.**  
Control: enforce uniqueness in PostgreSQL, catch the expected constraint conflict, roll back, and return 409.

**Threat: credentials leak from Git.**  
Control: secret scanning, environment/secret management, access controls, and rotation after exposure. Secret scanning is helpful but cannot guarantee that every secret is detected.

**Threat: an expired token is still treated as valid by the UI.**  
Control: server-side token expiry validation, consistent 401 handling, clearing authenticated UI state, and tests for expired tokens.

**Threat: public traffic reaches PostgreSQL directly.**  
Control: do not publish the database port publicly, restrict AWS security groups, and use internal container networking.

## 13. Connections to other sections

- **Frontend:** stores and sends the token but cannot enforce server-side permissions.
- **Backend:** validates JWTs, derives identity, checks ownership, validates input, and returns safe errors.
- **Database:** stores password hashes and enforces constraints; the app's database account should have limited privileges.
- **Testing:** two-user isolation, invalid-token, expired-token, input-validation, rate-limit, and secret-configuration tests verify controls.
- **Docker/Nginx:** controls network paths and public exposure but does not replace API checks.
- **CI/CD/AWS:** secrets, IAM permissions, security groups, TLS, patching, and deployment credentials are operational security controls.

## 14. Difficult defence questions and answers

**Q1. What is the difference between hashing and encryption?**  
Encryption is reversible with a key. Password hashing is designed to be one-way, with a salt and computational cost that make guessing expensive.

**Q2. Is a JWT encrypted?**  
Usually a signed JWT is encoded and signed, not encrypted. Its payload may be readable. Never put passwords or confidential data in its claims.

**Q3. Why is a valid JWT not enough to authorise an expense request?**  
It identifies the caller. The backend still must check that the caller owns the requested expense or has explicit permission to access it.

**Q4. Why is local storage a security concern?**  
JavaScript on the same origin can read it. If an XSS flaw allows an attacker to run script, a stored bearer token may be stolen. Explain the current trade-off and mitigations honestly.

**Q5. Why do we need both JWT_SECRET and SECRET_KEY?**  
They can serve different cryptographic purposes in the application. The key lesson is that the application must read the intended variable and fail safely when a required secret is missing. A variable defined in CI does nothing if the code never uses it.

**Q6. What is least privilege?**  
Give each user, process, database account, or cloud identity only the permissions required for its task. This limits damage if an account or component is compromised.

**Q7. What is SQL injection?**  
It occurs when untrusted input is treated as part of SQL instructions. Parameterised queries and safe ORM query construction keep values separate from SQL syntax.

**Q8. Does HTTPS protect the database from unauthorised queries?**  
No. HTTPS protects data in transit between network endpoints. Backend authorisation, database credentials, network controls, and constraints still determine which operations are allowed.

**Q9. Does a secret scanner prove the repository has no secrets?**  
No. It can detect many patterns but can miss some secrets and cannot undo a leak that already occurred. Use secure storage, reviews, limited access, and rotation when needed.

**Q10. Why are AWS IAM and JWT not the same?**  
AWS IAM controls access to AWS resources. JWT authentication identifies users of the Expense Tracker API. A user token should not grant AWS administrative access, and an AWS role does not automatically authorise a user to view another person's expenses.

**Q11. What should happen if a required signing key is missing?**  
Production startup should fail clearly rather than use a predictable fallback. Add tests for missing and placeholder values, and keep test configuration separate from production configuration.

**Q12. What if a user edits the request in browser developer tools?**  
The backend must treat the request as untrusted, validate the fields, derive ownership from the authenticated identity, and enforce permissions. Hiding controls in React is not a security boundary.

## 15. Security defence checklist

Be able to explain the login flow, password hashing, JWT creation/validation/expiry, ownership filtering, secret configuration, rate limiting, input validation, safe errors, database constraints, Nginx exposure, AWS security groups, and TLS. Use two test users to demonstrate isolation. Do not show real tokens or credentials. Do not claim a control is complete without code or runtime evidence.
