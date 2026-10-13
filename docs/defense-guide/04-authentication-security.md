# Session 4 — Authentication, Authorization, Validation, and Security

## 1. Start with the security problem

The Expense Tracker stores personal financial information. A user should be able to see their own records without seeing another person's expenses. The system must also avoid exposing passwords, tokens, database credentials, or internal errors.

Security is not one button or one library. It is a set of controls placed at the right boundaries. The frontend can improve usability, but the backend must make the final decision about access. The database must enforce structural rules. Deployment must protect the server and its secrets.

## 2. Authentication versus authorization

**Authentication** answers: “Who is making this request?”  
**Authorization** answers: “What is this identified user allowed to do?”

A login form authenticates a user when the server checks the submitted credentials and issues a token. A later request presents that token. Authorization then determines whether the identified user can access the requested resource.

A valid token does not automatically mean that the user can access every record. For example, user A may have a valid token but still must not read user B's expense. The route must check ownership.

## 3. Password hashing

A password should not be stored as plain text. If the database is exposed, plain-text passwords would immediately reveal the users' credentials. A password hash is a one-way representation designed for verification rather than recovery.

The reviewed authentication route uses Werkzeug's password-hashing helpers to generate a hash during registration and check the submitted password during login. The server returns user information and an access token after successful login, not the stored password hash.

A hash is not encryption. Encryption is intended to be reversible with a key; password hashing is intended to make it impractical to recover the original password. Password storage still needs sensible hashing parameters, unique salts handled by the library, and operational protection for the database.

## 4. JSON Web Tokens (JWT)

A JWT is a signed token that carries claims. In this project, the backend uses Flask-JWT-Extended. The login route creates an access token with the user's ID as its identity. Protected routes validate the token and read that identity.

A token is not a password and should not be exposed in logs, screenshots, source code, or shared messages. Anyone who steals a valid bearer token may be able to act as its owner until the token expires or is otherwise invalidated.

The JWT signature depends on a secret held by the server. If an attacker knows that secret, the trust in token signatures is compromised. The secret must be strong, unique, stored outside source control, and configured consistently across the app instances that need to validate tokens.

The reviewed baseline configures `JWT_SECRET_KEY` from `SECRET_KEY`. CI configuration has also used a separate `JWT_SECRET` variable, which exposes a configuration-name mismatch. The team should choose and document one canonical JWT secret setting, support migration from any old setting deliberately, and fail closed in production if the secret is absent or a known placeholder. Do not claim this mismatch is fixed on main unless the correction has been merged and tested.

## 5. Authentication token flow

A normal flow is:

1. User submits email and password over HTTPS.
2. Backend validates the fields and checks the password hash.
3. Backend returns a signed access token after successful authentication.
4. Frontend stores and uses the token according to the application's security design.
5. Frontend sends it in the `Authorization: Bearer <token>` header for protected API requests.
6. Flask-JWT-Extended validates the token and exposes its identity to the route.
7. The route checks whether the identified user may access the requested record.

For a defense, know where the token is created, where it is validated, how the user ID is read, which endpoints require a token, and what the client does when a token expires.

## 6. Input validation is security work

Input validation checks whether the request contains the expected type, shape, and permitted values. For example:
- amount is numeric, positive, and has supported precision;
- date is a valid date in the required format;
- category ID is an allowed integer referring to an existing category;
- email is normalised consistently;
- required text is not empty after trimming;
- month filters use a documented format.

Validation must be performed on the server even if the frontend already checks the same field. A malicious client can call the API directly.

Validation should reject bad values with a 400 response rather than allowing an obscure database error later. The application should not include stack traces or raw database messages in its public response.

## 7. Ownership and object-level authorization

Imagine an API endpoint `GET /api/expenses/41`. The ID identifies a row, but it does not prove the caller owns that row. A safe query scopes the lookup to both the record ID and the authenticated user's ID.

This prevents an insecure direct object reference / broken object-level authorization problem. The same principle must be applied to reading, editing, deleting, exporting, and aggregating private data. It is not enough to protect only the list page.

The reviewed expense routes use user-scoped queries. However, the category routes in the reviewed baseline are not consistently protected by JWT decorators. Categories may intentionally be shared across the application, but the team must explicitly decide whether they are global reference data or user-owned data. If they are global, changes should be restricted to the intended role; if they are user-owned, the schema and queries need an owner relationship.

## 8. Secrets and configuration

Secrets include database passwords, JWT signing keys, SSH private keys, cloud credentials, and access tokens. They should not be committed to the repository or printed in logs. Environment variables or a dedicated secret manager can supply them at runtime.

The reviewed Docker Compose file contains fixed example database credentials in its service environment. Fixed credentials may be acceptable for an isolated local exercise only if clearly treated as disposable, but they should not be reused in production. Move configuration to protected runtime variables or AWS secret storage, rotate any credential that may have been exposed, and avoid publishing real secret values in examples.

An example environment file should contain placeholders, not working production credentials. The app should reject missing or placeholder secrets in production. Merely creating a variable does not prove the application uses it.

## 9. Rate limiting and abuse resistance

Rate limiting restricts how frequently a client can call an endpoint within a period. The project uses Flask-Limiter and places limits on registration and login. This helps slow automated guessing and repeated abuse, but it is not a complete brute-force defence.

A complete design also considers:
- shared rate-limit storage if multiple backend instances run;
- sensible limits for the endpoint's purpose;
- monitoring and alerting;
- generic login errors that do not reveal whether an email exists;
- password quality and account recovery;
- HTTPS so credentials and tokens are not sent in clear text.

An in-memory limiter can behave differently across multiple processes or instances. A distributed deployment may need shared storage such as Redis, depending on the chosen limiter configuration.

## 10. HTTPS, reverse proxy, and AWS

When the application is deployed on AWS, the public traffic should use HTTPS. Nginx may serve as a reverse proxy, but listening on port 80 alone is not proof that TLS is configured. HTTPS can be provided by a properly configured reverse proxy/load balancer and certificate, with the internal routing designed accordingly.

AWS security groups should expose only required public ports. SSH should be restricted to trusted source addresses where possible. PostgreSQL should not be exposed publicly if only the application needs it. IAM permissions should follow least privilege, and secrets should not be placed in GitHub workflow text or a public repository.

The deployment must also avoid trusting arbitrary forwarded headers unless the proxy configuration is understood. A proxy changes what the backend sees about the original request, so forwarded headers should be handled deliberately.

## 11. Common security mistakes

- Trusting a user ID from the request body rather than the verified token.
- Hiding a button in React and assuming the endpoint is protected.
- Using a weak, missing, or hard-coded JWT secret.
- Storing passwords in plain text.
- Returning SQL errors or stack traces to clients.
- Forgetting rollback after a failed transaction.
- Leaving database ports open to the internet.
- Committing a real `.env` file.
- Treating rate limiting as a replacement for authentication.
- Claiming that the app is secure because it uses JWT or Docker.

Security claims must be supported by specific controls and tests.

## 12. Tough defense questions

**Q1. What is the difference between authentication and authorization?**  
Authentication verifies identity. Authorization decides what that identity may access or change. A logged-in user still needs ownership checks for personal financial records.

**Q2. Why hash passwords instead of encrypting them?**  
The system only needs to verify a submitted password, not recover the original. A purpose-built password hash makes offline guessing more expensive and avoids storing reversible credentials.

**Q3. What happens if someone steals a JWT?**  
They may be able to use it as the user until it expires or is revoked by the application's strategy. HTTPS, safe token handling, short lifetimes, secret protection, and careful logging reduce risk.

**Q4. Why is a JWT secret mismatch dangerous?**  
Different processes may sign or validate tokens using different keys, causing legitimate requests to fail. A weak or exposed key can let an attacker forge trusted tokens. Configuration must be explicit and fail closed in production.

**Q5. Why must the server validate an amount if React already does?**  
The request can come from a modified browser, script, or direct HTTP client. Server validation protects the business rule for every caller.

**Q6. Is JWT enough to stop one user reading another user's expense?**  
No. JWT identifies the caller; the route must still scope each resource query to that user's identity.

**Q7. Why should category endpoints be discussed separately?**  
The reviewed baseline has category routes that are not consistently protected by JWT. The team must define whether categories are shared reference data or user-owned and enforce that policy. It is unsafe to assume all routes are protected just because expense routes are.

**Q8. Why is a fixed database password in Compose a problem?**  
It can be copied into every environment and is visible to anyone with repository access. Production credentials should be unique, protected, rotated when exposed, and supplied through controlled runtime configuration.

**Q9. Does using AWS automatically make the application secure?**  
No. Cloud security depends on the configuration: network rules, IAM permissions, patching, secret storage, TLS, backups, and application-level access checks all matter.

**Q10. How would you prove another user cannot access an expense?**  
Create two accounts, create an expense as user A, then attempt to read, update, delete, and export that expense as user B. The API must not disclose or modify A's data. Save the request and response evidence without exposing real tokens.

## 13. Explain it in one minute

“Authentication proves who is making a request; authorization determines what that user may do. The backend hashes passwords, issues a signed access token after login, verifies it on protected routes, and must scope personal data to the token's user identity. Server-side validation and database constraints protect against requests that bypass the UI. Secrets must be strong and stored outside source control. Rate limits help reduce abuse, and AWS network controls plus HTTPS protect traffic and infrastructure. Each security claim should be backed by a specific test, not just by naming a library.”
