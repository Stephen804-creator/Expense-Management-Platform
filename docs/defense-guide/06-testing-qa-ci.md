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
