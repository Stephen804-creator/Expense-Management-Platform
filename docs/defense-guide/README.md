# Expense Tracker — Project Defense Study Guide

This folder contains eight connected study documents for the Expense Tracker capstone. They are written for teammates who need to understand the system from first principles, explain design decisions in simple language, and answer follow-up questions without memorising scripts.

## Read in this order

1. [Session 1 — Frontend and React](01-frontend-react.md)
2. [Session 2 — Backend and REST API](02-backend-api.md)
3. [Session 3 — PostgreSQL and data integrity](03-database-postgresql.md)
4. [Session 4 — Authentication and application security](04-authentication-security.md)
5. [Session 5 — Expense Tracker business logic](05-business-logic.md)
6. [Session 6 — Testing, QA, and CI](06-testing-qa-ci.md)
7. [Session 7 — Docker, Nginx, AWS, and deployment](07-docker-nginx-aws-deployment.md)
8. [Session 8 — End-to-end architecture, capstone alignment, and defense](08-system-integration-capstone-defense.md)

## Important note about the current repository

The repository currently contains both a root-level `src/` frontend and a `frontend/src/` frontend. The Docker Compose frontend service builds from `./frontend`. The root README claims that application records persist in browser localStorage, while `frontend/src/App.jsx` loads categories, expenses, budgets, and summaries through the Flask API. This is a documentation/source inconsistency; trace the active data path and update the README before claiming which persistence design is used.

Before the defense, use the active Docker build context and the deployed image to establish which frontend users actually receive. Update the project README so its feature and persistence claims match that implementation. Do not claim that a feature is backed by PostgreSQL unless its request path and persistence have been verified.

## Feature claims must match the deployed version

The root README advertises income management, but the Compose frontend entry point does not import an Income page, and the reviewed backend route directory does not show an income-specific API module. Treat income tracking as unverified in the active API-backed deployment path until the team demonstrates the full create/save/retrieve flow. Either implement and test it or revise the feature description to match the actual scope.

## One-file copy

If you want to send or read the entire guide as one normal Markdown file, use [the complete eight-session guide](EXPENSE_TRACKER_DEFENSE_GUIDE_ALL_SESSIONS.md). The eight individual files remain available above.

## How to study each document

For each concept, answer these questions in your own words:

- What is it?
- What problem does it solve?
- How does it work, step by step?
- Why did this project need it?
- Which other part of the application depends on it?
- What can fail if it is missing or incorrect?
- How would we test that it works?
- What evidence can we show the tutor?

Do not memorise a sentence you cannot explain. If a tutor changes the scenario, reason from the concept and the system's actual behaviour.

## Accuracy note: implemented, proposed, and not yet proven

This guide deliberately separates three things:

- **Source observation:** what can be seen in the repository source.
- **Proposed correction:** a change being prepared or reviewed in a branch or pull request; do not describe it as already present on `main` unless it has been merged.
- **Runtime evidence:** proof from a running environment, such as an AWS URL, browser journey, health response, deployment log, or database check.

The repository has a React frontend, Flask backend, PostgreSQL integration, Docker files, Nginx configuration, and GitHub Actions workflows. However, having two components in one repository does not by itself prove that the frontend uses the backend for every feature. The existing frontend documentation describes browser `localStorage` persistence, while the backend exposes database-backed API routes. Verify the actual frontend calls and end-to-end behaviour before claiming full integration.

The tutor has approved AWS as the deployment target. The deployment workflow must use AWS-compatible host, authentication, and deployment settings. This guide does not claim that automated AWS deployment or a live release has been independently verified.

The exact wording of the tutor's capstone handout takes precedence over this guide. Use the guide to understand and demonstrate the work, not to replace the official requirements or fabricate evidence.
