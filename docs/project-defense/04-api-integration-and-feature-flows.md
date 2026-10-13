# Session 4 — API Integration and End-to-End Feature Flows
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. Why integration is its own topic

A frontend can be correct by itself and a backend can be correct by itself, yet the application can still fail when the two disagree. Integration is the work of making the components communicate using the same routes, field names, data types, authentication rules, and response formats.

For example, React may submit an amount under one property name while Flask reads another. The frontend may expect a successful response to contain a field called data while the backend returns a different shape. A local frontend may call the wrong port after deployment. Each component can build successfully while the end-to-end feature remains broken.

The API contract is the agreement that prevents these mismatches. It should specify the endpoint, HTTP method, headers, request schema, response schema, status codes, authentication requirement, and important business rules.

## 2. Follow a request across the system

A typical deployed request follows this logical route:

**Browser / React → Nginx → Flask API → PostgreSQL → Flask response → React update.**

Nginx is the front door for HTTP traffic and can route requests to the frontend or API. Flask handles application logic. PostgreSQL stores persistent records. The response travels back to the browser, where React decides how to present the result.

Each component has a separate responsibility. Nginx routing does not prove a user is authorized. Flask authorization does not replace database constraints. A database commit does not guarantee the frontend correctly refreshes. A successful frontend build does not prove the API URL is correct in AWS.

When debugging, identify the last point known to work. Did the browser send the request? Did Nginx route it? Did Flask receive it? Did authentication pass? Did the database commit? What status and body came back? Did React handle that response? This is more reliable than randomly changing code.

## 3. Create expense flow

The create flow should be explained in terms of both user experience and data correctness:

1. The user enters amount, date, category, and other supported fields.
2. React checks simple input errors and submits the expected JSON shape.
3. The request includes authentication if required.
4. Flask derives the user's identity from verified authentication, validates the request, checks the category reference and other rules, and creates the record.
5. PostgreSQL enforces foreign keys and other constraints.
6. Flask returns a success response only after the write succeeds.
7. React shows confirmation and refreshes the list and any affected summary.

The user ID should not be trusted merely because it appears in the body. The backend should associate the record with the authenticated identity. Otherwise, a caller could try to create data under another user's account.

## 4. Read, update, and delete flows

**Read:** The backend should return only records the current user is permitted to see. Query filters and ownership checks are important. Hiding another user's row in React after the API returns it is not safe because the data has already been disclosed.

**Update:** The server must confirm the record exists and the caller can change it. It should validate the new values and commit the transaction. If the commit fails, the API should not return a success response. The frontend should update its state only after confirmation.

**Delete:** The server checks authorization and any dependencies, then deletes or rejects the request. In this application, category deletion has a dependency concern: a budget may reference the category. The correction branch adds a pre-check and a database RESTRICT rule, so a category with a linked budget should not be deleted. The expected API response is a conflict such as HTTP 409, and the budget must remain unchanged.

## 5. Authentication across the request

A login flow establishes identity and returns or sets authentication state according to the implementation. For a bearer-token API, the frontend sends the token in the Authorization header. Flask verifies the token before protected routes use the identity.

A token is not a user password, but it is still a credential: anyone who steals a valid bearer token may be able to act as that user until it expires or is invalidated. Do not log it or expose it in screenshots. The team should explain how the frontend stores and sends authentication information only after checking the actual code.

A common integration defect is inconsistent environment configuration. A frontend may point to a local API URL in production, or the backend may read a different environment variable from the one configured in CI or AWS. Check the configuration source, container environment, reverse-proxy routing, and the actual request URL.

## 6. Monthly summaries and budgets

A budget is a rule or limit; an expense is a transaction record. A monthly summary aggregates records for a defined time period. The team must explain the relationship between them without confusing them as the same data.

For example, the amount spent during a month can be calculated from expense records, while a budget stores the allowed limit for that month and perhaps a category. The application can compare the total with the limit to calculate remaining budget or overspending, but the exact formula must match the implementation.

The following questions must have explicit answers:
- Does a whole-month budget have no category, while a category budget has one?
- Can there be more than one budget of the same kind for the same month?
- Which date field determines the month?
- What happens if a category is deleted while a budget references it?
- Do edits and deletions immediately affect totals?
- Does the summary use the same active filters as the visible list?

These are business-rule questions. Do not guess the intended behavior in the defense; check the implementation and agree on the product rule as a team.

## 7. Filtering and date boundaries

Filtering may happen in the backend, the frontend, or in a combination, but the team must know where it happens in this application. Server-side filtering can reduce the amount of data sent to the browser and keeps behavior consistent across clients. Client-side filtering can be convenient for a dataset already loaded, but must not be mistaken for access control.

Date ranges are easy to get wrong. Suppose a user selects October 1 through October 31. If the implementation treats the end date as exclusive without adjusting it, October 31 records may be missing. If timestamps are used, time zones can create boundary errors. Tests should include records exactly on the start and end dates, not just dates in the middle.

The correction branch tightened validation so date-only input must be in canonical ISO format. The team should test both valid values and near-misses, including impossible dates and strings that parse loosely but are not the required format.

## 8. CSV export integration

Export should follow the intended authorization and filtering rules. A user must not be able to alter an ID or query string to export another user's data. The exported file should have predictable column names, a clear date and amount representation, and correct CSV escaping.

CSV has edge cases: values may contain commas, quotation marks, or line breaks. Spreadsheet software may interpret cells beginning with characters such as equals signs as formulas. The team should inspect the actual export code and decide whether a defensive transformation is required. Do not claim that export is secure merely because the file downloads.

## 9. Error mapping and user experience

The backend's status codes are part of the integration contract. React should distinguish validation failures, authentication failures, conflicts, and unexpected errors. If every error is presented as “something went wrong,” users cannot take corrective action. If internal server details are displayed, sensitive implementation information may leak.

A duplicate budget should be represented as a conflict, not an unexplained generic server error. If a category cannot be deleted because a budget depends on it, the message should explain the dependency. Unexpected failures should remain generic to the user while being diagnosed through protected logs.

## 10. Idempotency and retries

A request is idempotent when repeating it has the same intended effect as performing it once. A read is normally idempotent. A delete of an already deleted resource may be designed to return 404 or treat the result as already achieved. A create request can create duplicate records if blindly repeated.

Network failures create an ambiguity: the server may have committed the operation, but the response was lost. If the user clicks Save again, a duplicate may result. Systems can address this through careful UI handling, duplicate constraints, or idempotency keys for operations that require them. Explain what this project actually implements rather than claiming it has idempotency keys without evidence.

## 11. How to debug an integration failure

Use this sequence:

1. Reproduce the issue with a clear input and expected result.
2. Inspect the browser network request: URL, method, payload, headers, status, and response.
3. Check whether Nginx routed the request to the correct service.
4. Inspect Flask logs for the route and error, without exposing tokens or personal data.
5. Check database state and constraints where access is available.
6. Compare the API response with what React expected.
7. Add a regression test so the defect is less likely to return.
8. Repeat the full journey after the correction.

Do not jump straight to the database if the browser never sent a request. Do not blame React if Flask returns a 500 for a failed transaction. Follow the request through each boundary.

## 12. Difficult defense questions

**Q: Why can all services run and the application still fail?**  
They may not agree on URLs, ports, environment variables, request schemas, authentication, or response formats. Service availability is not the same as successful integration.

**Q: What is the difference between unit, integration, and end-to-end testing?**  
Unit tests isolate a small function or component. Integration tests verify components work together, such as a route and database. End-to-end tests exercise a user journey through the running application.

**Q: Why not filter another user's records in React?**  
The API would already have disclosed the records. Access control must be enforced before data is returned.

**Q: If the save response is lost, did the record save?**  
It may have. The client cannot infer failure from a missing response alone. Check the server state or design safe retry behavior.

**Q: Why can the budget be correct in the database but wrong on the dashboard?**  
The frontend may be showing stale state, use a different date range, or interpret the API response incorrectly. Compare the raw API result with the visible calculation.

**Q: What is the most useful evidence for a bug report?**  
The exact steps, input, expected result, actual result, request and response, relevant logs, tested commit/version, and a repeatable test.

## 13. Capstone alignment

The capstone requires an architecture diagram, working feature set, React frontend, Flask REST API, PostgreSQL, reverse proxy, health endpoint, automated pipeline, deployment, and documentation. This session explains the connections between those elements. The handout originally names Azure VM, but the tutor approved AWS as the alternative; the actual AWS deployment and its evidence should be described honestly.
