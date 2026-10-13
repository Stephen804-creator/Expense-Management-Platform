# Session 2 — Backend Development: Flask REST API and Business Rules

## 1. What is the backend?

The backend is the server-side part of an application. It receives requests, identifies the caller, validates submitted data, applies business rules, talks to PostgreSQL, and returns a response.

In Expense Tracker, Flask provides API routes for authentication, categories, expenses, budgets, and summaries. React asks for work; the backend decides whether the request is valid and permitted; PostgreSQL stores the result.

Think of a bank counter. A customer asks to withdraw money, but staff must verify identity and follow account rules before accepting the transaction. The customer interface cannot declare the transaction valid merely because the customer clicked a button.

## 2. What is an API?

API means Application Programming Interface: an agreed way for one piece of software to request services from another. A REST-style API commonly exposes resources through HTTP methods and paths.

- **GET** retrieves information and should not intentionally change stored data.
- **POST** creates a resource or triggers an operation.
- **PUT** updates or replaces a resource representation, depending on the contract.
- **PATCH** changes selected fields where supported.
- **DELETE** requests removal of a resource.

An endpoint is a specific operation exposed by the API. For example, GET /api/expenses and POST /api/expenses use the same path but perform different operations.

The route is not the whole system. A handler may authenticate the request, validate fields, query the database, commit a transaction, and format a response. Each step can fail and needs suitable handling.

## 3. Why Flask and blueprints?

Flask is a lightweight Python web framework. It supplies routing, request/response handling, and integration points without forcing every application into a large built-in structure.

The project groups routes into blueprints for authentication, categories, expenses, budgets, and summaries. This keeps related endpoints together and allows the main Flask application to register them as modules.

Blueprints help organisation but do not automatically create a clean architecture. If route handlers become very large, repeated validation and business rules may need service/helper functions. The important point is to understand where each responsibility currently lives and why it belongs there.

## 4. How one request travels through the backend

Consider POST /api/expenses.

1. The browser sends an HTTP request with a JSON body and bearer token.
2. Nginx forwards the /api request to the backend container.
3. Flask matches the HTTP method and URL to the expense route.
4. The JWT-required decorator checks the token.
5. The handler obtains the user identity from the verified token.
6. It reads and validates the JSON fields.
7. It creates a SQLAlchemy model object and adds it to the database session.
8. Commit asks PostgreSQL to make the transaction durable.
9. If a database exception occurs, the handler should roll back the failed transaction and return a safe error.
10. Flask returns a status code and response body.
11. The frontend parses the response and updates the screen.

This sequence is a debugging map. A 404 may indicate a route or proxy problem. A 401 may indicate missing, invalid, or expired authentication. A 400 usually points to invalid input. A 500 indicates an unexpected server failure and should be investigated through protected server logs.

## 5. HTTP status codes and why they matter

- **200 OK:** the request succeeded.
- **201 Created:** a resource was created.
- **204 No Content:** success with no response body.
- **400 Bad Request:** malformed input or failed validation.
- **401 Unauthorized:** authentication is missing or invalid.
- **403 Forbidden:** the caller is authenticated but not permitted to perform the action.
- **404 Not Found:** the route or resource does not exist, or the API deliberately avoids revealing a protected resource.
- **409 Conflict:** the request conflicts with existing state, such as a duplicate budget where the API defines that conflict.
- **429 Too Many Requests:** a rate limit was exceeded.
- **500 Internal Server Error:** an unexpected server-side failure occurred.

These codes are not interchangeable. Returning 500 for every problem makes the client harder to build and hides the difference between user-correctable input and a server defect. Returning 200 when a write failed is worse because the frontend is told that an operation succeeded when it did not.

The project commonly returns a success indicator, message, and data payload. A consistent contract allows the frontend to handle responses predictably. Verify the actual response format of an endpoint before assuming every route returns exactly the same fields.

## 6. Validation and business rules

Validation asks, “Is this input acceptable?” Business rules ask, “Is this operation allowed under the application's rules?”

For an expense, the backend should check that required fields exist; the amount is numeric and greater than zero; decimal precision is acceptable; the date is valid; the category exists; the caller is authenticated; and the record is assigned to the authenticated user rather than a user ID supplied by an untrusted browser.

Why use Decimal for money? Binary floating-point values cannot exactly represent many decimal fractions. Financial values should preserve intended decimal precision. PostgreSQL NUMERIC(12,2) stores exact decimal quantities at the specified scale. Converting an amount to a floating-point number for a display response may be acceptable in some designs, but calculations and rounding need to be defined and tested.

The frontend may repeat basic validation for usability. The backend must validate again because a caller can bypass the frontend and send a request directly. PostgreSQL constraints provide another layer for essential invariants.

## 7. Authentication and authorization are different

**Authentication** asks, “Who is making this request?” **Authorization** asks, “What is this caller allowed to do?”

A valid JWT establishes an identity, but it does not automatically permit the user to read every expense. Queries must be scoped to the authenticated user's ID. The expense routes use the authenticated identity as the user filter when listing and exporting expenses. A missing ownership filter could expose another user's financial information.

Do not trust a user_id submitted in the request body as the authority for record ownership. The server should derive ownership from the verified identity and enforce it on every relevant read, update, delete, summary, and export operation.

Categories require a clear policy. The schema may define them as shared records, but the product still needs to decide whether categories are global, administrator-managed, or owned by individual users. Do not invent a policy during defence; describe the actual policy and show how the backend enforces it.

## 8. SQLAlchemy sessions and database transactions

SQLAlchemy is the database toolkit/ORM used by the Flask application. An ORM maps Python objects and attributes to database tables and columns. It reduces repetitive SQL, but understanding SQL remains important for performance, relationships, and correctness.

A database session tracks changes and coordinates a transaction. Adding an object stages it; it does not prove that the row is permanently saved. Commit asks the database to accept the transaction. If a commit fails, the session must be rolled back before it can safely be reused. Otherwise SQLAlchemy may remain in a failed transaction state.

A transaction groups operations so the database can commit them together or roll them back. This avoids leaving half-finished changes after an error. The handler should identify expected conflicts specifically; unexpected errors should be logged in a controlled way and returned as generic server errors.

Do not catch every exception and label it “duplicate” or “invalid input.” A database can report a uniqueness violation, foreign-key violation, connection failure, permission failure, or programming defect. These failures need different internal diagnosis even when the public response must not reveal sensitive details.

## 9. Summaries, budgets, filters, and exports

The backend does more than basic CRUD (Create, Read, Update, Delete). It produces derived views of the data.

- **Date/month filters:** validate the requested format and use a precise date range.
- **Monthly summaries:** include exactly the records in the defined month and for the authenticated user.
- **Category summaries:** group by the intended category and month without mixing users' records.
- **Budgets:** define whether a limit applies to one category or the whole month, and handle duplicates consistently.
- **CSV export:** include only the authorised user's matching records and honour the filters selected by the user.

A common defect occurs when the dashboard and CSV export use different query logic. The dashboard may show one total while the exported file contains different records. Shared query-building logic and tests for both JSON and CSV can reduce this risk.

CSV has an additional security concern. If user-controlled text begins with spreadsheet formula characters, spreadsheet software may interpret it as a formula. The team should assess CSV formula injection and implement a safe export strategy where appropriate. Do not claim the risk is mitigated unless code and tests demonstrate it.

## 10. Rate limiting and safe errors

Rate limiting restricts how frequently a client can call an endpoint. It can help reduce brute-force login attempts and abusive traffic. The application configures Flask-Limiter and returns 429 for rate-limit errors. A limit stored only in process memory may behave differently across multiple workers or instances. A production multi-instance deployment usually needs a shared backing store and carefully chosen limits.

Error responses should help the user without exposing passwords, tokens, SQL statements, file paths, or stack traces. Detailed diagnostics belong in protected server logs, ideally with a request or correlation ID. Logs should not contain secrets or unnecessary personal financial data.

## 11. Connections to other sections

- **Frontend:** defines the request and response contract used by the shared API service.
- **Database:** stores records and enforces integrity constraints.
- **Security:** JWT validation identifies the caller; query and route logic enforce permitted access.
- **Testing:** API tests verify validation, status codes, ownership, rollback, and calculations.
- **Nginx:** routes public /api requests to Flask.
- **Docker Compose:** provides network and environment configuration and starts PostgreSQL before the backend depends on it.
- **CI/CD and AWS:** tests and packages a particular code revision, then deploys it and checks health.

## 12. Difficult defence questions and answers

**Q1. What is the difference between a route and an endpoint?**  
A route is a URL pattern registered by the application. An endpoint is a specific operation exposed through the API, including its HTTP method. GET and POST on the same path are different operations.

**Q2. Why use Flask if Python can already run code?**  
Python is a programming language. Flask provides web machinery for receiving HTTP requests, matching routes, reading request data, and returning HTTP responses.

**Q3. Why is commit not just a normal save function?**  
Commit asks the database to commit the current transaction. The database may reject it because of constraints or operational failure. Until commit succeeds, the application must not claim the record is durably saved.

**Q4. What is rollback and why is it necessary?**  
Rollback abandons the failed transaction state. After a failed SQLAlchemy commit, the session generally needs rollback before reuse. It prevents a later request from inheriting a failed session state.

**Q5. Why return 409 instead of 500 for a duplicate budget?**  
A duplicate is a conflict with existing state that the client can resolve. A 500 suggests an unexpected server problem. Return 409 only when the expected conflict has been identified; unrelated database failures must not be mislabeled.

**Q6. If the JWT is valid, can the user read any expense ID?**  
No. Authentication proves the token is valid, not that the user owns every record. The backend must scope queries and enforce ownership on every relevant operation.

**Q7. Why validate in the API if PostgreSQL has constraints?**  
The API provides clear error messages and checks complex business rules. The database remains the final integrity layer for invariants such as positive amounts, foreign keys, and uniqueness.

**Q8. Why not calculate all totals in React?**  
The browser can be manipulated and may not hold every record. Backend summaries help apply consistent rules across dashboards, filters, and exports. Presentation calculations can still be useful, but must not silently become the only trusted financial logic.

**Q9. What is REST?**  
REST is an architectural style for networked systems that treats information as resources and uses a uniform interface, commonly HTTP methods and status codes. Using HTTP alone does not automatically make an API well-designed REST.

**Q10. What is the difference between 401 and 403?**  
401 means valid authentication is missing. 403 means the caller is authenticated but the action is not allowed. Some applications use 404 to avoid revealing that a protected resource exists; that choice should be consistent.

**Q11. What is a race condition in duplicate checking?**  
Two requests can both check that a budget does not exist before either inserts it. Both may then try to create the same logical budget. Application checks alone cannot guarantee uniqueness under concurrency; a database constraint or unique index is required.

**Q12. What should you inspect if the API returns 500?**  
Use the request time or correlation ID to inspect protected logs, reproduce the request, check validation and database connectivity, identify the exception category, and add a regression test. Do not expose stack traces to the client.

**Q13. Does an existing Flask route prove the feature works?**  
No. It proves code is registered. Exercise the route with valid, invalid, unauthenticated, and unauthorised requests; also verify persistence and response behaviour.

**Q14. Why do we need rate limiting if authentication exists?**  
An authenticated account can still be abused, and login attempts can come from unauthenticated clients. Rate limiting reduces repeated requests but does not replace authentication, authorisation, validation, or monitoring.

## 13. Backend defence checklist

Be able to locate the app entry point, blueprints, JWT decorators, user-scoped expense query, budget/category rules, summary endpoints, CSV export, SQLAlchemy session handling, and health endpoints. For each feature, explain the request, rule, database operation, response code, and test that proves it works.
