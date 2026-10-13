# Session 8 — System Architecture, Testing, QA, Capstone Requirements, and Defense
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. How to explain the whole application

Expense Tracker is a personal expense-management web application. Its purpose is to let users manage expense records and categories, review monthly spending, set budgets, filter records, view dashboard summaries/charts, and export data to CSV. The project also demonstrates cloud and DevOps practices rather than only building a user interface.

The logical architecture is:

**User's browser → React frontend → Nginx reverse proxy → Flask REST API → PostgreSQL database.**

GitHub stores the source code and change history. GitHub Actions runs automated checks. Docker packages the application services. AWS provides the approved cloud deployment target. Health checks and logs help the team verify and troubleshoot the running system.

Explain each boundary separately. The browser is where the user interacts. React presents the interface. Nginx is the web entry point and traffic router. Flask implements application behavior and access rules. PostgreSQL stores persistent data. GitHub Actions checks proposed code. Docker provides consistent packaging. AWS provides infrastructure. None of these components automatically performs the job of every other component.

## 2. Architecture is about responsibility and communication

An architecture diagram should not be a collection of technology logos. It should show the components, the direction of communication, and the trust boundaries. It should distinguish public traffic from internal service traffic, show where the database resides, and indicate how code changes travel through CI/CD to deployment.

A good explanation answers:
- Which component receives a browser request?
- Where is the authentication decision made?
- Which component validates business rules?
- Which component stores the record permanently?
- Which ports or routes are exposed publicly?
- Where are secrets configured?
- How does code move from GitHub to AWS?
- How is the deployed version identified?
- How are failures detected and investigated?

A diagram is not proof that the implementation matches it. Verify the actual Compose file, Nginx configuration, workflow, and deployment setup. If the diagram shows a private database but the real deployment exposes its port publicly, the diagram is inaccurate.

## 3. What the capstone actually asks for

The supplied handout asks for:
1. An architectural diagram.
2. A GitHub repository with code and a README that explains how to run the project.
3. A live URL for testing; a server IP is acceptable instead of a custom domain.
4. The type and purpose of the web application.
5. Clear project objectives.
6. Team member names.
7. A presentation covering introduction, problem statement, architecture, tech stack, CI/CD, deployment, challenges, solutions, results, future improvements, conclusion, and demonstration screenshots.

It also lists application features: authentication, add/edit/delete expenses, expense categories, monthly summaries, budget limits, dashboard charts, date/category filtering, and CSV export.

Its DevOps requirements include React, Flask REST API, PostgreSQL, Docker Compose, GitHub Actions CI/CD, Docker image build and deployment, Nginx reverse proxy, application health checks, and secure environment variables. Common requirements include source control with branches and pull requests, automated build/tests, secrets protection, appropriate cloud IAM, application logs, basic server/container health monitoring, architecture explanation, setup and deployment instructions, environment variables, and troubleshooting.

The PDF names an Azure VM. The tutor approved AWS as an alternative, so AWS is the chosen implementation target. When presenting this, state the approval accurately. Do not rewrite the original document's wording or claim that Azure is literally absent from it.

## 4. Testing layers and what each proves

Testing is not one single activity.

**Unit tests** check a small unit of logic in isolation, such as a date-validation helper or a calculation.

**API/route tests** send requests to the Flask application and check status codes, response bodies, and business rules. They can test invalid inputs, authentication requirements, conflicts, and user isolation.

**Database/migration tests** verify schema constraints and data behavior, including whether migration scripts can apply to a controlled test database.

**Frontend tests** check components or interactions in the browser/test environment. A frontend build is not a substitute for functional tests.

**Integration tests** verify components work together, such as Flask and PostgreSQL or Nginx routing to Flask.

**End-to-end tests** follow a user journey through the running application, from browser interaction through API and database behavior and back to the interface.

**Manual acceptance tests** check whether the feature meets the user's intended behavior. They should still be repeatable and recorded.

A test result only proves what the test actually exercised. If a test checks a health endpoint, it does not prove CSV export. If CI builds the frontend, it does not prove a user can create and edit an expense through a browser.

## 5. Negative tests are essential

A demo that shows only successful input proves very little about robustness. Include negative cases such as:
- missing or malformed fields;
- invalid or impossible dates;
- non-finite amount values;
- expired or missing authentication;
- attempts to access another user's expense;
- duplicate monthly budget;
- category deletion while a budget depends on it;
- database failure and rollback;
- unavailable PostgreSQL;
- invalid production secret configuration;
- export attempts with unauthorized records;
- a migration failure during deployment.

For each test, write the precondition, action, expected status/result, actual result, and evidence. If the test has not been run, mark it as planned rather than passed.

## 6. How to read CI evidence honestly

A green GitHub Actions run means the configured jobs passed for a particular commit. The commit SHA matters because later changes may not have been tested by that run. Always match the test result to the exact code revision.

For this correction branch, a recorded checkpoint run passed all eight jobs for commit 927e35eaf3af827d8b52ef93f6f883c6bfcaa667, including the migration check and 35 backend tests. That is a historical CI checkpoint. Later source edits may have created a new commit, so check the latest run before stating that the current head is green.

The known CI checkpoint does not prove a populated AWS database migrated, that the latest image is running on AWS, that public health routes work, or that full browser journeys passed. These need separate runtime evidence. The frontend package has also been identified as lacking a functional test script at the recorded review point; recheck the current package before presenting this as the latest state.

## 7. QA findings: how to explain one professionally

A useful finding should state:
1. **Observation:** What the code or test actually shows.
2. **Risk/impact:** Why the behavior matters to a user or operator.
3. **Reproduction:** The precise steps or input.
4. **Expected behavior:** What should happen.
5. **Correction:** The proposed or implemented change.
6. **Verification:** The test that proves the correction.
7. **Evidence and status:** Commit, CI run, runtime result, and what remains unverified.

Do not turn a hypothesis into a confirmed defect. Label source-level concerns separately from runtime observations. Do not say “fixed” when code changed but the regression test has not passed. Do not say “deployed” when only an image build passed. This precision protects the team's credibility.

For example, category-budget integrity is a concrete case: a category-specific budget should not silently become a whole-month budget if the category is deleted. The correction adds an API pre-check and a database RESTRICT rule. CI can test the rule and route behavior; AWS runtime evidence is still needed to confirm the deployed database has the migration.

## 8. Source control and pull requests

A branch isolates a line of work. A commit records a change. A pull request proposes that changes be reviewed and merged into a target branch. CI can run on a pull request to catch regressions before merging.

A pull request being open is not the same as being merged. A correction being present on a branch is not the same as being present in the production deployment. During defense, state which branch contains the change, whether the PR was merged, and whether the deployment includes that commit.

PR #53 is being used for QA findings and audit documentation. PR #54 is being used for proposed application corrections. Both should remain separate in purpose: audit evidence belongs in the QA documentation stream, while executable corrections and regression tests belong in the code-correction stream. The project-defense learning guides are kept separately in the fork branch so they do not clutter either PR by default.

## 9. Release and acceptance checklist

Before the final demonstration, verify each item and record evidence:

- The project name and purpose are clear.
- The architecture diagram matches the real configuration.
- The README includes setup and run instructions.
- The correct React, Flask, and PostgreSQL versions/configuration are documented.
- Required environment variables are named and explained without exposing secrets.
- Authentication and cross-user isolation are tested.
- Expense create/read/update/delete work.
- Categories and budget behavior follow an agreed policy.
- Monthly summaries and charts match independently calculated results.
- Date/category filters work at boundary values.
- CSV export includes only authorized data and follows the expected filtering rules.
- Duplicate-budget and category-deletion conflicts are handled correctly.
- Docker services start and health checks behave as intended.
- CI is green for the exact release commit.
- Migrations have been tested on a clean database and a populated existing database.
- The running AWS deployment is tied to a known commit/image.
- Public health endpoints and key browser journeys have been tested.
- Logs and configuration do not expose credentials.
- The live URL or IP is accessible for the defense.
- Screenshots show actual application behavior and are not presented as proof of a live test if they were captured elsewhere.
- Team members can explain their contributions accurately.

## 10. A strong project introduction

A concise introduction can say:

“Expense Tracker is a personal expense-management web application designed to help users record and organize expenses, view monthly spending, set budget limits, filter transactions, and export records. The frontend is built with React, the backend exposes a Flask REST API, and PostgreSQL stores persistent data. Docker Compose packages the services, Nginx routes web traffic, and GitHub Actions automates configured checks. The team deploys to AWS under the tutor-approved alternative to the Azure VM named in the capstone handout. Our presentation will show the architecture, the user journey, the release process, the tests, and the evidence for what is verified.”

Adapt the last sentence to the actual deployment status. Do not claim a feature is complete until the team can demonstrate it.

## 11. Difficult defense questions

**Q: What problem does this project solve?**  
It provides a single application for recording and organizing personal expenses, reviewing spending, comparing spending with budgets, filtering records, and exporting data. Explain the user problem rather than merely listing technologies.

**Q: Why use React, Flask, and PostgreSQL together?**  
React handles the browser interface, Flask centralizes API logic and authorization, and PostgreSQL provides persistent relational storage and constraints. Each has a distinct responsibility.

**Q: Why not store everything in browser local storage?**  
Local browser storage is not a suitable authoritative multi-user database. It is device-specific, can be modified by the user, and does not provide server-side relational constraints or shared persistence.

**Q: What is the difference between a feature and its evidence?**  
A feature is behavior the application is intended to provide. Evidence is the test, logs, API response, database state, or recorded demonstration proving that behavior occurred.

**Q: If all CI jobs pass, why test AWS?**  
CI runs in a controlled environment. AWS may have different secrets, network rules, persistent data, image tags, routing, and startup conditions. Runtime verification checks those deployment-specific factors.

**Q: What is the biggest risk of saying “we deployed it” too early?**  
It may confuse a successful build or deployment command with a working release. A release must be tied to a tested commit and verified through the actual endpoint and critical user journeys.

**Q: What if a tutor asks about a feature the team did not implement?**  
Be honest. Explain the current behavior, its limitation, the design needed, and how it would be tested. Do not invent a feature or pretend a planned test passed.

**Q: How do you know the monthly total is correct?**  
Define the date period, create a known test dataset, calculate the expected total independently, call the API, and compare the displayed summary. Include boundary dates and different categories.

**Q: What is the difference between a challenge, a solution, and a result?**  
The challenge is the observed problem; the solution is the change made; the result is the evidence that the change achieved the intended outcome. A proposed solution is not a proven result until verified.

## 12. How to answer a confusing question

When a question is unfamiliar, do not guess immediately. Identify the layer being discussed: interface, API, database, security, container, pipeline, or cloud infrastructure. Explain the basic concept, trace the request or data through the system, state the rule that applies, and then give the project's concrete example. If you do not know whether the current code implements something, say that you would verify the source or test rather than inventing an answer.

A good answer is accurate, structured, and evidence-based. Confidence without evidence can make a technically correct project sound unreliable.

## 13. Final source-of-truth rule

The capstone PDF defines the submission requirements. The repository shows the implementation. CI logs show which automated checks ran on which commit. AWS runtime evidence shows what actually runs in the cloud. These are related but different sources. When they disagree, identify the discrepancy and resolve it rather than silently treating them as equivalent.

## 14. Capstone alignment

This session directly supports the required architecture diagram, repository and README, live URL, project purpose/objectives, team contributions, presentation topics, CI/CD, deployment, challenges, solutions, results, future improvements, and demonstration screenshots. It also covers the required stack and common cloud/DevOps expectations. The handout names Azure VM; AWS is the tutor-approved alternative chosen by the team.
