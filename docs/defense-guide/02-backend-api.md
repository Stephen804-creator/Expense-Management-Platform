# Session 2 — Backend Development and the REST API

## 1. What the backend is

The backend is the server-side part of the application. It receives requests, checks whether they are valid and permitted, applies business rules, interacts with PostgreSQL, and sends responses. The frontend is not allowed to be the final authority over financial data because a user can change browser code or call an endpoint directly.

In this project, the backend uses Python and Flask. SQLAlchemy provides the database object-relational mapping (ORM), Flask-JWT-Extended supports bearer-token authentication, and Flask-Limiter provides request limits. The application entry point is `backend/app.py`; feature routes are organised into Flask Blueprints.

## 2. The full request lifecycle

A request typically follows this sequence:

1. The browser creates an HTTP request, for example `POST /api/expenses`.
2. Nginx may receive it first and forward it to the backend container.
3. Flask matches the URL and HTTP method to a route.
4. Authentication middleware/decorators verify the token when the route is protected.
5. The route reads and validates the input.
6. The route checks business rules and loads any required records.
7. SQLAlchemy issues database operations through a session.
8. The transaction commits if the operation succeeds; it rolls back after a failed transaction.
9. Flask serialises the result into an HTTP response with an appropriate status code.
10. The frontend interprets that response and updates the UI.

Each step has a distinct responsibility. Nginx routing is not authentication. Authentication is not input validation. Validation is not database integrity. A successful database commit is not proof that the frontend displayed the right result.

## 3. What a REST API means

An API is a defined way for one program to request a service from another. REST-style APIs commonly use HTTP methods to express intent:

- `GET`: retrieve a resource or list;
- `POST`: create a resource or trigger a defined operation;
- `PUT`: replace/update a resource according to the endpoint's contract;
- `PATCH`: partially update a resource where implemented;
- `DELETE`: request removal of a resource.

An endpoint is the combination of a path, method, and behaviour. For example, `GET /api/expenses` and `POST /api/expenses` share a path but perform different operations.

A JSON response is a data format, not a guarantee that the request succeeded. The client must check the HTTP status and response structure.

## 4. Routes and Blueprints in this project

The Flask app registers Blueprints for authentication, categories, expenses, budgets, and summaries. A Blueprint groups related routes so the application entry point does not need to contain every feature's implementation.

Examples visible in the source include:
- authentication routes under `/api/auth`, including registration, login, and current-user lookup;
- expense routes under `/api/expenses`, including create, list, retrieve, update, delete, filtering, and CSV export;
- category routes under `/api/categories`;
- budget routes under `/api/budgets`;
- summary routes for aggregated information;
- `/health` and `/health/db` endpoints in the Flask application.

The exact list of routes and whether each is protected should be checked in the current target branch. Do not assume every route has the same security policy simply because it is in the same application.

## 5. Request validation and response codes

Validation asks whether input has the right shape and meaning. An amount should be numeric, greater than zero, and within the supported precision. A date should be valid. A category ID should refer to an existing category. A month filter should follow the documented format.

Common status codes:
- **200 OK:** request succeeded;
- **201 Created:** a resource was created;
- **400 Bad Request:** request data is malformed or invalid;
- **401 Unauthorized:** authentication is missing, invalid, or expired;
- **403 Forbidden:** the user is authenticated but is not permitted to perform the action;
- **404 Not Found:** the requested resource is absent or intentionally not visible to that user;
- **409 Conflict:** the request conflicts with an existing record or a database rule;
- **429 Too Many Requests:** rate limit was exceeded;
- **500 Internal Server Error:** an unexpected server failure occurred.

A good API returns a useful public message without exposing stack traces, SQL statements, passwords, tokens, or internal environment values.

## 6. Authentication and ownership checks in routes

A route that uses a JWT-required decorator verifies that the request contains a valid token. The handler can then obtain the user identity from the token. The identity should be treated as the authenticated user's identity, not as a user ID supplied freely in the request body.

For personal expenses, the backend should query by both the expense ID and the authenticated user's ID. If it queries only by expense ID, one user might be able to read or alter another user's record by guessing an ID. This is an object-level authorization issue.

The reviewed expense routes use the authenticated identity to scope expense queries. This is a good pattern to explain. However, the category routes in the reviewed baseline are not consistently protected by JWT decorators, and categories appear to be global rather than per-user-owned. That raises a policy question: are categories intended to be shared system data, or should each user own their categories? The team must decide and implement one policy consistently rather than assuming authentication on expenses automatically protects categories.

## 7. Database sessions, commits, and rollbacks

SQLAlchemy's session tracks pending changes. Adding an object to the session does not necessarily mean it has been permanently stored. A commit asks the database to complete the transaction. If a commit fails, the application should roll back the session before attempting another operation.

Why? A failed database transaction can leave the session in a failed state. Reusing it without rollback can cause later operations to fail even if their input is valid.

The correct error-handling pattern is:
- attempt the database operation;
- commit if it succeeds;
- catch expected conflicts specifically;
- roll back on failure;
- return the correct status code;
- log enough diagnostic information on the server without leaking it to the client.

Returning 409 for every exception is wrong because not every exception is a conflict. Returning 500 for a known uniqueness conflict is also poor API behaviour. The correction branch associated with PR #54 adds targeted error handling in some areas; confirm the branch and current tests before claiming the change is on main.

## 8. Query filters and pagination

A query filter narrows a result set, such as filtering expenses by category or month. It must be validated before it is used. A month filter should be converted into a date interval carefully. The common robust pattern is inclusive start and exclusive next-period boundary: `date >= first_day` and `date < first_day_of_next_month`. This avoids errors involving the last day or time component.

As data grows, returning every record can become slow and memory-intensive. Pagination limits each response to a page and makes the API more predictable. If pagination is not currently implemented, present it as a potential improvement rather than an existing feature.

## 9. Why the backend and frontend must share an API contract

The frontend and backend must agree on:
- endpoint paths and methods;
- request field names and types;
- response field names and types;
- authentication requirements;
- error response shape;
- date and amount formats;
- filtering and sorting behaviour.

For example, if the frontend sends `date` but the backend expects `expense_date`, the request may fail or save incomplete information. If the backend returns a 409 but the frontend treats every non-empty response as success, the UI will mislead the user.

The repository's documented frontend uses browser storage, while the backend exposes server API routes. The team should make a route-to-screen map and prove each screen uses the correct API before claiming complete end-to-end integration.

## 10. Tough defense questions

**Q1. Why not put all business logic in React?**  
Browser logic can improve usability, but it can be bypassed. The backend is the trusted place to enforce validation, ownership, uniqueness-sensitive operations, and rules that must apply to every client.

**Q2. What is the difference between 401 and 403?**  
401 means valid authentication has not been established. 403 means the server knows who the caller is but denies that action. Some systems use 404 to avoid revealing that another user's resource exists.

**Q3. Why do we need both an ORM and PostgreSQL?**  
PostgreSQL stores and protects the data. SQLAlchemy maps Python objects to database records and helps construct queries and transactions. The ORM does not replace the database's constraints.

**Q4. Why is a route-level “does this name already exist?” check not enough?**  
Two simultaneous requests can both pass the check before either commits. A database unique constraint is needed to protect integrity under concurrency; the route should convert the expected conflict into a controlled response.

**Q5. What happens if the database commit fails?**  
The operation must not be reported as successful. The session should be rolled back, the client should receive an appropriate sanitized error, and the server should retain enough diagnostic information for investigation.

**Q6. Is hiding an edit button authorization?**  
No. It only changes the UI. The backend must independently enforce permission when the request arrives.

**Q7. Why use Blueprints?**  
They group related routes into modules, making the code easier to navigate and test. They are an organisation mechanism, not a security feature by themselves.

**Q8. What is the purpose of a health endpoint?**  
It gives an external check a simple way to determine whether a service responds. A basic health check may not prove that every dependency or user journey works, so application and database health may need separate checks.

**Q9. Can a green API unit test prove the public website works?**  
No. It proves only the tested backend behaviour under the test conditions. Public routing, Nginx, browser integration, deployment configuration, and the live database require separate checks.

**Q10. How do you explain the category security question honestly?**  
“The current route policy must be checked explicitly. Expense routes are scoped to the authenticated user, while the reviewed category routes are not consistently protected. We need to decide whether categories are intentionally shared or user-owned and test the selected rule. We should not claim that all category operations are private until that is enforced and verified.”

## 11. Explain it in one minute

“The backend is the application's trusted processing layer. Flask receives HTTP requests, routes them to feature-specific Blueprints, validates inputs, checks authentication and ownership, applies business rules, uses SQLAlchemy to interact with PostgreSQL, and returns a response with the correct status code. Transactions must be committed on success and rolled back on failure. The frontend and backend must agree on the API contract. We test the API separately from the browser and deployed system because a route working in isolation does not prove the full user journey works.”
