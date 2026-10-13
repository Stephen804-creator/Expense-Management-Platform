# Session 2 — Backend Development and REST API

## 1. What the backend does
The backend is the server-side part of the Expense Tracker. It receives requests, identifies the caller, validates input, applies business rules, communicates with PostgreSQL, and returns structured responses. It is the trusted place for rules that cannot safely be left to the browser.

The project uses Flask, route blueprints, Flask-JWT-Extended, Flask-SQLAlchemy/SQLAlchemy, and Flask-Limiter. A route handles a particular request; a blueprint groups related routes. The app registers blueprints for authentication, categories, expenses, budgets, and summaries. This is easier to maintain than putting every feature in one file.

## 2. How a request is handled
A browser sends an HTTP request, such as POST /api/expenses. Nginx forwards it to Flask. Flask matches the path and method, parses the request, checks authentication when required, validates the data, queries or changes records, commits or rolls back a database transaction, and returns a response.

The flow is: Browser → Nginx → Flask route → validation/authentication → SQLAlchemy → PostgreSQL → response → browser.

Each step can fail for a different reason. Nginx can be unable to reach Flask; Flask can reject an invalid token; validation can reject the amount; PostgreSQL can reject a uniqueness conflict; or the frontend can mishandle a successful response. Troubleshooting should identify the failing layer rather than restart everything blindly.

## 3. REST and HTTP status codes
REST-style APIs expose resources through paths and use HTTP methods to describe operations. GET reads a resource. POST usually creates a resource or starts an operation. PUT updates a resource representation. DELETE requests deletion. The path /api/expenses represents the expense collection; /api/expenses/42 identifies one expense.

Common status codes:
- **200 OK:** operation succeeded.
- **201 Created:** a record was created.
- **204 No Content:** operation succeeded with no response body.
- **400 Bad Request:** malformed or invalid input.
- **401 Unauthorized:** missing, invalid, or expired authentication.
- **403 Forbidden:** authenticated caller is not permitted, where this policy is used.
- **404 Not Found:** resource does not exist.
- **409 Conflict:** request conflicts with existing state or a data-integrity rule.
- **429 Too Many Requests:** rate limit exceeded.
- **500 Internal Server Error:** unexpected server failure.

Status codes are part of the API contract. If every response is 200, clients and tests cannot reliably distinguish success from failure.

## 4. Authentication versus authorization
Authentication answers “Who is making this request?” Authorization answers “Is this caller allowed to perform this operation on this resource?”

Registration validates data and stores a password hash rather than the original password. Login checks the supplied password against the stored hash and issues a JWT access token. The frontend attaches the token as a bearer token. Protected routes validate the token and derive the current user identity from it.

A valid JWT does not authorize access to every row. Expense and budget operations must scope records to the identity from the verified token, not trust a user ID supplied by the browser. Otherwise a caller may alter a request and attempt to read, update, delete, or export another user's data.

Category routes need a separate policy decision. The current categories are shared data, and the audit has an open finding about authorization for category mutations. Do not claim every category operation is protected merely because expense and budget routes require JWT. The team needs to define whether categories are shared/admin-managed or user-owned and then implement and test that rule.

## 5. Why server-side validation is necessary
Browser input is untrusted. Requests may come from modified code, scripts, direct HTTP clients, or malicious callers. The backend should validate types, required fields, lengths, allowed ranges, references, and date formats before business logic or database writes.

Examples include:
- JSON body must have the expected structure.
- Names/descriptions must be the expected type and length.
- Amount must be finite, positive, within supported range, and have no more than two decimal places.
- Date and month values must use the exact required format.
- Category ID must be valid and refer to an existing category.
- A budget month must be normalized to the first day of the month.
- Invalid input should return a clear client error, not crash the server.

Python has edge cases: booleans are subclasses of integers, so a boolean may accidentally pass a loose integer check. Strings, lists, null values, NaN, and infinity can also produce surprising behavior if assumptions are not explicit. Validate types before conversion and test malformed input deliberately.

## 6. SQLAlchemy, commits, and rollback
SQLAlchemy maps Python models to relational tables and provides a query/transaction interface. It does not eliminate the need to understand SQL or database constraints.

A write generally adds or modifies an object, then commits. A commit can fail even if a prior query said the record did not exist. Two requests can race: both check for a duplicate, then both try to insert. The database constraint makes the final integrity decision.

If a commit fails, the code should roll back the SQLAlchemy session before using it again. Known constraint conflicts should map to a meaningful 409 response; unexpected errors should return a sanitized 500 response. Raw database exception text should not be sent to the browser because it may expose schema details or internals.

Catching every exception and returning the same response is not automatically robust. Good error handling distinguishes expected conflicts from unexpected failures, rolls back, avoids sensitive disclosure, and logs enough protected diagnostic information for investigation.

## 7. Rate limits, signing secrets, and proxy awareness
Flask-Limiter applies request limits to sensitive endpoints such as registration and login. Rate limiting can reduce brute-force attempts and accidental request floods, but it is not a replacement for authentication, monitoring, authorization, or network controls.

The audit notes that in-memory rate-limit storage produces a warning. In-memory counters are local to each process and reset on restart. Multiple workers or EC2 instances can maintain separate counters. A multi-instance production design needs a shared storage backend and tests that confirm the limit is consistent.

The backend validates production Flask and JWT signing secrets: they must be strong, not placeholders, and distinct. Real values belong in deployment secrets/environment configuration, not source control. If a secret was exposed, rotate it; a later commit does not erase the old value from Git history.

ProxyFix trusts one proxy hop in the current app. That must match the actual network path. Trusting forwarded headers from arbitrary clients or the wrong number of hops can corrupt client-IP or protocol assumptions and weaken rate limiting/logging.

## 8. Connections and capstone alignment
React sends requests and interprets status codes. Nginx routes /api to Flask. PostgreSQL persists records and enforces constraints. Docker Compose supplies environment configuration and internal networking. AWS EC2 hosts the stack. GitHub Actions runs lint/tests and migration checks. QA verifies access control, validation, error handling, and full user journeys.

The capstone requires a Flask REST API connected to PostgreSQL and a React client, packaged by Docker Compose, routed by Nginx, checked through GitHub Actions, and deployed to AWS EC2. Evidence should include route definitions, API requests/responses, authentication and authorization tests, validation tests, constraint tests, and logs or screenshots from the tested version. A route existing in source is not proof it is reachable through Nginx.

## 9. Tough questions
**Why use Flask instead of putting logic in React?** The server is the trusted place for validation, authorization, and database operations. Browser code is user-controlled.

**What is the difference between 401 and 403?** 401 means valid authentication is missing; 403 means the caller is authenticated but not allowed. The app should apply a consistent policy.

**Why use JWT?** It provides a signed token that the API validates on later requests. It creates responsibilities around expiry, signing secrets, storage, and session invalidation.

**Why are database constraints needed if the API checks duplicates?** Concurrent requests can pass the API pre-check simultaneously. The database constraint protects the final stored state.

**What should happen after a failed commit?** Roll back the session, map known conflicts correctly, and return a safe response for unexpected failures.

**How do you stop user A reading user B's expenses?** Derive identity from the verified token and scope every user-specific list, read, update, delete, summary, and export query to that identity.

**Does a JWT prove authorization?** No. It identifies a valid token subject; endpoint policy still decides what that subject may do.

**Why not return raw database errors?** They can expose SQL, schema, or internal information and make the API contract unstable.

**What is the risk of in-memory rate limiting?** Counters are not shared across workers/instances and reset when a process restarts.

**How do you prove endpoint security?** Test missing/invalid/expired tokens, two-user isolation, malformed input, expected conflicts, unexpected failures, response sanitization, and the same paths through Nginx.

## 10. Defense-ready summary
“The Flask backend exposes organized REST endpoints, validates untrusted input, authenticates protected requests with JWT, applies business rules, uses SQLAlchemy to access PostgreSQL, handles transaction rollback and expected conflicts, and returns explicit HTTP status codes. The database remains the final integrity guard, and the deployed API must be tested through Nginx and AWS.”
