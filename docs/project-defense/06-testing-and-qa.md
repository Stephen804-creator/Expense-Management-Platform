# Session 6 — Testing, Quality Assurance, and Evidence

## 1. What is software testing?

Software testing checks whether an application behaves as expected under defined conditions. It is not simply opening the website and seeing that the homepage appears. Expense Tracker contains authentication, data entry, ownership rules, calculations, filtering, budgets, exports, containers, and deployment configuration. A defect in any one of these can affect the user or expose data.

A **test case** states the preconditions, actions, and expected result. A **test result** records what actually happened. A **defect** is a difference between expected and actual behaviour. A **regression test** is a test added or reused to ensure a previously fixed defect does not return.

A good test should be repeatable. If it passes only when the tester clicks in a particular order or uses a certain personal browser state, it may not be reliable evidence.

## 2. The levels of testing

### Unit tests

A unit test checks a small unit of logic in isolation, such as a date parser, amount validator, or helper function. It is fast and helps pinpoint the source of a failure. It does not prove that Nginx, Flask, PostgreSQL, and the browser work together.

### API or component tests

API tests call Flask routes and inspect status codes, response bodies, authentication behaviour, and database effects. Frontend component tests can verify rendering and interactions in isolation. They test more behaviour than a pure helper test but still may not include the full deployed environment.

### Integration tests

Integration tests verify that components work together—for example, a Flask route writing to PostgreSQL, or Nginx forwarding a public /api request to Flask. They catch contract mismatches between components.

### End-to-end tests

End-to-end (E2E) tests follow a user journey through the actual browser interface and the connected services. Example: register, log in, create an expense, reload the page, verify the expense appears, filter it by month, and export the matching CSV.

E2E tests are slower and can fail for environment reasons, so they do not replace unit/API tests. They prove something different: the full user journey works under the tested conditions.

### Operational tests

Operational verification checks deployment-specific behaviour such as container health, public health routes, logs, migrations on an existing database, TLS, and rollback. A unit test cannot prove the AWS VM has the expected image or security-group configuration.

## 3. The test pyramid

A sensible strategy usually has many small, fast tests; a smaller number of API/integration tests; and a focused set of complete browser journeys. Operational checks are added for the deployed environment.

Why not test only through the browser? A failed E2E test can have many possible causes, and debugging is slower. Unit and API tests help isolate logic. Why not test only the backend? That misses frontend wiring, Nginx paths, browser storage, and UI feedback. Different layers catch different classes of defect.

## 4. What must be tested in Expense Tracker?

### Authentication

- Valid registration creates a user and stores a password hash, not a plaintext password.
- Correct credentials can log in.
- Incorrect credentials are rejected without revealing which sensitive part was wrong.
- Missing, invalid, and expired tokens are rejected by protected endpoints.
- The UI returns to an appropriate unauthenticated state after a 401 response.

### Expense management

- An authenticated user can create a valid expense.
- Missing, malformed, zero, negative, or overly precise amounts are rejected as defined.
- Invalid dates and unknown categories are rejected.
- A successful create persists after page reload.
- Update/delete operations enforce the intended ownership policy.
- One user's expenses and exports cannot expose another user's data.

### Categories

- Create, read, update, and delete operations follow the documented category policy.
- Duplicate category names return a clear conflict response if names are unique.
- A category with linked expenses cannot be deleted if the database policy restricts it.
- A category with linked budgets must not silently change the meaning of those budgets.

### Budgets

- A budget can be created for the intended month and category.
- The month is stored and interpreted consistently.
- Duplicate category budgets for the same user/month/category are rejected.
- Duplicate whole-month budgets are rejected if only one is allowed.
- Simultaneous requests cannot bypass uniqueness.
- A budget's spent value agrees with qualifying expenses.

### Summaries and filters

- Month boundaries are tested: first day included, first day of the next month excluded.
- Category filters include only the selected category.
- Summary totals are compared with manually calculated expected values.
- Empty data, one record, multiple categories, and month changes are covered.
- Invalid filter parameters return clear errors rather than silently broadening the query.

### CSV export

- The exported columns and date/amount formats are correct.
- The export contains only the authenticated user's records.
- Selected date/category filters are honoured.
- Empty exports behave sensibly.
- CSV formula-injection risk is assessed for user-controlled text.
- The response has appropriate content type and download behaviour.

### Deployment

- CI runs for the relevant branch/commit.
- Database migrations apply in order on a fresh database.
- Migrations are tested against a populated database before production use.
- Docker images build.
- Containers become healthy.
- Nginx forwards the public API paths correctly.
- Public health endpoints report the expected state.
- The running AWS version is tied to the tested commit or image digest.
- A documented recovery/rollback procedure is tested where possible.

## 5. Test data and isolation

Tests should use a dedicated test database, never a production database. Test fixtures should create a known starting state and clean up reliably. If tests share a database without isolation, one test can leave a user or category that changes the next test's result.

Test data isolation matters especially for authentication and ownership tests. Create at least two test users and ensure the second cannot read or export the first user's records. The test must prove the security boundary, not merely prove that each user can read their own data.

A test database should have a clear name and safety guard. The test setup should refuse to run destructive cleanup if its configured URL points to a production database. Where feasible, transactions can be rolled back after a test; some operations or separate connections require a different cleanup strategy.

Do not let tests pass by skipping the assertions that matter. If a test runner returns a special status because no tests were collected, treating that as a successful job may be useful during an initial scaffold phase, but it is not evidence that functionality is tested. Once tests are a capstone requirement, absence of tests should be visible as a gap.

## 6. Assertions and expected results

An assertion compares the actual outcome with the expected one. A weak test might only check that an endpoint returns a response. A stronger test checks the exact status code, relevant response fields, and database side effect.

For example, after creating an expense, a useful test checks the 201 response, retrieves the expense through the authenticated API, verifies the amount/category/date, and confirms another user cannot retrieve it. A failed update should leave the original value unchanged.

Tests should include both positive and negative cases:

- Positive: valid request succeeds.
- Negative: invalid request is rejected.
- Boundary: values near limits are handled correctly.
- Security: unauthorised access is denied.
- Failure recovery: after a database error and rollback, a later valid request still succeeds.
- Concurrency: simultaneous requests do not violate uniqueness.
- Regression: the originally reported defect stays fixed.

## 7. CI is evidence, but only for its exact scope

Continuous Integration runs automated checks on a specific source revision. In this project, the workflow can include secret scanning, component detection, database migration checks, backend lint/tests, frontend lint/build, Docker builds, and an aggregate success check.

A passing CI run proves that those configured jobs passed against the tested commit and their configured environment. It does not prove that every possible behaviour is correct, that the AWS VM has deployed that commit, or that browser journeys work against the public application.

Always record the exact commit SHA and CI run URL. If new code is committed after a passing run, the old result does not automatically validate the new code.

A job can also be green because a test command is optional, skipped, or configured to tolerate “no tests collected.” Read the workflow and job logs, not only the green badge.

## 8. QA evidence and defect reporting

A useful finding includes:

1. **Title and priority:** concise description of the risk.
2. **Observed behaviour:** what the code or test actually showed.
3. **Expected behaviour:** what the requirement or contract says should happen.
4. **Impact:** why a user, data integrity, security, or deployment is affected.
5. **Reproduction steps:** the inputs and sequence needed to reproduce it.
6. **Correction:** what needs to change.
7. **Verification:** the exact retest that will prove the correction.
8. **Closure evidence:** test result, commit, CI run, deployment version, logs, or screenshots.

Distinguish a source-level finding from a runtime finding. Source inspection can identify a missing route or hard-coded credential. It cannot prove how a live AWS security group is configured. Runtime verification requires access to the actual environment and a recorded result.

## 9. The status ladder: do not overclaim

Use these labels precisely:

- **Found in source:** the code/configuration shows the issue or behaviour.
- **Correction proposed:** a branch contains a proposed change.
- **CI passed:** the configured checks passed for a particular commit.
- **Staging verified:** a deployed test environment passed the named test.
- **Production verified:** the actual live environment passed the named test.

An open pull request is not merged. A merged change is not necessarily deployed. A deployment job passing is not proof that all user journeys work. This distinction protects the team's credibility during a defence.

## 10. How to demonstrate QA professionally

For each major feature, prepare a short demonstration with a clear expected result.

Example: user isolation.

1. Register or use two test accounts.
2. Log in as User A and create a fake expense.
3. Confirm User A can read it.
4. Log in as User B.
5. Attempt to retrieve or export User A's data.
6. Confirm the API does not expose User A's record.
7. Save the test result and remove any sensitive test token from screenshots.

Example: budget uniqueness.

1. Create a budget for one user, month, and category.
2. Submit the same logical budget again.
3. Confirm the duplicate is rejected with the documented conflict status.
4. Submit two simultaneous requests in a controlled test.
5. Confirm the database retains only one matching budget.
6. Confirm the failed transaction is rolled back and a later valid request succeeds.

## 11. Difficult defence questions and answers

**Q1. What is the difference between unit, integration, and E2E testing?**  
Unit tests check small pieces of logic; integration tests check components working together; E2E tests follow a user journey through the full application. They find different types of defects.

**Q2. If all CI jobs are green, can you guarantee the application has no bugs?**  
No. CI only checks the tests and steps configured for that commit. It reduces risk but cannot cover every input, environment, or user behaviour.

**Q3. Why should tests use a separate database?**  
To prevent test data from corrupting real records and to make results repeatable. The test setup should explicitly guard against running destructive tests against production.

**Q4. Why test two users?**  
To prove that the ownership boundary works. A test where one user only reads their own data cannot detect every cross-user exposure bug.

**Q5. What is a regression test?**  
A test that preserves a known correction so the same defect is caught if it returns in a later change.

**Q6. Why is a successful HTTP response not enough?**  
The status may be wrong, the response may contain incorrect data, or the database may not have persisted the change. Assert the status, payload, and resulting state.

**Q7. Why test month boundaries?**  
Date filters often fail around midnight or the last day of a month. Testing the first day and first day of the next month confirms the range is defined precisely.

**Q8. What is test isolation?**  
Each test starts from a known state and does not depend on leftovers from another test. Isolation improves repeatability and makes failures easier to diagnose.

**Q9. What is the difference between a source audit and runtime testing?**  
A source audit examines code/configuration. Runtime testing exercises the actual running system. Source can show that a route exists, but only a request can verify how the deployed route behaves.

**Q10. What does a passing migration job prove?**  
It proves the migrations passed in the database environment used by that job. It does not prove that the populated AWS database is updated or that deployment recovery is safe.

**Q11. How do you know a fix is really fixed?**  
Reproduce the original defect, apply the correction, run a test that would fail without the correction, check for regressions, and record the exact commit and results. For deployment defects, repeat the test in the relevant environment.

**Q12. Why is “no tests collected” not a strong pass?**  
Because no behaviour was exercised. The pipeline may remain green for scaffold convenience, but the team must not use that result as evidence of functional correctness.

**Q13. What evidence should close a production deployment finding?**  
The exact deployed commit/image, deployment logs, successful public health checks, relevant browser/API test results, and a rollback or recovery result if that was part of the risk.

**Q14. Why are screenshots not enough by themselves?**  
A screenshot shows one visible state but may not reveal which code version produced it or whether data persisted. Combine screenshots with test outputs, request responses, commit identifiers, and deployment records.

## 12. QA defence checklist

Be ready to explain test levels, fixtures, assertions, test database isolation, CI jobs, migration testing, security regression tests, and the difference between a passing build and a verified deployment. For every claim, name the evidence that supports it.
