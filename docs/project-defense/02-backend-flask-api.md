# Session 2 — Backend Development with Flask REST API
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. What is the backend?

The backend is the server-side part of the application. It receives requests, checks whether they are valid and authorized, applies business rules, reads or changes persistent records, and returns a response. This project uses Flask to expose a REST-style HTTP API and PostgreSQL to store persistent records.

Think of a bank counter. A customer can request a withdrawal, but the system must check identity, account rules, and whether the transaction can complete. A user interface cannot be trusted to make those decisions. Likewise, Flask must not accept an expense update merely because React displayed an Edit button.

## 2. What Flask does

Flask is a Python web framework. It receives HTTP requests and maps them to application functions. A route associates a URL and HTTP method with a function. A Blueprint groups related routes into a module, such as authentication, expenses, budgets, categories, or summaries. This separation makes the code easier to navigate than one enormous file.

A request handler should follow a controlled sequence: parse the request, validate input, establish the authenticated user when required, enforce authorization and business rules, perform the database operation, handle expected failures, and return an appropriate response.

A request body is data supplied by the caller, not trusted truth. A caller can send missing fields, wrong types, invalid dates, non-finite numbers, an identifier belonging to another user, or unexpected fields. The server must decide what is acceptable.

## 3. What is a REST-style API?

An API is an interface through which software communicates. In a REST-style API, URLs identify resources and HTTP methods describe the intended operation. Typically GET reads, POST creates, PUT or PATCH updates, and DELETE removes. The exact route design is defined by the application contract.

A method alone does not make an operation safe. DELETE still requires authentication, authorization, validation, and database constraints. GET should not unexpectedly modify records.

The frontend and backend need an **API contract**: endpoint path, method, required headers, accepted request fields, response structure, status codes, and error behavior. If React sends a field named expense_date while Flask expects a different field, or the frontend expects an object while the API returns an array, the feature may fail even though both projects build.

## 4. HTTP requests and responses

An HTTP request has a method, path, headers, and sometimes a body. Headers may include content type or authentication information. The body often carries JSON data.

The response has a status code, headers, and often a JSON body. Common meanings include:

- **200 OK:** The request succeeded.
- **201 Created:** A resource was created.
- **400 Bad Request:** Input is malformed or invalid.
- **401 Unauthorized:** Authentication is missing or invalid.
- **403 Forbidden:** The caller is authenticated but lacks permission.
- **404 Not Found:** The resource is unavailable at that endpoint. Some applications intentionally avoid revealing another user's resource.
- **409 Conflict:** The request conflicts with current data or a business rule, such as a duplicate budget or an attempt to delete a category still referenced by a budget.
- **500 Internal Server Error:** An unexpected server-side failure occurred.

These codes help the frontend decide whether to display validation feedback, ask the user to sign in, explain a conflict, or show a general failure. Returning 200 for every outcome makes error handling unreliable. Returning 500 for every exception also hides the difference between expected conflicts and genuine failures.

## 5. Validation, business rules, authentication, and authorization

These concepts answer different questions.

**Validation:** Does the submitted value have an acceptable shape and value? For example, is an amount finite and is the date valid?

**Business rule:** Is the operation allowed by the product's rules? For example, is there already a monthly budget, or can a category be deleted while a budget depends on it?

**Authentication:** Who is making the request?

**Authorization:** Is that identified user allowed to perform this action on this record?

A request may pass validation but fail authorization. An expense ID can be a valid integer but belong to another user. The backend must scope queries and writes to the verified identity, not trust a user ID supplied in the body.

## 6. Database sessions and transactions

The application uses a database session to work with SQLAlchemy models. A session tracks changes and sends SQL statements to PostgreSQL. A transaction groups database operations so they can be committed or rejected.

A commit asks the database to make the transaction permanent. It can fail even when the Python code appeared correct: a unique constraint may reject a duplicate, a foreign key may reject a missing reference, or the database connection may fail.

After a failed SQLAlchemy transaction, the session normally must be rolled back before reuse. Catching an error without rollback can leave the session in a failed state and cause later operations to fail too.

The backend should distinguish known integrity conflicts from unexpected errors. A known duplicate under a particular unique constraint may be returned as HTTP 409. An unrelated database failure must not be falsely described as a duplicate. Detailed SQL or exception text should not be sent to the client; it may expose schema or implementation details. Protected server logs can preserve safe diagnostic context.

## 7. Example: creating an expense

A robust create operation should follow this sequence:

1. Require valid authentication if the endpoint is protected.
2. Parse the request and confirm the body has the expected structure.
3. Validate amount, date, description, and optional category ID.
4. Confirm that referenced records exist and follow the product's category policy.
5. Associate the expense with the authenticated user from the verified token—not a user ID blindly accepted from the body.
6. Add the model to the session and commit the transaction.
7. If a known constraint conflict occurs, roll back and return the correct conflict response.
8. If an unexpected error occurs, roll back, log safe diagnostic information, and return a generic server error.
9. Return a response that allows React to update the interface.

Before the commit succeeds, the backend must not tell the user the record is safely saved.

## 8. Ownership and authorization

Expense Tracker manages personal financial records, so user isolation is central to correctness. A logged-in user must not be able to retrieve another user's expenses by changing an ID in a URL, query parameter, or JSON body.

A safe pattern is to derive the current user from verified authentication and use that identity in database queries. An update can look up the expense using both its ID and the authenticated user's ID, or fetch it and explicitly verify ownership before modifying it. Delete operations need the same protection.

A global category catalog and a per-user category system are different product designs. The team should describe the policy actually implemented and check that all category routes follow it consistently. Do not invent a per-user ownership rule if the current implementation treats categories as shared. The essential point is that the policy must be explicit, enforced server-side, and tested.

## 9. Error handling

Errors should be useful without leaking secrets. The client may receive “A budget already exists for this month” with status 409. It should not receive a database connection string, SQL statement, stack trace, JWT secret, or full exception dump.

Server logs should provide enough context for diagnosis while avoiding passwords, bearer tokens, and unnecessary personal financial data. A broad exception handler is not automatically robust: if it does not roll back, classify known errors, or preserve useful safe diagnostics, it can hide defects.

## 10. Why database constraints matter even when Flask checks first

Suppose Flask checks whether a monthly budget exists before inserting one. Two requests can arrive almost simultaneously. Both may check before either commits, and both may believe the budget is absent. This is a race condition.

A database unique constraint or unique index protects the rule at the point where concurrent writes are resolved. Flask can then translate a known conflict into a useful response. The route improves feedback; the database enforces the invariant.

Similarly, a category-delete pre-check is helpful, but a foreign-key constraint protects against concurrent requests or alternate code paths. The proposed correction adds a route-level check and migration 004 with ON DELETE RESTRICT so a category referenced by a budget cannot be deleted underneath that budget.

## 11. Connections to other parts

- **React:** Calls the API and interprets the returned status and data.
- **PostgreSQL:** Enforces persistent constraints and stores committed records.
- **Authentication:** Supplies a verified identity for authorization decisions.
- **Nginx:** Routes external traffic; it does not replace Flask's access checks.
- **Docker Compose:** Starts the backend with its required services and configuration.
- **GitHub Actions:** Runs automated checks against proposed changes.
- **AWS:** Hosts the deployed environment; server settings and secrets must be supplied securely.
- **QA:** Checks successful journeys and negative cases such as invalid input, expired tokens, duplicate data, and cross-user access.

## 12. Difficult defense questions

**Q: Why can a valid HTTP request still be rejected?**  
Valid syntax does not guarantee valid business meaning or permission. The amount may be invalid, the record may not exist, a duplicate may conflict with a rule, or the caller may not own the record.

**Q: Why not validate only in React?**  
The API can be called without the React interface. Server-side validation is the trusted enforcement point.

**Q: Why should a duplicate budget return 409 instead of 500?**  
A known duplicate is a predictable conflict with current state, not an unexpected crash. The frontend can explain the conflict.

**Q: Why keep database constraints if Flask checks for duplicates?**  
Concurrent requests can pass the application check at the same time. The database constraint is the final guarantee.

**Q: What should happen when commit raises an error?**  
Roll back the failed transaction, classify known errors, avoid leaking internal details, and never report success for a failed write.

**Q: What is the difference between 401 and 403?**  
401 means valid authentication has not been established; 403 means the caller is known but lacks permission. Some systems use 404 to avoid revealing the existence of another user's record.

**Q: What protects an expense if a user changes its ID in the URL?**  
The backend must scope the query or verify ownership against the authenticated identity. An unpredictable ID is not a substitute for authorization.

**Q: Does Docker make the API secure?**  
No. Docker packages and isolates processes. It does not automatically validate input, authorize users, secure tokens, or fix application defects.

## 13. What to demonstrate

Show a successful create/read/update/delete journey, then a negative case such as invalid input, an unauthenticated request, or an attempt to access another user's record. Point to the route and test responsible for the behavior. Clearly distinguish automated checks from manual checks.

## 14. Capstone alignment

The capstone requires a Flask REST API connected to PostgreSQL, the expense-management features, authentication, health checks, secure environment variables, testing, Docker-based deployment, and CI/CD. The backend is the policy and data-processing boundary that joins these requirements. The handout originally says Azure VM; the tutor approved AWS as an alternative, so describe AWS as the approved target rather than misquoting the PDF.
