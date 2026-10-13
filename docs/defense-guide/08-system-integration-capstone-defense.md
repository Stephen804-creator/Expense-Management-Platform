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
