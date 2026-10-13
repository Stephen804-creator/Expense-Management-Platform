# Project Defense Study Pack — Expense Tracker

**Project title:** Expense Tracker  
**Description/subtitle:** Enhanced Personal Expense Management Platform  
**Purpose of this folder:** Eight detailed study sessions for the team to understand the application, connect the technical layers, prepare for difficult defense questions, and explain the implementation accurately.

These guides are learning documents, not evidence that every described behavior has been deployed or verified. The source code, test results, capstone handout, and actual AWS environment remain the sources for confirming implementation status.

## The eight sessions

1. [Frontend Development with React](01-frontend-react.md) — components, state, forms, charts, filters, CSV export, usability, and frontend/API interaction.
2. [Backend Development with Flask REST API](02-backend-flask-api.md) — routes, HTTP, validation, authorization, transactions, errors, and business rules.
3. [PostgreSQL, Data Modelling, and Transactions](03-postgresql-data-model.md) — tables, keys, relationships, uniqueness, NULL behavior, transactions, migrations, and persistence.
4. [API Integration and End-to-End Feature Flows](04-api-integration-and-feature-flows.md) — trace a user action through React, Nginx, Flask, PostgreSQL, and back; understand filters, budgets, exports, and failures.
5. [Authentication, Authorization, and Application Security](05-authentication-security.md) — passwords, JWT, secrets, ownership, CORS, TLS, rate limiting, safe errors, and threat modelling.
6. [Docker, Nginx, GitHub Actions, and CI/CD](06-docker-nginx-cicd.md) — images versus containers, Compose, health checks, migration order, CI evidence, and release traceability.
7. [AWS Deployment, Networking, Health Checks, and Monitoring](07-aws-deployment-monitoring.md) — EC2 concepts, security groups, IAM, deployment versions, persistence, monitoring, and evidence. The tutor approved AWS as an alternative to the Azure VM named in the original capstone handout.
8. [System Architecture, Testing, QA, Capstone Requirements, and Defense](08-architecture-testing-capstone-defense.md) — architecture, the actual capstone checklist, testing layers, CI limitations, QA evidence, presentation structure, and difficult questions.

## Recommended study method

Read the sessions in order. For each session:

1. Explain the concept aloud without looking at the document.
2. Open the corresponding source code or configuration in the repository.
3. Identify the actual route, component, model, workflow, or deployment setting that demonstrates the concept.
4. Answer the hard questions aloud using the application's real behavior.
5. Mark each statement as one of: **verified in source**, **passed in CI for a named commit**, **verified on AWS**, or **not yet verified**.

Do not memorize sentences without understanding them. A tutor can ask the same concept using different words. The aim is to understand the responsibility, data flow, risks, and evidence.

## Important project-specific accuracy notes

- The capstone PDF asks for React, Flask REST API, PostgreSQL, Docker Compose, GitHub Actions CI/CD, Docker image build/deployment, Nginx, health checks, secure environment variables, source-control workflow, documentation, and a live test URL.
- The PDF names Azure VM deployment. The tutor approved AWS as an alternative, and the team uses AWS. Describe this as an approved substitution; do not say the PDF originally specified AWS.
- A green CI run proves only the checks that actually ran for that commit. It does not automatically prove the current branch is green, a populated AWS database migrated, the latest image is live, or end-to-end browser journeys passed.
- The category-budget correction adds a route-level pre-check and migration 004 using ON DELETE RESTRICT. Explain the purpose: deleting a category must not silently convert a category-specific budget into a whole-month budget. Confirm current CI and AWS evidence before claiming the running environment has the correction.
- The current guides are stored on the separate fork branch `stephen/project-defense-guides`. They are not automatically part of PR #53 or PR #54 and have not been merged into the original project.

## Where these documents belong professionally

These guides are useful team training and defense preparation materials. They are not required runtime code and should not be mixed into the production application bundle. If the team agrees to keep them in the main repository, add the folder as a separate documentation-only pull request or a clearly scoped addition to an appropriate documentation PR after review. PR #53 is the closer fit if the team wants repository documentation/QA evidence, but its existing purpose should be checked before adding a large training pack. PR #54 should remain focused on executable corrections and regression tests.

Before proposing a merge, ask the team lead whether the repository should contain internal defense notes. A concise architecture/operations guide in the main README or docs is often useful to future maintainers; eight long study guides may be better kept in a dedicated docs folder or shared separately with the team.
