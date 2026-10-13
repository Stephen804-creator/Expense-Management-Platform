# Session 5 — Authentication, Authorization, and Application Security
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. Security starts with the data being protected

Expense Tracker contains personal financial information. Even if this is a student project, a user's expenses, dates, categories, and spending patterns should not be exposed to another user. Security is therefore not a single feature called “login”; it is a set of controls across the browser, API, database, deployment configuration, and operational process.

A useful way to explain security is to ask four questions: Who is making the request? What is that person allowed to do? What data can the person see or change? How do we prevent secrets and internal details from being exposed?

## 2. Authentication versus authorization

**Authentication** verifies identity: “Who are you?” Logging in is an authentication operation.

**Authorization** decides permission: “What may you do?” A valid login does not automatically give a user access to every expense in the database.

For example, User A and User B may both authenticate successfully. User A must still not read User B's expense by changing an ID in a URL. This is an authorization test, not a login test. The backend should use the verified identity to scope database queries and verify ownership before reads, updates, deletes, and exports.

Hiding a button in React is not authorization. A caller can bypass the interface and call Flask directly. Permission checks must be enforced on the server.

## 3. Passwords and login

Passwords should never be stored as plain text. A password-hashing algorithm stores a derived value that is deliberately expensive to reverse, normally with a unique salt. On login, the server verifies the submitted password against the stored hash. It should not decrypt or retrieve the original password because the application should not have stored it in recoverable form.

Hashing is not the same as encryption. Encryption is designed to be reversible with a key; password hashing is designed to verify a password without keeping the original.

The team must inspect the actual user model and login code before stating which hashing library and parameters are used. Do not claim a particular algorithm solely because it is common practice.

Login errors should avoid giving attackers unnecessary clues about whether an email address exists. Rate limiting can reduce repeated automated attempts, but it does not replace password hashing, secure transport, monitoring, or appropriate password policy.

## 4. Tokens and JWT

A JSON Web Token (JWT) is a signed set of claims. A server can verify the signature and read claims such as subject and expiry. A signed token is not automatically encrypted: its contents may be readable by anyone who possesses it. Do not put secrets or unnecessary personal financial information in the payload.

The signing secret is critical. If an attacker obtains the secret, they may be able to forge valid tokens. The secret must be unpredictable, kept out of source control, and configured consistently in every environment. A development fallback that silently accepts a missing or placeholder secret can make production security fail open.

Token expiry limits the lifetime of a stolen token. The application should respond safely to expired or invalid tokens. A logout button does not necessarily revoke an already issued stateless JWT unless the application has a revocation mechanism or uses short-lived tokens with an appropriate refresh strategy. Explain the actual design rather than assuming logout invalidates every copied token.

The correction work for this project addresses a configuration mismatch among JWT-related environment variable names and documents the intended configuration. Production should fail closed if a required secret is missing or clearly a placeholder. CI success does not prove that the AWS environment has the right secret; runtime configuration must be checked without printing the secret itself.

## 5. Secrets and environment variables

A secret is a credential that grants access, such as a database password, JWT signing secret, API key, or deployment key. An environment variable is one way to supply configuration to a process without hard-coding it in source. In production, a secret manager or protected deployment secret is preferable where available.

An example file such as .env.example should show variable names and safe placeholders, never live credentials. A .gitignore entry helps prevent accidental commits but is not a security boundary by itself. Secrets can still be exposed through logs, build output, shell history, screenshots, or copied configuration files.

If a real credential is committed, deleting it in a later commit is not enough because it may remain in Git history. Rotate the credential, review access logs if appropriate, and remove or remediate the exposed history according to the team's process.

The database account should follow least privilege: grant only the permissions needed by the application. Avoid using a database superuser for routine web requests. Network rules should allow access only from the required services.

## 6. Input validation and injection risks

Treat every client-supplied value as untrusted, including query parameters, route IDs, JSON bodies, file contents, and exported text. Validate type, range, format, and relationships before use. Use parameterized database operations or ORM query mechanisms rather than constructing SQL from user-controlled strings.

Output encoding also matters. If a description is inserted into HTML without proper escaping, a malicious value could become executable script in a browser. React escapes ordinary text rendering by default, but unsafe HTML APIs or custom rendering can reintroduce the risk. Inspect actual usage before declaring the app vulnerable or safe.

CSV export has a different concern: spreadsheet applications may interpret a cell starting with formula characters as a formula. Correct CSV quoting alone may not prevent spreadsheet formula injection. Review export code and define a tested mitigation where needed.

## 7. HTTP, TLS, CORS, and reverse proxies

TLS/HTTPS protects data in transit between the browser and the public endpoint. Without it, credentials and tokens may be exposed on the network. The team must verify the deployed endpoint actually uses HTTPS rather than assuming that an AWS server automatically enables it.

CORS is a browser policy that controls whether a web page from one origin may read responses from another origin. It is not user authentication and does not stop command-line clients or direct API calls. Configure allowed origins narrowly for production rather than using a permissive policy without a reason.

Nginx acts as a reverse proxy and may serve the frontend, route API paths, and pass requests to Flask. It can add security headers and manage TLS if configured to do so, but it does not replace backend validation or authorization. Headers such as content-type protection, framing restrictions, and a carefully designed Content Security Policy can reduce some browser risks; they must be compatible with the application and tested, not copied blindly.

## 8. Rate limiting and abuse protection

Rate limiting restricts how frequently a client can perform an operation. It can protect login, registration, exports, or expensive endpoints from excessive requests. A process-local in-memory limiter may reset on restart and may not coordinate across multiple server instances. A multi-instance deployment may need a shared store or an upstream control.

The team should know which routes are rate-limited, how client identity is determined behind Nginx, and whether proxy headers are trusted safely. If an application trusts a spoofable forwarded IP header, attackers may bypass or manipulate rate-limit identity. Rate limiting should be tested for both legitimate requests and repeated abuse.

## 9. Errors, logs, and privacy

Client-facing errors should not reveal passwords, tokens, SQL statements, stack traces, connection strings, or secret values. Server logs should capture enough information to investigate failures but avoid recording bearer tokens and unnecessary financial details.

Security logging and application debugging have different purposes. A useful event might record a failed authentication attempt and timestamp, but not the raw password submitted. Access to logs should be limited, and retention should be considered.

## 10. Threat modelling for this application

Threat modelling means identifying what can go wrong, who might cause it, what asset is at risk, and which control reduces the risk. A practical list for Expense Tracker includes:

- A user reads another user's expenses by changing an ID.
- A user exports another user's data.
- A stolen token is reused.
- A weak or leaked JWT secret allows forged authentication.
- A committed database password is reused by an attacker.
- A malformed amount or date corrupts calculations.
- Concurrent requests create duplicate budgets.
- A category deletion silently changes a budget's meaning.
- A public health endpoint exposes detailed database credentials or internal errors.
- A deployment exposes the database port unnecessarily.

For each risk, name the control and evidence. “We care about security” is not evidence. A route-level ownership check plus a negative test is evidence of a specific control. A secret scan passing is useful but does not prove that the AWS runtime configuration is correct.

## 11. Hard defense questions

**Q: If JWT is signed, why is the secret important?**  
The signature allows the server to detect forged or changed claims. If the signing secret leaks, an attacker may create tokens that pass verification.

**Q: Is a JWT encrypted?**  
Not necessarily. A typical signed JWT has readable claims. It should not contain secrets.

**Q: Is CORS a security wall around the API?**  
No. It is a browser cross-origin policy. It does not replace authentication and authorization.

**Q: If a user cannot see the Delete button, can they delete the record?**  
They may still call the API directly. The backend must reject unauthorized requests.

**Q: Does .gitignore make committed secrets safe?**  
No. It helps prevent future accidental tracking, but a secret already committed must be treated as exposed and rotated.

**Q: Does logout invalidate a copied JWT?**  
Only if the system has a revocation mechanism or another design that makes the token unusable. A stateless token may remain valid until expiry.

**Q: Why not show the database exception to the user?**  
It may expose internal schema and configuration details. Return a safe message and keep diagnostic details in protected logs.

**Q: What does least privilege mean?**  
Each account or service receives only the permissions needed for its job, limiting the damage if that credential is misused.

## 12. Evidence needed before claiming security is complete

Confirm the actual login/hash implementation, authorization checks on each protected route, production secret validation, environment variable names, HTTPS behavior, Nginx headers and routing, database network exposure, log redaction, rate-limit behavior, and tests for cross-user access. A CI secret scan is one layer of evidence, not a complete security assessment.

## 13. Capstone alignment

The capstone explicitly requires that passwords and API keys not be committed, environment variables or secrets be used, and cloud IAM permissions be appropriate. It also requires authentication and a working deployed application. Security should therefore be explained across React, Flask, PostgreSQL, Docker, GitHub Actions, Nginx, and AWS—not as a login page alone. AWS is the tutor-approved alternative to the Azure VM named in the original handout.
