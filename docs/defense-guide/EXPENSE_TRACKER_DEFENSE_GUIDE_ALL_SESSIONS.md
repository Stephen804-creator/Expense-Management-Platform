# Expense Tracker — Complete Project Defense Study Guide

This file combines the eight individual study sessions. For easier reading, each session is also available as its own Markdown file in this folder.

## Contents

1. [Session 1 — Frontend Development and React](#session-1)
2. [Session 2 — Backend Development and the REST API](#session-2)
3. [Session 3 — PostgreSQL, Data Modelling, and Data Integrity](#session-3)
4. [Session 4 — Authentication, Authorization, Validation, and Security](#session-4)
5. [Session 5 — Expense Tracker Features and Business Logic](#session-5)
6. [Session 6 — Testing, Quality Assurance, and Continuous Integration](#session-6)
7. [Session 7 — Docker, Nginx, GitHub Actions, and AWS Deployment](#session-7)
8. [Session 8 — System Integration, Capstone Alignment, and Difficult Defense Questions](#session-8)

---

# Session 1 — Frontend Development and React

## 1. What the frontend is

The frontend is the part of the Expense Tracker that a person sees and uses in a browser. It includes the page layout, navigation, forms, buttons, transaction lists, budget indicators, charts, messages, and the interaction that happens after a user clicks something.

Imagine a restaurant. The frontend is like the menu and the waiter who receives the order. It helps the customer express what they want and shows the result. The kitchen and store room are different parts of the system. In a web application, the backend validates and processes requests, and the database stores persistent records. A good-looking page is not proof that data has been saved safely.

The project frontend uses React, JavaScript, Vite, Tailwind CSS, Lucide React icons, and Recharts. The documented UI includes dashboard summaries, income and expense pages, budgets, categories, a unified transaction ledger, and monthly reports.

## 2. What React contributes

React is a JavaScript library for building user interfaces from components. Instead of writing one enormous page, a developer divides the interface into smaller pieces that can be understood and reused.

A **component** is a reusable piece of UI. Examples include a navigation sidebar, a summary card, a budget progress indicator, a confirmation dialog, or a transaction row. A component usually receives information, renders some UI, and may respond to user events.

A **parent component** can pass information to a child through **props**. Props are inputs. The child should not silently change the parent's data; it should notify the parent through a callback when the user performs an action.

**State** is data that can change while the application is running. For example, the currently selected month, text typed into a search box, whether a modal is open, or the transactions currently loaded. When React state changes, React can render the affected UI again.

A useful distinction:
- Props answer, “What information did the parent give this component?”
- State answers, “What changeable information does this component need to remember?”
- A normal local variable does not automatically make React re-render when it changes.

## 3. The React concepts a tutor may probe

### JSX

JSX is a syntax that lets a developer describe UI in JavaScript using markup-like expressions. It is not HTML sent directly to the browser unchanged. The build tool transforms the source into browser-executable JavaScript.

### Hooks

- `useState` stores component state.
- `useEffect` runs side effects after rendering, such as synchronising with browser storage or reacting to an external system.
- `useMemo` can cache a calculated value between renders when its dependencies have not changed. It is an optimisation, not a correctness or persistence mechanism.

Hooks must be used according to React's rules; they should not be called conditionally in a way that changes their order between renders.

### Context

React Context lets a value be made available to components lower in the tree without passing the same prop through every intermediate component. This project documents context providers for authentication and toast notifications. Context is useful for shared UI state, but it is not a database and does not replace backend authorization.

### Event handling

A click handler or form-submit handler responds to a user action. For a form, the handler should read the submitted values, validate them, prevent unintended page reload where appropriate, submit or save the data, then show a result. If an operation fails, the UI should not pretend that it succeeded.

## 4. How a form should work

Consider adding an expense:

1. The user enters an amount, category, date, and optional description.
2. The frontend checks basic input rules and provides understandable feedback.
3. The frontend sends the data to its configured persistence layer.
4. The backend, if used, independently validates the request. Frontend validation is not a security boundary because a user can bypass the UI and send a request directly.
5. The backend commits the valid record to PostgreSQL and returns a response.
6. The frontend checks the response status, updates its displayed data, and shows success or error feedback.
7. If the user refreshes or opens another device, the data should still exist if it was successfully saved to the shared backend database.

The last two steps expose an important distinction in this repository. The frontend README describes state stored in browser `localStorage`, while the Flask backend separately provides database-backed API endpoints. Browser storage and PostgreSQL are not the same data store. Before claiming that an expense submitted in the visible UI is saved to PostgreSQL, verify that the relevant React handler actually calls the API and that the response is used correctly.

## 5. Browser storage versus a backend database

`localStorage` is a key-value store inside one browser profile on one device. It can be useful for preferences, theme selection, or a local prototype. It is not a shared multi-user database, does not enforce server-side permissions, and can be cleared or edited by the user.

PostgreSQL is a server-side database. If the application sends a record to the Flask API and the backend commits it to PostgreSQL, the record can persist across browsers and devices subject to the account's permissions and the database's availability.

A common defense trap is saying, “We use PostgreSQL, so every screen saves to PostgreSQL.” That conclusion does not follow from the existence of PostgreSQL files. Trace the specific UI action to the actual API call, backend route, and database commit.

## 6. Validation, accessibility, and responsive design

Frontend validation improves usability: required fields, readable date formats, sensible error messages, disabled submit buttons while saving, and confirmation before destructive actions. The backend must repeat security- and integrity-critical validation.

Responsive design means the interface adapts to different viewport sizes. It is not just making the page smaller; the developer must ensure that navigation, forms, tables, charts, and modals remain usable on mobile and desktop.

Accessibility includes semantic headings, labels associated with inputs, keyboard operation, visible focus, meaningful button text, and not relying on colour alone to convey whether a budget is over its limit. An icon-only button should have an accessible name.

## 7. What can go wrong in the frontend

- **Stale state:** a displayed total is calculated from an old array after a mutation.
- **Incorrect effect dependencies:** a side effect runs too often or fails to run when its input changes.
- **Duplicate submission:** a slow request allows the user to submit the same expense twice.
- **False success:** the UI displays success before the server confirms the save.
- **Unsafe rendering:** untrusted text is inserted as executable HTML rather than rendered as text.
- **Broken error handling:** a failed API request leaves the screen looking as though data was saved.
- **State and database drift:** browser storage contains a record that the server does not, or vice versa.
- **Missing tests:** a build can pass even though a user journey is functionally broken.

## Important repository detail: two frontend source locations

The repository contains both a root-level `src/` directory and a `frontend/src/` directory. The root `package.json` can build the root Vite app, but the Docker Compose configuration builds the frontend service from `./frontend`. That means the frontend packaged by the Compose deployment is the one under `frontend/`, not automatically the root-level `src/` version.

This matters in a defense: do not point to a component in the root `src/` tree and assume that it is running on AWS. Check the Docker build context, the active workflow, and the deployed image. The root README describes localStorage-backed records, while `frontend/src/App.jsx` includes state loaded through the Flask API. These are different persistence designs. The team should identify which interface is actually deployed and keep the README aligned with it.

**Likely tutor question: “Why do you have two frontend folders, and which one is running?”**  
A strong answer is: “The repository currently contains two frontend source trees, so we verify the actual entry point rather than guessing. Docker Compose builds `./frontend`, so that is the frontend used by that Compose configuration. We should consolidate or clearly label the unused tree and update the README to avoid describing one version while deploying another.”

## 8. How the frontend connects to the other sections

React is responsible for presentation and interaction. The API contract connects it to Flask. Authentication supplies the token needed for protected requests. PostgreSQL is the durable source of truth when the feature is truly backend-connected. Nginx can route browser requests to the frontend or API. Docker packages the frontend. GitHub Actions can lint and build it. AWS hosts the running deployment.

If any link is missing, the application may look complete but fail as a full-stack product. For example, a budget card can render perfectly while displaying browser-only data that is absent from PostgreSQL.

## 9. Capstone and AWS alignment

For a credible demonstration, show the actual frontend in the deployed application, perform a real action, refresh the page, sign in as another user if supported, and show that the first user's records are not exposed. Capture the exact deployed URL and the commit or image version tested. Do not present a local Vite development server as proof of an AWS deployment.

The frontend checks commonly include:
- install dependencies using the lockfile;
- run the configured lint command;
- run the production build;
- test real user journeys in a browser;
- confirm mobile layout and keyboard-accessible controls;
- confirm API failures are displayed honestly.

The package currently lists lint and build scripts but no dedicated frontend test script. Therefore, a green build is useful evidence, but it is not the same as a passed browser end-to-end test suite.

## 10. Tough defense questions and strong answers

**Q1. Why use React instead of writing one HTML page?**  
React supports reusable components and state-driven rendering. That helps keep a multi-page dashboard consistent and makes interactive features easier to organise. It is not automatically better for every website; it is appropriate here because the interface contains multiple interactive views and shared UI.

**Q2. What is the difference between state and props?**  
Props are inputs passed into a component by its parent. State is data the component or its state owner manages over time. Both influence rendering, but they have different ownership and update rules.

**Q3. Does `useEffect` save data to a database?**  
No. `useEffect` is a mechanism for running side effects. It might synchronise with browser storage or call an API if written to do so. Persistence depends on the actual operation and its successful completion.

**Q4. Why is frontend validation not enough?**  
Users can bypass the page and send arbitrary HTTP requests. The backend must validate data and enforce permissions because it is the trusted boundary protecting the database.

**Q5. If a record remains after refreshing the page, does that prove PostgreSQL works?**  
No. It might have been read from `localStorage`. To prove PostgreSQL persistence, trace the network request, verify the backend response, and confirm the record is retrieved from the backend/database in a controlled test.

**Q6. Why use Context?**  
Context makes selected shared values available to multiple components without repeatedly passing props through every layer. It does not replace a server-side user identity or authorization check.

**Q7. What is the difference between a production build and a running production app?**  
A build proves the source can be compiled into deployable assets. It does not prove those assets were deployed, that Nginx serves them, that the API is reachable, or that the app behaves correctly in a browser.

**Q8. How would you debug a dashboard total that is wrong?**  
First identify the displayed number and its source. Inspect the transaction data, filters, date boundaries, and aggregation formula. Compare the frontend calculation with the backend response and database records. Reproduce with a small known dataset and test edge cases such as an empty month.

**Q9. Why not store passwords or trusted authorization decisions in React state?**  
The browser is controlled by the user and its state can be inspected or altered. The frontend may hold temporary display state or an access token according to the design, but the backend must verify authentication and authorization for protected data.

**Q10. What would you improve first?**  
I would first prove and standardise the frontend-to-API integration for each feature, then add repeatable browser tests. A polished screen is not enough if the persistent data flow is incomplete.

## 11. Explain it in one minute

“The frontend is the user's interface. We use React components to divide the dashboard into manageable pieces, state to represent changing UI data, props to pass information, and Context for selected shared concerns such as notifications or authentication state. The frontend must validate input for usability, call the correct persistence layer, and show the server's actual result. Our repository documents localStorage-based frontend state as well as a Flask/PostgreSQL backend, so we must verify that each feature is truly API-connected before claiming database persistence. We validate that through network inspection, backend tests, browser journeys, and a deployed demonstration.”

---

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

## 10. How this backend session connects to the capstone

The capstone is not satisfied merely because a Flask file exists. The backend must participate in a working path from the React screen to the PostgreSQL database and back.

- **React frontend:** calls API endpoints and displays returned data or error messages. The frontend must use the actual endpoint paths and response shape.
- **Flask REST API:** validates incoming data, checks identity and permissions, applies business rules, and calls the database layer.
- **PostgreSQL and SQLAlchemy:** store and retrieve records. A successful API response should reflect a completed database operation when the feature claims to save data.
- **Nginx:** routes public `/api/` requests to Flask and normal browser requests to the frontend. An endpoint that works directly on Flask can still fail through Nginx.
- **Docker Compose:** runs the frontend, backend, database, and proxy as connected services. Service names and health dependencies matter.
- **GitHub Actions:** provides automated evidence such as linting, tests, and builds. Passing CI does not itself prove that AWS is running that exact version.
- **AWS deployment:** the running server, environment variables, database, health routes, and deployed commit/image must agree with the version being demonstrated.

**A complete backend demonstration:** log in, create an expense with a valid amount/date/category, show the API response, retrieve the expense again, then explain where the ownership filter is enforced and how the database persists the record. Do not rely on a screenshot of a form as proof that the API or database works.

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

---

# Session 3 — PostgreSQL, Data Modelling, and Data Integrity

## 1. What the database is responsible for

A database is the organised, persistent store for application records. In the Expense Tracker, those records may include users, expenses, categories, budgets, and calculated summaries. The database must preserve relationships and reject invalid combinations even if a route contains a programming mistake or two requests arrive at the same time.

The project uses PostgreSQL with SQLAlchemy. PostgreSQL is the database engine; SQLAlchemy is the Python layer that helps the backend represent rows as objects and execute database operations. They are related, but they are not the same technology.

A record in React state or browser `localStorage` is not automatically a PostgreSQL record. To claim database persistence, the feature must call the backend, and the backend must successfully commit the record.

## 2. Tables, rows, columns, and keys

A **table** holds records of one general kind. A **row** is one record. A **column** is one attribute of that record. For example, an expense row may contain an ID, user ID, category ID, amount, description, expense date, and creation timestamp.

A **primary key** uniquely identifies a row. It lets other records refer to it reliably even when two people have similar names or descriptions.

A **foreign key** is a database rule that says a value in one table must refer to an allowed row in another table. An expense's `category_id`, for example, should refer to a real category. Foreign keys prevent orphaned relationships.

A **unique constraint** prevents two rows from sharing a value or combination of values that must be unique. A **check constraint** can enforce a rule such as a positive amount. Application validation is useful, but database constraints protect integrity when concurrent requests or other code paths bypass a pre-check.

## 3. Why relationships matter

An expense belongs to a user. It references a category. A budget belongs to a user and a month, and may optionally reference a category. These relationships affect what can be deleted and how totals are calculated.

Suppose a budget of ₦50,000 is for “Transport” in October. If deleting the Transport category silently sets that budget's category ID to null, the budget might now be interpreted as a whole-month budget. That changes the meaning of saved data without the user editing the budget. This is not just a UI problem; it is a data-integrity problem.

The correction proposed in PR #54 adds migration 004 to change the category-budget foreign key to `ON DELETE RESTRICT`, so a category referenced by a budget cannot be deleted until the relationship is handled. It also adds a route-level conflict response. This correction must be described as proposed until the relevant branch is merged and deployed.

## 4. What SQLAlchemy does

SQLAlchemy's ORM maps database tables to Python model classes. A model defines how the application represents a row and which columns or relationships exist. For example, an `Expense` model may map to the `expenses` table.

Typical operations include:
- query a user's expenses;
- create an object and add it to the session;
- modify fields on an existing object;
- delete a record;
- commit or roll back a transaction.

The ORM makes application code easier to write, but the actual database still enforces constraints and stores the data. An ORM model and a SQL migration must agree. If the model says one thing and the deployed database schema says another, the app can behave differently between environments.

## 5. Transactions: commit and rollback

A transaction groups database operations into a unit. The aim is that the operation completes consistently or does not partially apply.

For an expense creation:
1. Validate the amount, date, category, and authenticated user.
2. Create the database object.
3. Add it to the session.
4. Commit the transaction.
5. Return success only after the commit succeeds.

If the commit fails, roll back. A rollback tells the database session to abandon the failed transaction state. Without it, later requests may continue to fail even after the original problem has passed.

For operations involving multiple records, transaction boundaries matter even more. If a process updates a budget and a ledger record, the team must define whether both updates must succeed together. Partial updates can make totals inconsistent.

## 6. Money, decimals, and dates

Financial amounts should not be handled carelessly with binary floating-point arithmetic. Decimal values such as 0.1 cannot always be represented exactly as binary floats. The backend's expense route uses Python's `Decimal` for amount validation, which is a sensible approach for decimal money values. The database column precision and serialisation should also be checked to ensure the value is stored consistently.

The API should define the accepted precision. If the application supports two decimal places, a request with three decimal places should be rejected or handled by a documented rounding rule. Silent rounding can produce unexpected totals.

Dates also need a clear contract. The backend uses ISO-like date strings in API responses and supports month filtering. A month filter should use a precise range: from the first day of the requested month up to, but not including, the first day of the next month. This avoids missing records on the final day.

The application should distinguish an **expense date** (when the transaction happened) from a **created timestamp** (when the record was stored). Those values answer different questions.

## 7. Budget uniqueness and concurrent requests

A user should not accidentally create two indistinguishable budgets for the same month and category if the business rule permits only one. The intended unique key may include user, month, and category. But a whole-month budget has a null category, and database engines differ in how unique constraints treat nulls. The team must inspect the actual schema and use a constraint/index strategy that correctly handles both category-specific and whole-month budgets.

An application-only duplicate check is not race-safe. Two requests can arrive at almost the same time, both see no existing budget, and both attempt to insert. A database unique constraint is the final protection. The backend should then return 409 for the expected duplicate conflict and 500 for an unexpected database failure.

## 8. Migrations and why they exist

A migration is a controlled change to the database schema, such as adding a column, index, table, or changing a foreign key. It allows the team to evolve the schema in a repeatable way instead of relying on someone manually editing a live database.

This repository includes numbered SQL migration files. The CI workflow applies migrations in order against a fresh PostgreSQL service and checks for expected tables. Docker's PostgreSQL initialization directory behaves differently: initialization scripts run when a new data directory is created. If a persistent volume already exists, adding another SQL file to that directory does not automatically rerun it.

Therefore, “the migration file exists” does not mean “the production database has the migration.” A deployment process must explicitly apply pending migrations and verify success before restarting the app. Test both a clean database and a database with existing data.

## 9. Persistent volumes, backups, and recovery

A Docker container is replaceable. A PostgreSQL data volume holds data beyond the lifetime of a container. Removing and recreating the database container should not erase the volume, but deleting the volume can erase the stored data.

Backups are separate from persistent storage. A volume protects against some container lifecycle events; it does not protect against accidental deletion, disk failure, corruption, or a destructive migration. A production plan should define backup frequency, retention, access control, and restore testing.

Before a schema change on a real database, the team should know:
- what data the migration changes;
- whether it is backward compatible;
- how long it may lock tables;
- what happens if it fails halfway;
- whether rollback is possible;
- how to restore from backup;
- how the running app behaves if the new schema is not ready.

## 10. Data integrity examples the tutor may use

**Example A: Category does not exist.** The backend should reject an expense referencing a nonexistent category. The database foreign key is a final safety net.

**Example B: Two requests create the same budget.** A unique constraint prevents both from being stored when the business rule says one is allowed. The API returns a controlled conflict.

**Example C: Database commit fails.** The route rolls back and does not return success. A later valid request should still work.

**Example D: Category deletion is attempted while an expense or budget references it.** The database and API should follow a documented policy. A conflict response is often safer than silently changing the meaning of financial records.

**Example E: Migration file is added to a running AWS instance.** The database will not necessarily run it automatically just because the file exists. The deployment process must apply it and capture the result.

## 11. How the database session connects to the capstone

PostgreSQL is the persistence layer in the capstone architecture. SQLAlchemy is the Python layer that maps backend operations to database queries, but it does not replace PostgreSQL or make constraints unnecessary.

- **React and Flask:** React sends a request; Flask validates it and calls the ORM; SQLAlchemy issues SQL; PostgreSQL checks constraints and stores the change; the API returns the result.
- **Docker Compose:** the database service must be healthy before the backend depends on it. The named volume is important because deleting and recreating a container should not automatically erase persistent records.
- **Migrations:** a SQL file mounted into PostgreSQL's initialization directory normally runs only when the database data directory is initialized for the first time. An existing volume needs a deliberate migration process; adding a file does not mean it automatically ran.
- **GitHub Actions:** migration checks in a temporary test database prove the tested migration sequence in that environment. They do not prove that an existing AWS database was upgraded successfully.
- **AWS:** credentials belong in protected runtime configuration, not source code. The team must verify backups, volume persistence, and a recovery procedure for the actual server.

**Capstone proof:** show the schema and constraints, run a known API operation, read back the stored row, and show migration/test evidence. If asked whether production data is safe, explain the actual backup and recovery process; do not claim backups exist unless there is evidence.

## 11. Tough defense questions

**Q1. Why do we need foreign keys if Flask checks that a category exists?**  
The route check improves the error message, but a race or another code path can still change the data between the check and insert. A foreign key enforces the relationship at the database boundary.

**Q2. What is the difference between a primary key and a unique constraint?**  
The primary key is the row's main identity and cannot be null. A unique constraint protects a uniqueness rule for one or more columns. A table can have one primary key and multiple unique constraints.

**Q3. Why not store every amount as a JavaScript number and trust it?**  
Browser numbers use floating-point representation and are user-controlled. Financial values need defined precision, server validation, and suitable database types. The backend's use of `Decimal` helps with decimal input validation, but the whole path must be checked.

**Q4. What is a migration?**  
A versioned, repeatable database schema change. It keeps environments in sync and makes the evolution of the schema reviewable.

**Q5. Why does adding a SQL file not update an existing database automatically?**  
The PostgreSQL image's initialization scripts are designed for initial database creation. Existing persistent data directories are not treated as brand-new databases, so deployment must explicitly run migrations.

**Q6. What does rollback mean?**  
It abandons the current failed transaction state so it does not remain pending and block subsequent work. It does not automatically undo a transaction that was already committed.

**Q7. Why is a unique index still needed if the route checks for duplicates?**  
Because concurrent requests can both pass the check. The database is the final authority for a uniqueness rule.

**Q8. How would you prove migration 004 works?**  
Apply it to a clean database and to a copy of a database with existing data. Inspect the foreign-key delete rule, create a budget linked to a category, attempt deletion, and confirm that the request fails safely while the budget retains its original category and amount.

**Q9. What is the difference between persistence and backup?**  
Persistence means the data remains available across normal application restarts. Backup is a separate recoverable copy used when data is lost or damaged. One does not replace the other.

**Q10. If CI says migrations passed, can we say AWS is safe?**  
No. CI proves the migration worked in the test database it created. AWS may have existing data, different configuration, locks, or a different migration history. Runtime evidence is still required.

## 12. Explain it in one minute

“PostgreSQL stores the durable records. Tables represent entities, primary keys identify rows, foreign keys preserve relationships, and unique constraints protect rules such as one budget per user, month, and category. SQLAlchemy lets Python work with these records, but PostgreSQL remains the final authority on database constraints. A transaction should commit only when the operation succeeds and roll back on failure. Migrations change schemas in a controlled order, and existing Docker volumes need explicit migration execution. We prove database correctness with migration tests, constraint checks, rollback tests, and safe tests against existing data.”

---

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

## 12. How security connects to the capstone

Security must hold across the full deployed request path, not just on the login screen.

- **React:** collects credentials and sends the token with protected API calls. Client-side hiding of buttons is not authorization.
- **Flask:** validates credentials, verifies the JWT, derives the current user identity, checks ownership/permissions, validates input, and returns safe errors.
- **PostgreSQL:** stores users and application records. Foreign keys and constraints protect data relationships, but they do not by themselves prove that a request is allowed to read a record.
- **Nginx and networking:** route requests and should be configured with safe headers and an appropriate HTTPS/TLS arrangement for public use.
- **Docker and AWS:** production secrets must be injected securely; placeholder secrets and development defaults must not be used for a public deployment. The public server should expose only the ports and services required.
- **Testing and CI:** include unauthenticated access, invalid tokens, cross-user record access, malformed values, and secret/configuration checks. A green build without these cases is not proof of complete security.

**Capstone proof:** demonstrate that user A cannot read, edit, delete, or export user B's records by changing an ID or calling the API directly. Show the exact test result and the deployed configuration evidence available to the team.

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

---

# Session 5 — Expense Tracker Features and Business Logic

## 1. Start with the problem the application solves

The Expense Tracker is intended to help a person record spending, organise it by category, set budgets, review totals, and understand patterns over time. The point is not simply to store numbers. The application should help users answer practical questions: How much did I spend this month? Which category used the most money? Am I close to my budget? Can I retrieve or export the transactions behind a total?

A defense answer should connect every feature to a user problem, the data it needs, the rule it applies, and how the team proves the result is correct.

## 2. Expense records

An expense generally needs an identity, owner, category, amount, date, and optional description. These fields serve different purposes:
- ID identifies the record.
- User ID determines ownership.
- Category groups the spending.
- Amount records the monetary value.
- Expense date says when the spending occurred.
- Description gives human-readable context.
- Created timestamp says when the system stored it.

The backend expense routes validate amount and date input and scope list and detail queries to the authenticated user. An expense should not be saved if its category is invalid or if its amount violates the documented rules.

The user ID must come from verified authentication, not from an arbitrary user-supplied field. Otherwise a caller might create records under another person's identity.

## 3. Categories

Categories group similar expenses, such as food, transport, utilities, or education. Categorisation supports filtering, charts, budget limits, and reports. A category is not the expense itself; many expense rows may reference one category.

A category deletion policy must consider the records that reference it. If a category is used by existing expenses or budgets, deleting it can break history or alter the meaning of a budget. The team should decide whether to prevent deletion, archive the category, or reassign linked records. That choice should be documented and tested.

The reviewed baseline returns a conflict when category deletion fails because an expense references the category. The separate budget relationship needs its own protection. The correction proposed in PR #54 adds a budget-reference check and a database-level `ON DELETE RESTRICT` rule through migration 004. Confirm whether this correction has reached the branch being demonstrated.

## 4. Budgets

A budget is a spending limit for a defined period. The project backend uses a month and may allow a budget linked to a category or a whole-month budget without a category.

A category budget asks, “How much should I spend on transport in October?” A whole-month budget asks, “What is my total spending limit for October?” These are different meanings, even if both use the same amount type.

To determine budget usage, the application needs a clear rule. A category budget should sum qualifying expenses for that user, category, and period. A whole-month budget should sum qualifying expenses for that user and period. The calculation must not include another user's records, the wrong month, or records outside the selected category.

The remaining amount is commonly:
`remaining = budget_limit - eligible_spending`

A negative remaining amount means spending exceeded the limit. The percentage used is commonly:
`percentage_used = eligible_spending / budget_limit × 100`

The implementation must decide how to display percentages above 100%, how to handle a zero or missing limit, and how to handle rounding. It should not divide by zero.

The budget's uniqueness rule also matters. If only one budget is allowed for a user/month/category, both the API and database need to enforce it. Whole-month budgets need deliberate handling because their category ID may be null.

## 5. Income and expenses are not interchangeable

The frontend README documents income management and a unified transaction ledger. The reviewed backend route registration includes authentication, categories, expenses, budgets, and summaries, but no dedicated income route is visible in that registration. The database migration check also names tables for budgets, categories, expenses, monthly summaries, and users, but not an income table.

This does not prove that every income-related screen is broken, but it does mean that the team must not assume income has the same backend persistence as expenses. Check the actual React income handler, API calls, backend routes, and database schema. If income remains in browser storage, say so clearly and do not describe it as a server-persisted feature.

A professional defense acknowledges a gap precisely: “The frontend documents income functionality, but backend persistence for income is not established by the current route and schema evidence. We are checking and aligning the feature before claiming full-stack persistence.”

## 6. Dashboards and reports

A dashboard summarises underlying records. It may show total income, total expenses, net savings, budget status, and recent transactions. A report groups records by a time period and may break spending down by category.

A total is only as correct as the records and filters used to calculate it. To test a monthly report:
1. Create a small known set of transactions with different dates and categories.
2. Calculate the expected result independently.
3. Request or open the report for the target month.
4. Compare total, transaction count, category breakdown, and net result.
5. Add a transaction on the first day of the next month and confirm it is excluded from the previous month.
6. Repeat with no transactions and with a single transaction.

If the frontend calculates the report from localStorage, its result may differ from the server's result. The team must identify which source is authoritative for the deployed feature and avoid combining two datasets accidentally.

## 7. Search, filters, and CSV export

Search and filters should narrow records without changing them. The backend supports expense filtering by category and month, and a CSV export route uses the same base query. Reusing a query builder is valuable because the list and export should apply the same ownership and filter rules.

CSV export has security and correctness concerns. It should export only the authenticated user's records and honour the chosen filters. Values should be escaped using a proper CSV writer so commas, quotes, or newlines do not break the file format. If user-controlled text is later opened in spreadsheet software, the team should consider spreadsheet formula injection for cells beginning with formula characters.

A CSV export is not the same as a database backup. It may omit internal relationships, settings, or other tables.

## 8. Consistency across the application

Suppose the dashboard says ₦20,000 was spent this month, but the transaction page totals ₦18,000. That inconsistency could come from different date filters, duplicated records, a missing category filter, stale browser state, or different data sources.

For each calculated value, the team should identify:
- the source records;
- the user and date filters;
- the formula;
- the point where the calculation runs;
- the expected rounding;
- how the result is tested.

The same rule should not be independently implemented in three places unless there is a clear reason. If the frontend needs an instant preview, it can calculate locally, but the server should remain authoritative for persisted data and protected summaries.

## 9. Business rules versus technical choices

A business rule describes what the application must mean. For example, “one budget per user, month, and category” is a business rule. PostgreSQL's unique index is a technical mechanism used to enforce it.

A category may be global or user-owned. That is a product/security decision, not something React or PostgreSQL can decide automatically. Once the team decides, the schema, API queries, UI, tests, and documentation must agree.

A tutor may intentionally ask an ambiguous question to see whether the team distinguishes the rule from the implementation. State the rule first, explain how it is enforced, and describe the evidence.

## 10. How business logic connects to the capstone

Business logic is the meaning of the application. It explains what counts as an expense, which category a budget applies to, how a monthly total is calculated, and what should happen when data is invalid or conflicts with existing records.

- **React:** collects values and presents totals, progress, and alerts. It can make a screen easier to use, but it must not be the only place where important rules are enforced.
- **Flask:** is responsible for validating values and applying server-side rules before records are saved or totals are returned.
- **PostgreSQL:** preserves critical relationships and uniqueness rules even when concurrent requests arrive or an API route has a bug.
- **Reports and dashboards:** must use consistent date boundaries, categories, ownership filters, and formulas. A dashboard total and a report total should agree when they cover the same records and period.
- **QA and CI:** use known test data with manually calculated expected totals. Test normal cases, empty months, boundary dates, duplicate budgets, deleted categories, and invalid amounts.
- **AWS deployment:** the demonstrated result must come from the actual deployed API and database. A feature in source code or a browser-only demo does not prove that it works on the server.

**Important accuracy rule:** the repository has both a root-level frontend and a `frontend/` frontend. The Compose file builds `./frontend`; the root README also describes localStorage-backed records. Verify the active UI and persistence path before claiming that every listed screen is connected to PostgreSQL. The guide teaches the intended rules, but only tested implementation evidence proves that a feature is complete.

## 10. Tough defense questions

**Q1. Why store a category ID instead of the category name in every expense?**  
The ID creates a stable relationship to one category record. If the label changes, linked expenses can still refer to the same category. A foreign key can also ensure that the referenced category exists.

**Q2. What is the difference between an expense date and a creation timestamp?**  
The expense date is when the spending happened; the creation timestamp is when the record entered the system. A user may record yesterday's expense today.

**Q3. How would you test the monthly total?**  
Use a small known dataset with records before, within, and after the month. Calculate the expected sum independently and verify the API/report uses the correct start-inclusive and next-month-exclusive boundaries.

**Q4. What should happen when a budget is exceeded?**  
The application should calculate eligible spending, compare it with the limit, and show an accurate over-budget state or alert. The alert is a consequence of the rule, not a replacement for calculating the total correctly.

**Q5. Why is a negative remaining budget not necessarily an error?**  
It may correctly mean that spending exceeded the limit. The UI should distinguish a valid over-budget result from invalid input or a failed calculation.

**Q6. Why should a budget linked to a category not become a whole-month budget when the category is deleted?**  
Because the budget's meaning changes without the user's consent. The deletion should be blocked or handled through an explicit documented reassignment/archive process.

**Q7. Does CSV export prove that all data can be restored?**  
No. CSV is a convenient export of selected records. It may omit relationships, account settings, or other tables and is not a complete backup/restore system.

**Q8. How do you know the dashboard and report are consistent?**  
Test both against the same known records, filters, and formulas. If they use different sources or different date boundaries, the numbers may diverge.

**Q9. The UI has an income page. Does that prove income is stored in PostgreSQL?**  
No. The screen could store data locally. Verify the actual network request, backend endpoint, database schema, and persistence after reload or login on another browser.

**Q10. Why not calculate every total only in React?**  
Client-side calculations are useful for display, but they can be modified and may use stale or incomplete data. Authoritative reports and protected financial data should be calculated from the correct server-side records or verified server responses.

**Q11. What is the difference between a category budget and a whole-month budget?**  
A category budget limits spending in one category. A whole-month budget limits spending across eligible categories. The query filters and uniqueness rules must reflect that difference.

**Q12. What is your first debugging step if a budget alert is wrong?**  
Reproduce it with known transactions, verify ownership/category/month filters, calculate the expected sum independently, inspect the backend response and persisted rows, then inspect the frontend's display logic.

## 11. Explain it in one minute

“The application turns individual expense records into useful information. Categories group records, budgets compare eligible spending with a limit, and reports aggregate transactions by month. Each calculation depends on clear business rules for ownership, category, date range, precision, and uniqueness. The database must preserve the relationships behind those calculations. We test features using small datasets with known expected answers, and we verify that the frontend and backend use the same data source. We do not claim a screen is server-persisted merely because it appears in React.”

---

# Session 6 — Testing, Quality Assurance, and Continuous Integration

## 1. Why testing is more than checking whether the app opens

A page loading proves only that some code ran. It does not prove that the amount is correct, one user cannot see another user's records, a database failure is handled safely, or the deployment is running the version the team tested.

Testing is a way to gather repeatable evidence that the application behaves as required. Quality assurance also examines whether the requirements are clear, risks are understood, and the team has evidence to support its claims.

A strong defense distinguishes source review, automated tests, manual testing, CI results, and live runtime verification. They answer different questions.

## 2. Testing levels

### Unit tests

A unit test checks a small unit of behaviour in isolation, such as validating an amount or calculating a monthly total. Unit tests are usually fast and make it easier to pinpoint the source of a failure.

### Integration tests

Integration tests check whether components work together. A backend integration test may call a Flask route using a test client and a test PostgreSQL database. It can verify the request, authentication, database operation, and response together.

### End-to-end tests

End-to-end (E2E) tests exercise a complete user journey through the interface and the services behind it. For example: register, log in, add an expense, view the expense, filter by month, export a CSV, and verify the result after reloading.

E2E tests are more realistic but can be slower and more sensitive to environment problems. They complement unit and integration tests; they do not replace them.

### Manual exploratory testing

A tester deliberately tries different scenarios, edge cases, and unexpected sequences. It is useful for discovering usability problems and interactions that were not anticipated by the scripted tests. Important findings should be recorded so the same behaviour can be reproduced.

## 3. What a test case contains

A useful test case states:
- **Precondition:** what must already be true;
- **Input/action:** what the tester does;
- **Expected result:** what should happen;
- **Actual result:** what happened in the test;
- **Evidence:** logs, screenshots, request/response, database state, or test output;
- **Environment/version:** which commit and environment were tested.

“Tested the expense page” is too vague. “With user A logged in, create a ₦250.50 expense dated 2026-10-10, reload the list, and verify the record is returned only for user A” is reproducible.

## 4. Important test scenarios for this project

### Authentication
- registration succeeds with valid input;
- duplicate email returns a controlled conflict;
- wrong password does not reveal whether the account exists;
- missing, invalid, and expired tokens are rejected;
- protected endpoints reject unauthenticated requests.

### Ownership
- user A creates an expense;
- user B attempts to read, update, delete, and export A's expense;
- user B must not receive or modify A's private data;
- summaries must be scoped to the correct user.

### Amount and date validation
- reject zero and negative amounts if the rule requires positive values;
- reject non-numeric and non-finite amounts;
- reject amounts with unsupported precision;
- reject impossible dates and invalid month filters;
- check month boundaries and leap-year cases where relevant.

### Budgets and categories
- reject duplicate budgets under the defined uniqueness rule;
- test both category-specific and whole-month budgets;
- confirm a budget calculation uses the right month and category;
- prevent category deletion from corrupting a linked budget;
- simulate a database conflict and confirm rollback/recovery.

### Export and reports
- exported rows match the selected filter;
- a user's export does not include another user's data;
- CSV quoting handles commas and quotes in descriptions;
- report totals match independently calculated expected values.

### Deployment
- Nginx serves the frontend;
- Nginx forwards API requests to Flask;
- public health checks return the expected status;
- database health reports a failure when PostgreSQL is unavailable;
- migration failure does not leave a falsely healthy or half-updated deployment.

## 5. Test data isolation

Tests must not accidentally use or delete production data. A test database should have a distinct database name and credentials, and the test setup should fail safely if it detects a production database.

Tests that create records need predictable cleanup or an isolated disposable database. Resetting Flask's testing flag or clearing a rate limiter is not the same as isolating the database. The test should be repeatable: running it twice should not cause duplicate records or depend on leftover data.

If the test suite uses PostgreSQL-specific constraints, testing only with SQLite can miss real behaviour. Use the same database engine for tests that depend on PostgreSQL semantics.

## 6. Continuous Integration (CI)

CI automatically runs checks when code is pushed or a pull request is opened. The project workflow includes checks such as component detection, secret scanning, database migration checks, backend lint/tests, frontend lint/build, Docker image builds, and an aggregate success check.

CI is valuable because it gives the team a repeatable minimum quality gate. It catches some problems before code is merged. But CI can only check what the workflow actually runs.

For example, the frontend package has lint and build scripts but no dedicated test script in the reviewed package configuration. A workflow step using `npm test --if-present` can skip frontend tests when there is no test script. The build can still pass. Therefore, do not interpret “frontend job passed” as proof that all frontend behaviours were tested.

Similarly, a backend workflow that treats “no tests collected” as an allowed exit code can pass without executing backend tests. In the current correction work, backend tests have been added and a reported CI checkpoint passed 35 backend tests, but each claim should be tied to the exact commit and workflow run rather than copied indefinitely into the future.

## 7. Lint, build, test, and deploy are different

- **Lint** checks source against configured style and static rules.
- **Build** transforms source into deployable artifacts.
- **Automated tests** execute defined behaviour and compare results with expectations.
- **Container build** checks whether an image can be built from the configured Dockerfile.
- **Deployment** places a version into an environment.
- **Health check** checks a defined signal from the running service.
- **E2E test** verifies an actual user journey across relevant components.

One green check does not imply the others passed. A successful Docker build does not prove the image runs. A health endpoint does not prove that expense creation works. A successful API test does not prove Nginx or AWS networking is configured correctly.

## 8. Regression testing

A regression is a previously working behaviour that breaks after a change. When a bug is found, the team should add a test that fails before the fix and passes after it. This prevents the same defect from returning.

For example, if deleting a category used to change a category-specific budget into a whole-month budget, add a regression test that creates the linked budget, attempts deletion, and verifies the budget remains unchanged. A test that only checks the error message is weaker than one that also verifies the database state.

## 9. Test evidence and defect closure

A finding should have a clear description, impact, reproduction steps, expected result, actual result, owner, correction, retest, and evidence. The team should distinguish:
- open issue;
- correction implemented;
- correction passed in CI;
- correction verified in a staging environment;
- correction verified in production.

Do not mark a live deployment issue closed merely because the code compiles. The fix may not be deployed, the migration may not have run, or the public route may still be wrong.

## 10. Tough defense questions

**Q1. What is the difference between unit and integration testing?**  
A unit test checks a small unit in isolation. An integration test checks how components work together, such as a Flask route and PostgreSQL.

**Q2. Why do we need E2E tests if backend tests pass?**  
Backend tests do not exercise the full browser, Nginx routing, deployed frontend, or the complete user interaction. E2E tests prove a broader path.

**Q3. Does a green CI run mean the application is production-ready?**  
Not by itself. It proves the configured jobs passed for a particular commit. The quality of the conclusion depends on what those jobs test and whether deployment/runtime checks have also passed.

**Q4. What does linting prove?**  
It proves that the code passed configured static/style checks. It does not prove business logic is correct or that the application works at runtime.

**Q5. Why should tests use PostgreSQL if production uses PostgreSQL?**  
Database engines differ in constraint behaviour, data types, and SQL semantics. Tests that rely on those behaviours should run against the same engine.

**Q6. Why is test database isolation important?**  
Without it, tests may read, change, or delete real data; they may also pass or fail depending on records left by previous runs. Isolation makes tests safer and repeatable.

**Q7. What is a regression test?**  
A test added to capture a known bug so the same behaviour cannot silently return in a later change.

**Q8. What evidence closes a deployment bug?**  
A passing test tied to the relevant commit plus runtime evidence from the target environment: deployment version, logs, health response, and the actual affected user journey.

**Q9. Why can a workflow pass when frontend tests are missing?**  
If the test command is optional and the package has no test script, the workflow can skip that step. A green job does not create tests that do not exist.

**Q10. How should QA report something it could not test?**  
State the exact test not performed, why it was not performed, what evidence is missing, the risk, and the steps needed to verify it. Do not label it passed or failed without the evidence.

## 11. Explain it in one minute

“Testing provides evidence at different levels. Unit tests check small rules, integration tests check components such as Flask and PostgreSQL together, and end-to-end tests verify complete user journeys. CI automates configured checks such as linting, migrations, tests, and container builds. A green workflow only proves the checks it actually runs for a specific commit. We therefore record test scope, environment, expected and actual results, and evidence. Live AWS deployment and browser behaviour require runtime verification beyond source review and CI.”

---

# Session 7 — Docker, Nginx, GitHub Actions, and AWS Deployment

## 1. What deployment means

Deployment makes a version of the application available in an environment where users can access it. It is not the same as writing code, committing code, building a container image, or passing CI.

A deployed system needs the right application version, runtime configuration, network routes, database, secrets, persistent storage, health checks, and a way to recover if an update fails.

The tutor has approved AWS as the deployment target. This guide explains the intended AWS deployment, but it does not claim that the current workflow is already configured correctly for AWS or that a live deployment has been independently verified.

## 2. Docker and containers

Docker packages an application and its runtime dependencies into an image. A container is a running instance of that image. Images help make builds repeatable and reduce the “it works on my laptop” problem.

A Dockerfile describes how to build one service image. Docker Compose describes how multiple services work together, including their environment, dependencies, ports, health checks, and volumes.

The repository's Compose configuration defines four main services:
- **frontend:** builds the React frontend;
- **backend:** runs Flask and reads runtime environment configuration;
- **postgres:** runs PostgreSQL and stores database files in a named volume;
- **nginx:** exposes the public HTTP port and routes requests to frontend or backend.

Containers on the Compose network can reach one another by service name. For example, the backend can use the Compose service name `postgres` as its database host. This is different from a browser on the user's laptop, which cannot automatically resolve an internal Compose service name.

## 3. Why PostgreSQL needs a persistent volume

The database container can be stopped or replaced during an update. If the database files exist only in the container's writable layer, replacing that container risks losing them. A named volume stores the database data separately from the container lifecycle.

A volume is not a complete backup. It protects data from ordinary container replacement, but not necessarily from accidental volume deletion, storage failure, corruption, or a bad migration. A real deployment needs backup and restore procedures.

The Compose file mounts the initial schema SQL into PostgreSQL's initialization directory. Those scripts are used when PostgreSQL initialises a new data directory. They do not automatically rerun on an already-initialised volume. New schema changes therefore need a deliberate migration command and verification.

## 4. Nginx as a reverse proxy

Nginx receives HTTP requests and forwards them to the correct internal service. In this project, the configuration has an upstream for the frontend and another for the backend. The root route serves the frontend, while API paths are forwarded to Flask. A dedicated `/api/health` route maps to the backend's `/health` endpoint.

Why put Nginx in front?
- The browser can use one public origin for the UI and API.
- Internal service ports do not need to be public.
- Nginx centralises routing and can be configured for TLS, compression, request limits, and security headers.
- The application can change internal container names without changing the public URL contract, provided Nginx configuration remains correct.

Nginx routing is not the same as application authorization. A request reaching Flask must still be authenticated and authorised by the backend where required.

## 5. Health checks: what they prove and what they do not

A health check is a small test that reports whether a service is responding according to a defined rule. The project includes backend health endpoints, a database health endpoint, and Compose health checks.

A basic `/health` response can prove that the Flask process responds. A database health check can run a simple query to prove that the application can reach PostgreSQL. A public `/api/health` check can additionally exercise the Nginx-to-backend path.

These checks do not prove that registration, expense creation, budgets, reports, and exports all work. A service can be healthy while a feature is broken. Use layered health checks plus user-journey tests.

The public route must be checked from outside the container network. Testing `http://backend:5000/health` inside Compose does not prove that a browser can reach the service through Nginx and the AWS security group.

## 6. GitHub Actions: CI versus CD

**Continuous Integration (CI)** runs checks on code changes: lint, tests, migration validation, secret scanning, and container builds. It helps catch defects before changes are accepted.

**Continuous Delivery/Deployment (CD)** publishes artifacts and may deploy them to a target environment. The workflow must select the exact version that passed CI, authenticate safely to the target, apply migrations in the correct order, restart services, and verify the running version.

A workflow file existing in Git is not proof that it has run successfully. A workflow run passing is not proof that the public deployment works. A successful SSH command is not proof that the correct application version is live.

## 7. AWS deployment concepts

For an EC2-based deployment, an EC2 instance is the virtual server that runs the Compose stack. The security group acts as a network firewall for inbound and outbound traffic. The deployment should expose only necessary ports.

A typical design would:
- allow public HTTP/HTTPS only as required;
- restrict SSH access to trusted addresses or use a managed access method;
- keep PostgreSQL private to the application network;
- store secrets outside the repository;
- run patched operating-system and container images;
- configure DNS and HTTPS certificates if a domain is used;
- keep database data on persistent storage with backups;
- record the exact deployed commit and image digests;
- provide a rollback procedure.

AWS IAM should grant the deployment identity only the permissions it needs. Do not use a broad administrator credential just because it makes the first deployment easier.

## 8. Critical source-to-runtime mismatch to understand

The reviewed `.github/workflows/cd.yml` contains cloud-host and secret-variable assumptions that must match the actual AWS deployment. It also describes pulling images and running Compose, so the team must verify that the exact tested image is deployed and that required database migrations succeed before application services restart.

The team has stated that AWS is the approved target. The workflow should therefore be updated to use the agreed AWS deployment method and secret names, or the team must document the separate deployment process actually used. Until that is done, do not tell a tutor that the repository's automated CD workflow deploys to AWS merely because the app was manually deployed there.

A robust workflow should use the exact commit tested by CI and preferably deploy immutable image tags or digests. A moving `latest` tag is convenient but can make it harder to prove which image is running. If the VM also pulls source code independently, the code and image can diverge unless both are pinned to the same commit.

## 9. A safe deployment sequence

A release should be performed in a controlled order:

1. CI checks the exact commit.
2. Build and publish immutable images from that commit.
3. Connect to the target AWS environment using protected credentials.
4. Confirm the deployment directory and configuration are correct.
5. Wait for PostgreSQL to become ready.
6. Apply required migrations in order and stop if one fails.
7. Pull or select the exact image version that passed CI.
8. Restart services in the intended order.
9. Wait for container health checks.
10. Call the public health endpoint through Nginx.
11. Run a small safe smoke test or browser journey.
12. Record the commit, image digests, deployment logs, health response, and result.
13. If verification fails, follow the rollback plan rather than leaving an uncertain release live.

A migration can be difficult to roll back if it has modified or deleted data. Back up before risky schema changes and design migrations to be safe for the version transition.

## 10. Evidence to show for AWS

A professional demonstration can include:
- public application URL;
- AWS EC2 instance state and region, without exposing credentials;
- security-group rules showing only intended public ports;
- deployed commit SHA or immutable image digest;
- deployment workflow run or deployment log;
- public health response through Nginx;
- database health result;
- screenshot or recording of a successful user journey;
- proof that data remains after refresh or a controlled restart;
- backup/restore or rollback evidence if required by the capstone.

Do not share SSH private keys, passwords, access tokens, private IP information that is not needed, or the contents of a production `.env` file.

## 11. Tough defense questions

**Q1. Why use Docker?**  
Docker packages the application and its runtime dependencies consistently. It helps make local, CI, and server environments more repeatable, but it does not automatically solve security or data persistence.

**Q2. What is the difference between an image and a container?**  
An image is the packaged template. A container is a running instance created from that image.

**Q3. Why is PostgreSQL a separate service?**  
It has a different responsibility and lifecycle from the API. Separating it allows independent configuration and health checks, and a persistent volume protects the data across ordinary container replacement.

**Q4. What does Nginx do here?**  
It receives public requests and routes the frontend and API paths to their internal services. It can also support TLS and other HTTP controls when configured. It does not replace backend authorization.

**Q5. Why can a service be healthy while a feature is broken?**  
A health endpoint may only prove the process or database responds. It does not exercise every business rule or user journey.

**Q6. Why must migrations run before the new application version?**  
The new code may expect a schema change. If the database is not updated, requests can fail. The workflow should stop if migration fails and should not blindly restart into an incompatible state.

**Q7. Does passing CI prove the AWS deployment works?**  
No. CI runs in its own environment. AWS has its own networking, secrets, existing data, image versions, and runtime conditions. The live system needs separate checks.

**Q8. Why must the deployment workflow be checked if AWS is the approved target?**  
Because workflow configuration must match the real server, authentication method, image registry, secrets, and deployment commands. A workflow that merely has a successful CI build does not prove that it can deploy to AWS.

**Q9. What is the risk of deploying `latest`?**  
It can point to different image content over time, making rollbacks and evidence less precise. Immutable tags or digests make it easier to identify the exact artifact that was tested.

**Q10. What would you do if a migration failed during deployment?**  
Stop the release, preserve logs, avoid restarting into an incompatible schema, inspect the migration's partial effects, and follow a tested recovery plan. Restore from backup if necessary. Do not rerun blindly without understanding whether the migration is safe to repeat.

**Q11. Is a persistent volume the same as a backup?**  
No. A volume protects against some container lifecycle events; a backup is a separate recovery mechanism and must be tested.

**Q12. How would you prove the public app is the version that passed CI?**  
Record the tested commit, build images from that commit, deploy immutable image tags/digests, and expose or record the deployed version. Then compare the live deployment record to the CI run.

## 12. Explain it in one minute

“Docker packages the frontend and backend and Compose connects them to PostgreSQL and Nginx. Nginx routes public requests, PostgreSQL uses a persistent volume, and health checks provide limited signals about service availability. GitHub Actions CI checks code and builds; CD publishes and deploys a specific version. Our agreed cloud target is AWS, so the deployment workflow must be checked against the actual AWS server, secrets, image version, and deployment commands before we claim automated AWS deployment. We prove a release with the exact commit or image digest, migration logs, public health checks, and a real user journey.”

---

# Session 8 — System Integration, Capstone Alignment, and Difficult Defense Questions

## 1. The whole application in one picture

Think of the Expense Tracker as a chain of responsibilities:

`User → Browser/React → HTTP request → Nginx → Flask API → validation/business rules → SQLAlchemy → PostgreSQL → API response → React updates the screen`

For deployment, add another chain around it:

`Git commit → GitHub Actions checks → build container images → deploy exact version to AWS → run migrations → health checks → browser verification`

Every arrow is a possible failure point. A React page may be available while the API is down. The API may be healthy while PostgreSQL is unreachable. The database may contain correct records while the frontend shows stale localStorage data. CI may pass while the AWS deployment uses an older image. A good defense explains how the team verifies the whole chain rather than treating the technologies as a list.

## 2. What “full stack” actually means

A full-stack application has connected responsibilities across the client, server, and data store. Having a React folder, Flask folder, and PostgreSQL service in the same repository is necessary for this architecture but not sufficient to prove every feature is connected.

For each user-facing feature, trace:
1. Which React component handles the action?
2. What state or form values does it read?
3. Does it call an API or use localStorage?
4. Which HTTP method and endpoint does it use?
5. Does the backend route validate the input?
6. Does it identify the authenticated user?
7. What database query or transaction runs?
8. What response status and data return?
9. How does the UI handle success and failure?
10. What test proves the whole path works?

Use this route map for the demo. If any step is not implemented, record it as a gap rather than inventing the missing link.

## 3. Responsibilities by layer

### Frontend — React
Owns the interface, user interaction, local presentation state, form usability, loading states, and readable error messages. It should not be the final authority on permission or database integrity.

### Reverse proxy — Nginx
Routes incoming requests to the appropriate internal service. It may handle TLS and related HTTP controls when configured. It does not decide whether a user owns an expense.

### Backend — Flask
Owns API behaviour, server-side validation, authentication checks, authorization, business rules, database operations, and consistent responses.

### ORM — SQLAlchemy
Maps Python model objects to relational tables and helps manage queries and transactions.

### Database — PostgreSQL
Persists records and enforces constraints such as primary keys, foreign keys, and uniqueness.

### Container layer — Docker/Compose
Packages and connects services and defines runtime configuration, health checks, and persistent volumes.

### Automation — GitHub Actions
Runs configured quality gates and may publish/deploy artifacts. The workflow's actual configuration and result must be checked; its presence in the repository is not proof that a release happened.

### Cloud — AWS
Provides the compute and networking environment for the deployed system. The application still needs secure configuration, monitoring, backups, updates, and a verified release process.

## 4. How to align the capstone requirement with implementation

The project brief as relayed by the team calls for a full-stack application using React, Flask, PostgreSQL, Docker Compose, GitHub Actions, Nginx, a cloud VM, a health endpoint, and documentation/evidence such as a README, live URL, and screenshots. The tutor has approved AWS as the alternative cloud target, so the deployment explanation in the defense should use AWS consistently.

Use the original capstone handout as the final authority for exact wording. The mapping below is a practical evidence map, not a substitute for that handout.

| Expected area | What it means | Evidence to prepare | Common false claim to avoid |
|---|---|---|---|
| React frontend | Usable interface and feature flows | Screenshots plus browser demonstration | “A build proves all buttons work.” |
| Flask backend | Routes, validation, auth, business logic | Route tests and request/response evidence | “The frontend is the security boundary.” |
| PostgreSQL | Persistent relational data | Schema, migrations, constraint tests | “A SQL file existing means production ran it.” |
| Docker Compose | Services work together | Compose config, container status, logs | “Building an image proves the service is running.” |
| GitHub Actions | Repeatable automated checks | Run URL, commit SHA, job results | “CI passed, so live AWS is proven.” |
| Nginx | Public request routing | Public route checks and config | “A route works inside Docker, so it works publicly.” |
| AWS VM | App deployed to the agreed cloud target | Public URL, instance details, deployed version | “AWS is approved, so the current workflow already deploys there.” |
| Health endpoint | Service responds through the intended route | HTTP status/body from public endpoint | “Health means every feature works.” |
| README and documentation | Reproducible setup and clear architecture | Updated setup, environment variables, architecture, limitations | “A setup guide can omit required secrets and migration steps.” |
| Screenshots and live demo | Evidence that a real scenario works | Dated screenshots, recording, test account | “A screenshot alone proves database persistence.” |

## Repository consistency risk: duplicate frontend source trees

The repository has both root-level `src/` and `frontend/src/`. The root README describes localStorage-backed application records, but the Compose frontend is built from `./frontend`, whose main app loads data through the Flask API. These are not interchangeable implementations. A component may exist in the root tree but never be included in the Docker image used for the AWS deployment.

Before the defense, the team should confirm the actual build context and deployed image, update the README to describe that version, and either remove the unused source tree in a separate reviewed change or explain why both trees are retained. Do not claim both implementations are active unless the build and deployment prove that.

**Evidence to prepare:** the relevant Compose `build:` setting, the frontend entry point included in the image, the workflow's build context, the deployed image tag/digest or commit, and a browser demonstration of a transaction saved through the Flask API and retrieved after reload/login.

## Repository consistency risk: duplicate frontend source trees

The repository has both root-level `src/` and `frontend/src/`. The root README describes localStorage-backed application records, but the Compose frontend is built from `./frontend`, whose main app loads data through the Flask API. These are not interchangeable implementations. A component may exist in the root tree but never be included in the Docker image used for the AWS deployment.

Before the defense, the team should confirm the actual build context and deployed image, update the README to describe that version, and either remove the unused source tree in a separate reviewed change or explain why both trees are retained. Do not claim both implementations are active unless the build and deployment prove that.

**Evidence to prepare:** the relevant Compose `build:` setting, the frontend entry point included in the image, the workflow's build context, the deployed image tag/digest or commit, and a browser demonstration of a transaction saved through the Flask API and retrieved after reload/login.

## 5. The most important integration risk in this repository

The frontend documentation describes browser `localStorage` as a persistence layer, while the backend includes PostgreSQL-backed routes. The backend route registration reviewed includes auth, categories, expenses, budgets, and summaries, but does not show a dedicated income route. This means the team must verify feature-by-feature whether the visible React screens call the server API or operate on local browser data.

This issue should be discussed constructively and precisely. Do not say “the app is fake” or assume all functionality is broken. Say: “The repository contains both a frontend and backend, but we need end-to-end evidence that each displayed feature is wired to the intended API and persistent database. We are mapping each screen to its endpoint and test so our claims match the implementation.”

## 6. The most important deployment alignment risk

The CD workflow must be consistent with the AWS deployment method the team actually uses. The team should verify the host, secrets, image version, commands, and recovery path, or clearly document the verified manual deployment procedure.

Until that is done, separate three statements:
- **Approved target:** AWS has been accepted by the tutor, according to the team.
- **Configured pipeline:** the source workflow must be checked to determine whether it deploys to AWS.
- **Verified live deployment:** the deployed URL, version, health response, and user journey must be tested.

Those statements are not interchangeable. This distinction protects the team's credibility during the defense.

## 7. A defensible live demonstration sequence

Prepare a dedicated test account and non-sensitive sample data. Avoid using real financial information or showing real credentials.

1. Open the public AWS application URL over HTTPS if configured.
2. Record the deployed version or commit associated with the demonstration.
3. Register or log in using a test account.
4. Create a category if the product policy permits it.
5. Add an expense with a known amount and date.
6. Show the new expense in the list.
7. Refresh the page and verify whether the record persists from the intended source.
8. Filter by month or category.
9. Show a budget calculation using a known expected total.
10. Open a report and compare its total with the sample records.
11. Export CSV and inspect the exported rows.
12. If a second test account is available, show that it cannot access the first account's private expense.
13. Open the public health endpoint and explain exactly what it verifies.
14. Show the relevant CI run and deployed commit.
15. Explain known limitations without pretending they have passed tests.

If a step cannot be demonstrated, say what is missing and what evidence would be needed. A clear limitation is more credible than a false claim.

## 8. Tough cross-system questions

**Q1. Trace an expense from the user's click to PostgreSQL.**  
A strong answer names the React form and handler, the HTTP method and endpoint, the Nginx route, the Flask route, server validation, the authenticated user identity, the SQLAlchemy operation, the transaction commit, the response, and the frontend's handling of that response.

**Q2. What happens if Nginx is running but Flask is down?**  
Nginx may still serve static frontend files, but API requests will fail or return a proxy error. Health checks and browser tests should detect this. A running Nginx container alone does not prove the application is fully available.

**Q3. What happens if Flask is running but PostgreSQL is down?**  
Simple process health may still succeed, but database-dependent operations will fail. A database health endpoint and tests for dependency failure provide a clearer signal.

**Q4. What happens if the database saves the expense but the frontend request times out?**  
The user may retry and accidentally create a duplicate. The UI should handle uncertain outcomes carefully, and the backend may need an idempotency strategy for operations where duplicate submissions are a serious risk. At minimum, the system should not blindly claim success or failure without checking the outcome.

**Q5. Why do we need both API tests and browser tests?**  
API tests can precisely verify backend rules. Browser tests prove the visible interface calls the API correctly and handles the result. Neither alone proves the whole system.

**Q6. How can the team prove that a report total is correct?**  
Use a small dataset with known amounts, dates, users, and categories. Independently calculate the expected sum, call the report, and compare totals and included records. Include boundary dates and empty results.

**Q7. Why is a database constraint stronger than checking in Python?**  
A route-level check can race with another request or be bypassed by another code path. A database constraint is enforced where the data is stored and protects concurrent operations.

**Q8. How do you know the live app is the version that passed CI?**  
Build from the tested commit, use immutable image tags/digests, record the deployment version, and compare the running version with the CI run. A moving `latest` tag alone is weak evidence.

**Q9. What if migration 004 works on a fresh database but fails on AWS?**  
The AWS database may contain data that violates the new constraint or may be at a different schema version. Inspect existing references, back up, test against a copy of production-like data, and define a safe remediation before applying it.

**Q10. Why not expose PostgreSQL directly so the frontend can connect to it?**  
The browser should not have database credentials or direct database access. The backend provides controlled validation and authorization and keeps database credentials private.

**Q11. Does an access token mean every API request is safe?**  
No. The token must be valid, the route must enforce authentication, the resource must be scoped to the user, inputs must be validated, and the database must enforce integrity.

**Q12. How do we know the workflow deploys to the approved AWS environment?**  
The source configuration has not yet been fully aligned with the team's updated deployment decision. The correction is to update the pipeline and secrets for the actual AWS method and then prove it with a successful run and live verification. Approval of a target is not the same as implementation of its deployment pipeline.

**Q13. How do you respond when you do not know whether a feature is fully integrated?**  
Do not guess. Explain the intended flow, state the evidence already checked, identify the missing link, and describe the test needed to close the gap. Defending software means accurately describing its behaviour, including limitations.

**Q14. What is the difference between availability and correctness?**  
Availability means a service can be reached and responds. Correctness means it produces the right result under the specified rules. A service can be available but return incorrect totals or expose another user's data.

**Q15. What is the single biggest proof that the system works?**  
No single screenshot or CI badge proves everything. The strongest evidence is a traceable set: tested commit, automated test results, correct database constraints, successful deployment, public health checks, and a repeatable browser journey that demonstrates the required features.

## 9. How to answer questions without memorising scripts

Use the **definition → reason → project example → proof → limitation** pattern.

Example question: “Why do you use Nginx?”

1. **Definition:** Nginx is the reverse proxy in front of the frontend and backend.
2. **Reason:** It routes incoming paths to the correct internal service and can centralise HTTP/TLS controls.
3. **Project example:** The root path goes to the frontend, and API paths go to Flask.
4. **Proof:** Show the Nginx configuration and test the public route through it.
5. **Limitation:** Nginx does not replace backend authentication and the current source must be checked for HTTPS and deployment configuration.

This method works for almost any question because it shows understanding rather than memorisation.

## 10. Final defense checklist

Before the defense, the team should be able to show or explain:

- [ ] The role of React, Flask, SQLAlchemy, PostgreSQL, Docker Compose, Nginx, GitHub Actions, and AWS.
- [ ] One complete expense journey from the browser to the database and back.
- [ ] Which features use backend persistence and which, if any, still use browser storage.
- [ ] The exact authentication and ownership checks for personal records.
- [ ] How amount, date, category, budget, and report rules are enforced.
- [ ] How duplicate records and failed transactions are handled.
- [ ] How database migrations are applied to both new and existing databases.
- [ ] Which automated tests ran for the exact commit being demonstrated.
- [ ] Which checks are absent or optional, including frontend functional tests if still missing.
- [ ] The actual AWS deployment method and whether the CD workflow matches it.
- [ ] The deployed version, public health response, and browser-test evidence.
- [ ] A clear list of limitations and the next steps to close them.

## 11. Final one-minute project explanation

“The Expense Tracker is a web application designed to organise expenses, categories, budgets, and reports. React provides the user interface; Flask provides the API and server-side rules; SQLAlchemy connects the backend to PostgreSQL; Docker Compose runs the services together; Nginx routes public requests; and GitHub Actions automates defined checks and build steps. AWS is the agreed cloud target. We judge the system by evidence across the whole path: code and API tests, database integrity, the exact deployed version, public health checks, and a repeatable browser journey. We also distinguish completed functionality from work that still needs integration or runtime verification.”
