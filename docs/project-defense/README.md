# Expense Tracker — Project Defense Study Pack

**Project description:** Enhanced Personal Expense Management Platform.

This pack helps every team member explain the application from first principles, connect the components, answer difficult technical questions, and describe what the available evidence actually proves.

## The eight documents

1. [Frontend development — React and the user interface](01-frontend-react.md)
2. [Backend development — Flask REST API and business rules](02-backend-flask-api.md)
3. [Database — PostgreSQL, relationships, constraints, and migrations](03-database-postgresql.md)
4. [Authentication, authorization, and application security](04-auth-security.md)
5. [End-to-end architecture and component integration](05-system-architecture-and-data-flow.md)
6. [Testing, quality assurance, and evidence](06-testing-and-qa.md)
7. [Docker Compose, Nginx, networking, and health checks](07-docker-nginx.md)
8. [CI/CD, AWS deployment, operations, and capstone defence](08-cicd-aws-capstone.md)

## The four principles used throughout

Every guide does four things:

1. **Teaches the concept from the beginning:** what it means, how it works, why it exists, and what can fail.
2. **Connects it to the rest of the application:** how the frontend, API, database, security, tests, containers, and cloud deployment depend on each other.
3. **Applies it to Expense Tracker:** examples use users, expenses, categories, budgets, monthly summaries, charts, filters, and CSV export.
4. **Aligns it to the capstone:** architecture diagram, repository and README, React, Flask REST API, PostgreSQL, Docker Compose, GitHub Actions CI/CD, Docker images, Nginx reverse proxy, health checks, secure environment variables, logs/monitoring, and live deployment.

## A crucial rule for an honest defence

A design description is not proof that the feature works. A route, React component, database table, workflow file, Dockerfile, or health-check definition proves that something was written or configured; it does not by itself prove that the complete live journey works.

Separate these claims:

- **Designed:** this is the intended architecture or behaviour.
- **Implemented:** code for the behaviour exists.
- **Tested:** a recorded test exercised it and passed.
- **Deployed:** the relevant version is running in the target environment.
- **Verified live:** a real request to the deployed application produced the expected result.

Do not claim a stronger status than the evidence supports. If a question exposes a gap, explain the intended design, state what has been verified, and identify the next test or correction.

## Deployment target

The capstone PDF names Azure VM deployment. The team has stated that the tutor accepted AWS as an alternative and the application was deployed on AWS. This pack therefore teaches the cloud section using **AWS EC2 as the VM target**. Before the defence, make sure the actual repository workflow, README, secret names, architecture diagram, and live URL agree with the AWS deployment that the team will demonstrate. Do not describe Azure resources as if they were used.

## How to study

Read the explanation, draw the flow or data relationship, explain it aloud in your own words, answer the difficult questions without looking, then locate the related source file. For answers that depend on behaviour, collect evidence such as test results, a CI run, deployment logs, or a live request. Remove tokens, passwords, and personal financial data from screenshots.

A strong defence answer follows this pattern: **definition → reason → how it works here → connection to another component → failure/risk → evidence**.

## Project in one sentence

Expense Tracker is a web application that lets an authenticated user record and review financial transactions, organise them by category, set budgets, view summaries, filter records, and export expense data, with a React frontend communicating with a Flask REST API and PostgreSQL through a containerised, reverse-proxied deployment.

Use that sentence as a starting point. A defence requires you to explain every part, not merely repeat the sentence.
