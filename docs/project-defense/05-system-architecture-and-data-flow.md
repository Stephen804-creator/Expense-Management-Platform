# Session 5 — System Architecture and End-to-End Data Flow

## 1. What is system architecture?

System architecture is the arrangement of the major parts of an application and the rules governing how they communicate. It explains not just which technologies were chosen, but why each component exists, where responsibilities belong, and how a user's request moves through the system.

The capstone architecture is a web application with a React frontend, a Flask REST API, PostgreSQL storage, Docker Compose for container orchestration, and Nginx as a reverse proxy. The deployed cloud VM target is AWS EC2, based on the team's statement that the tutor accepted AWS as an alternative.

The intended high-level path is:

User's browser → public AWS endpoint → Nginx → React static frontend or Flask API → PostgreSQL

The browser should not connect directly to PostgreSQL. The database should not need a public internet endpoint for normal application traffic. The backend is the controlled layer between user requests and persistent data.

## 2. What each component is responsible for

**React frontend:** presents screens, accepts user actions, manages temporary UI state, and sends API requests.

**Flask API:** authenticates requests, validates input, enforces business rules and permissions, reads/writes data, and returns HTTP responses.

**PostgreSQL:** stores durable records and enforces constraints such as keys, relationships, positive amounts, and uniqueness rules.

**Nginx:** accepts public HTTP traffic, serves or routes frontend requests, and forwards the public API path to Flask.

**Docker:** packages each service and its runtime dependencies into an image.

**Docker Compose:** defines the services, networks, environment, health checks, dependencies, ports, and persistent volumes needed to run the application together.

**GitHub Actions CI:** automatically checks source changes, runs lint/tests, validates database migrations, scans for secrets, builds frontend assets, and builds container images according to the workflow configuration.

**CD workflow and AWS EC2:** publish images and, when deployment is enabled and correctly configured, deploy a selected version to the VM and check whether the service responds.

These are different responsibilities. A container is not the same thing as a database. Docker Compose is not the same thing as CI/CD. Nginx is not the application backend. AWS EC2 is a virtual server, not a replacement for React, Flask, or PostgreSQL.

## 3. Why separate the frontend, backend, and database?

Separation creates boundaries.

The frontend can change its layout without rewriting the database. The backend can change its internal implementation while keeping the same API contract. The database can enforce integrity regardless of which page sends a request.

The separation also supports security. Browser code is delivered to the user and must be considered inspectable and modifiable. Therefore database credentials belong on the server side, not in React source or a frontend build. Business rules such as record ownership must be enforced in Flask, not merely by hiding a button.

The cost of separation is that components can disagree. The frontend may send a field called date while the backend expects expense_date. Nginx may route a path incorrectly. The backend may expect a database environment variable that Compose never supplies. Architecture is successful only when the contracts and configuration between components are tested.

## 4. The architecture diagram: what it should show

The capstone asks for an architecture diagram that includes users, internet/network, cloud infrastructure, application components, database/storage, and the CI/CD pipeline.

A useful diagram should show:

1. A user browser.
2. Internet traffic reaching the AWS EC2 public endpoint.
3. Nginx as the public entry point.
4. A frontend container serving the React build.
5. Nginx forwarding /api requests to the Flask container.
6. The Flask container connecting to PostgreSQL over an internal Docker network.
7. A named PostgreSQL volume storing persistent data.
8. GitHub repository and pull requests.
9. GitHub Actions CI checks.
10. Image build/publish and the CD/deployment connection to AWS.
11. Health-check and log paths.
12. The security boundary: only intended ports are public; database traffic remains internal.

Do not draw an architecture that claims a service exists if it has not been deployed. If a component is planned rather than implemented, label it as planned. If the diagram says HTTPS but the live endpoint uses only HTTP, the diagram and reality disagree.

## 5. A full user journey: adding an expense

A strong defence explains the whole journey rather than listing technologies.

1. The user enters an expense in React.
2. The form checks obvious input mistakes.
3. The shared API service sends a POST request to the public /api/expenses path and attaches the bearer token.
4. Nginx matches the /api path and forwards the request to Flask.
5. Flask verifies the token, derives the user's identity, validates amount/date/category, and creates the model object.
6. SQLAlchemy submits the insert to PostgreSQL.
7. PostgreSQL enforces required fields, foreign keys, and the positive-amount rule.
8. If the transaction succeeds, Flask returns a creation response.
9. React receives the response and reloads or updates the relevant state.
10. The transaction list shows the record; summary and budget views should now reflect it.
11. If a later GET request is made after page reload, the record should still appear because it was stored in PostgreSQL.

Every arrow is a potential failure boundary. The browser may have no network; Nginx may route to an unavailable backend; the token may be expired; validation may fail; PostgreSQL may reject the insert; or the response may be lost after the server commits. The UI must not treat all of these outcomes as success.

## 6. What happens when something fails?

### Invalid input

Flask returns a 400 response with a safe message. The frontend displays the message near the form or in a toast. No invalid record should be committed.

### Missing or expired token

The backend returns 401. The frontend clears or invalidates its authenticated state and requests login again. It should not keep acting as if the user is authenticated.

### Duplicate budget

The database's unique indexes are the final protection against duplicate budgets under concurrency. The API should identify the known uniqueness conflict, roll back, and return 409. A random database error should not be mislabeled as a duplicate.

### Database unavailable

The backend cannot complete operations that require PostgreSQL. Its health endpoint should report the correct state, logs should help diagnose the failure, and the frontend should not show false success. A simple /health endpoint that only proves the Flask process responds is not the same as a database readiness check.

### Nginx path mismatch

If the frontend requests /api/health/db but Nginx only defines /api/health and a general /api/ route, the resulting path may not map to the backend endpoint the way the team expects. Test every documented health path through the public reverse proxy, not only by calling Flask directly inside the container.

### Deployment failure

A new container image may fail to start or may be unhealthy. A safe deployment process should check prerequisites, preserve the previous known-good version where possible, stop if migrations fail, and have a documented rollback path. Do not claim automatic rollback unless the workflow actually implements and verifies it.

## 7. API contracts connect independent teams

The frontend and backend can be developed separately only when they agree on the API contract.

The contract should define:

- HTTP method and path.
- Authentication requirements.
- Required and optional fields.
- Data types and date/amount formats.
- Success status and response structure.
- Error statuses and response structure.
- Filtering and pagination behaviour, if supported.
- Ownership and permission rules.
- Whether a successful mutation returns the created record or requires a follow-up GET.

For example, if the frontend sends category_id as a string while the backend expects an integer, the backend should validate or safely convert it and return a clear error if invalid. The two teams should not rely on assumptions that are absent from documentation or tests.

## 8. Source of truth and derived views

PostgreSQL is the durable source for expense-related records. React state is a temporary representation. Charts, budget progress, and monthly summaries are derived views.

This distinction matters when values disagree. Trace a displayed total from the UI to the API response, backend query, database records, and any transformation. Decide which layer owns the calculation and test it. If a stored summary table is used, the team must explain how it stays consistent with the expense rows. A table existing does not prove it is populated or kept accurate.

## 9. Stateless API and scaling

A REST API is often designed so each request carries the information needed to authenticate and process it, instead of relying on hidden server-side page state. JWTs can support this style, although revocation and token storage still need a plan.

Containers make services easier to package and recreate consistently, but they do not automatically make the application scalable. If multiple backend instances are added, they need consistent database configuration and a shared strategy for any stateful components such as rate-limit counters. In-memory rate limiting can behave differently on each instance. Database connection counts, resource limits, logs, and deployment orchestration also matter.

For this capstone, explain the architecture that exists rather than claiming that multi-instance autoscaling is implemented unless it is actually configured and demonstrated.

## 10. Why the same-origin reverse-proxy pattern is useful

With Nginx, the browser can load the frontend from the public host and call /api on the same origin. Nginx sends frontend paths to the frontend service and API paths to Flask.

This can simplify the browser's API URL and avoid unnecessary cross-origin complexity. It also centralises the public entry point. However, it does not mean CORS, CSRF, or security questions automatically disappear; the correct protections depend on the actual authentication and origin design.

The browser should not use internal service names such as backend or postgres. Those names are for container-network communication. A user's browser cannot resolve them in the same way.

## 11. Connections to testing and operations

An architecture diagram explains intended paths. Integration tests verify that components actually communicate as intended. End-to-end tests verify user journeys across multiple components. Health checks provide a narrow signal that a process or dependency is responding. Logs help explain failures after they occur.

The CI pipeline checks a code revision before deployment. The deployment process must then ensure the deployed revision is the one that passed CI. If an image tagged latest changes over time, deployment records should still identify the exact commit or image digest that is running.

## 12. Difficult defence questions and answers

**Q1. Why is the frontend not allowed to connect directly to PostgreSQL?**  
The browser is an untrusted environment. Direct database access would expose credentials and bypass backend validation and authorisation. The API is the controlled boundary.

**Q2. What is the difference between Docker and Docker Compose?**  
Docker builds and runs containers from images. Docker Compose describes and coordinates multiple services, their networks, environment, volumes, health checks, and startup dependencies.

**Q3. What is the difference between Nginx and Flask?**  
Nginx is the reverse proxy/public web server that routes traffic. Flask implements the application API and business rules. Nginx forwarding a request does not mean it has validated the expense.

**Q4. What is the difference between CI and CD?**  
CI automatically checks source changes through tasks such as linting, testing, scanning, and builds. CD automates publishing and deploying a tested version. A passing CI run does not by itself prove the live service is updated.

**Q5. What is the difference between a health check and an end-to-end test?**  
A health check is a narrow signal that a process or dependency responds. An end-to-end test exercises a user journey across multiple layers. A healthy Flask process may still have a broken registration or CSV export flow.

**Q6. Why does the database use a private container network?**  
The frontend and public internet do not need direct access to the database. Keeping PostgreSQL internal reduces the exposed attack surface and lets only authorised application services connect.

**Q7. What happens if PostgreSQL is healthy but Flask is not?**  
The database can be running while the API is unavailable. The reverse proxy or frontend may fail to reach the backend. Each service needs its own health and logs, and the overall user journey must be tested.

**Q8. Why should a deployment identify an exact commit?**  
Without a commit or immutable image identifier, the team may not know which code is running. The version deployed should correspond to the version that passed CI, and rollback should identify the previous known-good version.

**Q9. Why does an architecture diagram need a CI/CD pipeline?**  
The capstone evaluates how source code becomes a running application, not only the application components. The diagram should show source control, automated checks, image publication, deployment to AWS, and post-deployment health verification.

**Q10. If the public health route returns 200, is the whole app working?**  
Not necessarily. The route may only prove that one process responds. Test database readiness separately and run browser journeys such as login, creating an expense, reading it back, and exporting data.

**Q11. Can we say the application is scalable because it uses containers?**  
No. Containers improve packaging and repeatability. Scaling also requires instance orchestration, capacity planning, shared state, connection limits, monitoring, and verified load behaviour.

**Q12. How do you diagnose a request that works inside Flask but fails through Nginx?**  
Check the public URL, Nginx location matching and proxy_pass behaviour, service DNS/network, container health, and response logs. Test through the same public route that the browser uses.

## 13. Architecture defence checklist

Draw the request path without notes. Explain every component's responsibility and one failure it can experience. Demonstrate a complete expense journey and a failure case. Make sure the diagram matches the actual AWS deployment, public ports, container names, API paths, and currently deployed code.
