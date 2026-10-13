# Session 8 — CI/CD, AWS Deployment, Operations, and Capstone Defence

## 1. What deployment means

Deployment means making a selected version of the application available in a target environment. It is more than uploading source code. The environment needs compatible runtime dependencies, configuration, secrets, database access, network rules, a running process, and a way to verify that the application works.

In this project, the agreed deployment target is AWS, using Amazon EC2 as the virtual machine host. EC2 supplies the server; Docker Compose runs the application services on it. Describe only the resources that the team actually configured and can demonstrate.

The team should describe what is actually running: frontend, Flask backend, PostgreSQL, and Nginx containers; the public entry point; where the database files persist; how secrets are provided; and how the deployed version is identified. Do not claim PostgreSQL is on Amazon RDS if it is running as a Compose container on the EC2 instance.

## 2. What is AWS EC2?

Amazon EC2 is a service for launching virtual servers in AWS. The instance has a machine image, CPU/memory capacity, storage, network interfaces, and an operating system. The team installs or configures the runtime and deploys the application to it.

Important EC2-related concepts:

- **Instance:** the virtual server running the application containers.
- **Security group:** a virtual firewall controlling inbound and outbound traffic for the instance/network interface.
- **Key pair / SSH access:** a method for administrators or deployment automation to connect securely to the instance.
- **EBS volume:** block storage that can persist independently of an instance's temporary runtime state, depending on configuration and lifecycle.
- **Public IPv4 address or DNS name:** a route clients can use to reach the service, subject to network rules.
- **IAM role/policy:** permissions for identities or services to access AWS resources.
- **Cloud monitoring/logging:** tools for inspecting instance and service health, resource use, and failures.

A security group should expose only the required public ports. SSH should be restricted to authorised source addresses or another controlled access method. PostgreSQL should not be publicly exposed for ordinary application use. Production credentials must not be stored in a public repository.

## 3. What is CI?

Continuous Integration (CI) automatically checks code changes when developers push commits or open pull requests. The aim is to discover errors before the changes are merged or deployed.

The repository's CI workflow can perform tasks such as:

- detecting which components exist;
- scanning for committed environment files and secrets;
- starting a temporary PostgreSQL service;
- applying SQL migrations and checking expected tables;
- installing backend dependencies;
- running backend linting and tests;
- installing frontend dependencies;
- running frontend linting and a production build;
- building Docker images;
- reporting an aggregate result.

The exact workflow file and job logs are the evidence. Do not claim a job runs merely because its name appears in a comment. Inspect conditions, skipped steps, and test commands.

A passing CI run is tied to a specific commit SHA. If code changes afterwards, the earlier result does not verify the newer code. If a test command is optional or configured to tolerate the “no tests collected” result, a green badge alone does not prove that the component has functional test coverage.

## 4. What is CD?

Continuous Delivery/Deployment (CD) automates the movement of a tested version toward a running environment. A typical pipeline is:

Source commit → CI checks → image build → image publish → deployment to EC2 → health check → browser/API verification.

A workflow may publish Docker images to a container registry and then use SSH or a deployment agent to update the EC2 host. The deployment should identify the exact code revision or immutable image digest being installed.

A moving tag such as latest is convenient but can change over time. The team should also retain an immutable commit-based tag or digest and record which version is running. Otherwise, it may be difficult to prove that the deployed application is the one that passed CI or to roll back to the prior known-good version.

## 5. Important current repository alignment issue

The current correction branch's CD workflow is written for AWS EC2. It runs after CI succeeds on a push to `main`, publishes commit-tagged container images to GHCR, and only attempts deployment when `DEPLOY_ENABLED=true`. The deployment uses `AWS_EC2_HOST`, `AWS_EC2_USER`, and `AWS_EC2_SSH_KEY`, checks out the tested commit on the VM, applies the configured migrations before restarting application services, and performs a health check. This is evidence of workflow configuration, not proof that the latest image is already running on the live VM. The team must show the successful deployment run, the deployed commit/image, and live health/browser evidence.

This means the repository's current workflow must be reconciled with the AWS deployment before the team claims that the checked-in CD workflow automatically deploys to AWS. The application might have been deployed manually or by a different process, but that is not the same claim as “this workflow deploys it to AWS.”

Before the defence, update the workflow, deployment documentation, secret/variable names, and architecture diagram to match the approved AWS process, then run it and capture evidence. Do not rename variables without checking the deployment command, instance configuration, and GitHub environment settings. The target is AWS; the automation must be made consistent with that target and verified.

## 6. Deployment sequence and why order matters

A safe deployment needs a defined sequence:

1. Identify the exact revision that passed CI.
2. Build and publish the corresponding images.
3. Connect to the intended AWS EC2 instance using protected deployment credentials.
4. Pull the intended image tags/digests.
5. Ensure required environment variables and secrets exist.
6. Ensure PostgreSQL is ready.
7. Apply the required migrations in the correct order.
8. Stop if a migration fails; do not continue blindly with an incompatible schema.
9. Start or update the application services.
10. Check container health and the public health route.
11. Exercise important browser/API journeys.
12. Record the deployed commit/image and deployment result.
13. Keep a recovery/rollback procedure available.

Why run migrations before restarting the application? The new code may depend on new columns, indexes, or foreign-key rules. Starting code against an old schema can cause runtime errors. However, database migrations are not automatically reversible. A rollback may require a forward fix or restoring from a verified backup, so the team must understand the migration's impact before applying it to production.

## 7. Environment variables and GitHub secrets

Configuration differs between local development, CI, and production. The same source code may need a local database host, a temporary CI database host, and a production database host.

Environment variables can supply non-secret configuration such as service hostnames and database names. Secret stores or protected GitHub Actions secrets should supply passwords, signing keys, and SSH private keys. A secret must not be printed in workflow logs or embedded into a frontend build.

The production application should fail clearly if required secrets are missing or set to known placeholders. Test secrets should be separate from production secrets. Access to deployment secrets should be restricted to trusted workflows and people.

Secret scanning is useful, but it does not guarantee that every credential is detected or that a credential has never been exposed. If a secret was committed, rotate it; deleting it from the latest file may not remove it from Git history.

## 8. Health checks, logs, and monitoring

The capstone requires a health endpoint, application logs, and basic server/container health monitoring.

A simple health endpoint may report that Flask responds. A database health endpoint can attempt a lightweight query such as SELECT 1. These checks answer different questions. A process can be alive while its database connection is broken.

A deployment should check the actual public route through Nginx, not only the internal Flask route. The public path can fail because of Nginx routing, service DNS, container health, or firewall configuration even when Flask works inside the container.

Logs should help answer:

- Which service failed?
- At what time?
- Which request or deployment was involved?
- Was the database reachable?
- Which code/image was running?
- Did the health check recover?
- Were credentials or sensitive data accidentally logged?

Do not place tokens, passwords, SSH keys, or full personal financial records in logs. Monitoring is not just a dashboard screenshot; it is the ability to detect and diagnose an issue with evidence.

## 9. Backups, recovery, and cost

If PostgreSQL runs in a container on EC2, the named Docker volume preserves database files when the container is recreated, but it is not a complete backup strategy. The host or storage can still fail, data can be deleted, and a bad migration can damage data.

A recovery plan should identify what is backed up, how often, where backups are stored, who can access them, and how restoration is tested. A backup that has never been restored is not strong evidence of recoverability.

Cloud cost also matters. A running EC2 instance, storage, public IPv4 usage, data transfer, and other resources can incur charges. Choose a size suitable for the demo, restrict unnecessary services, monitor usage, and stop or clean up resources when the project no longer needs them. Never promise a particular monthly price without checking the current region, instance type, storage, traffic, and pricing date.

## 10. Capstone submission requirements: what to prepare

The capstone PDF asks for an architecture diagram, a GitHub repository with a well-documented README and run instructions, a live URL, the application's type/purpose/objectives, team member names, and a presentation covering introduction, problem statement, architecture, technology stack, CI/CD, deployment, challenges, solutions, outcomes, future improvements, conclusion, and screenshots/demo.

The additional application features include user authentication; add/edit/delete expenses; categories; monthly summaries; budgets; charts; date/category filters; and CSV export. The DevOps requirements include React, Flask REST API, PostgreSQL, Docker Compose, GitHub Actions CI/CD, Docker image build/deployment, Nginx reverse proxy, health checks, secure environment variables, and VM deployment.

For the defence, prepare evidence for each claim:

- **Purpose/objectives:** explain the financial tracking problem and intended user benefit.
- **Architecture:** show the actual component diagram and explain the request flow.
- **Technology stack:** explain why each technology is used, not just list names.
- **Repository/README:** demonstrate setup steps and required environment variables using placeholders.
- **Feature demo:** log in, create a test expense, filter it, view summaries/budgets, and export CSV.
- **Security:** demonstrate protected routes and two-user isolation; show secret handling without revealing secrets.
- **CI/CD:** show a run for the exact source revision and explain the jobs that actually ran.
- **Deployment:** open the AWS live URL and identify the version running.
- **Health/logs:** show public health behaviour and explain where failures are diagnosed.
- **Challenges and solutions:** describe real defects, the cause, the correction, and the evidence that verifies the fix.
- **Future improvements:** distinguish ideas from implemented capabilities.

## 11. How to explain the project's purpose

A concise explanation could be:

“Expense Tracker is a personal expense management web application. It allows an authenticated user to record financial transactions, organise them into categories, define monthly budgets, view spending summaries and charts, filter records, and export expense data. The React frontend communicates with a Flask REST API, PostgreSQL stores the durable records, Docker Compose runs the services, Nginx routes public traffic, and GitHub Actions checks and packages code before deployment to AWS.”

Then explain the business problem: people need to record spending, understand where money goes, compare actual spending with a budget, and retrieve records for review. The project makes these activities available through one interface and demonstrates cloud and DevOps practices around a working web application.

Do not claim bank integration, automatic bank transaction import, financial advice, multi-currency conversion, or features that are not implemented and verified.

## 12. Common deployment failure scenarios

**Wrong secret name:** the workflow provides a variable that the application or SSH action does not read. The service starts with missing configuration or deployment fails. Compare the code's exact variable names with GitHub secrets and environment settings.

**Wrong image tag:** CI tests one commit but deployment pulls a moving tag that now points to another image. Use commit-based tags/digests and record the deployed version.

**Database not ready:** the backend starts too early and cannot connect. Use meaningful health checks and dependency ordering, and inspect the logs.

**Migration not applied:** the code expects a new column or constraint, but the existing database has the old schema. Apply versioned migrations explicitly and verify on a populated database.

**Port blocked:** the application is healthy inside EC2 but users cannot reach it. Check the EC2 security group, host firewall, published port, Nginx status, and public address.

**Health check too shallow:** /health returns 200 while the database or main user flow is broken. Test database readiness and complete journeys.

**Failed deployment leaves the site down:** the new container is broken and the old version is gone. Test the recovery procedure and preserve a known-good image/version.

## 13. Difficult defence questions and answers

**Q1. Why AWS EC2 instead of a platform-as-a-service?**  
EC2 provides control over the virtual server, networking, Docker, and deployment process, which fits the capstone's cloud/DevOps focus. It also requires the team to manage more operational responsibilities than a managed platform.

**Q2. What is the difference between EC2 and Docker?**  
EC2 is the cloud virtual machine. Docker packages and runs application containers on that machine. One EC2 instance can host several containers.

**Q3. What is the difference between CI and CD?**  
CI checks changes through tests, linting, scanning, and builds. CD publishes and deploys a tested version and then verifies it. CI passing does not mean the production site was updated.

**Q4. What is the difference between a container image and the deployed version?**  
An image is a packaged template. The deployed version is the particular image or code revision actually running in the environment. The team should identify it by commit tag or immutable digest.

**Q5. Why not store database passwords in the repository?**  
Repositories can be copied, scanned, and retained in history. Secrets belong in protected configuration or a secret manager, and exposed credentials must be rotated.

**Q6. What does an AWS security group do?**  
It controls network traffic allowed to reach or leave the associated resource. It should expose only required ports and restrict administrative access.

**Q7. Does a public IP mean the application is available to everyone?**  
No. The instance must be running, the service must listen on the expected port, the security group and host firewall must allow traffic, and Nginx/application services must be healthy.

**Q8. Why are migrations part of deployment?**  
Application code and database schema must remain compatible. Migrations apply reviewed schema changes to existing data; deployment must stop if a required migration fails.

**Q9. How do you prove the AWS deployment matches the source code?**  
Show the CI run and tested commit, the published image tag/digest, deployment logs, and the running version information. A screenshot of the homepage alone cannot establish which commit is running.

**Q10. What is the difference between a backup and a Docker volume?**  
A volume stores persistent data outside a container's writable layer. A backup is a recoverable copy protected from some failures, including accidental deletion or host loss. A volume alone is not a backup.

**Q11. What evidence proves that the AWS deployment workflow actually deployed the version being defended?**  
Show the successful CI run for the exact commit, the CD run triggered from that successful push to `main`, the commit-tagged image published to GHCR, the deployment log showing that same commit/image was selected on EC2, and successful public health checks. Then demonstrate browser journeys against the live site. A workflow file or green CI badge alone proves neither that deployment occurred nor that the live site is running that version.

**Q12. What would you do if deployment succeeded but health checks failed?**  
Stop treating the release as successful. Inspect service status and logs, verify ports and environment, check database/migration state, and restore the previous known-good version or follow the documented recovery procedure.

**Q13. What is the difference between monitoring and logging?**  
Logs record events and diagnostic details. Monitoring evaluates signals such as health, errors, latency, and resource use to show whether the system is operating within expected conditions. They complement each other.

**Q14. Why should the architecture diagram match the actual deployment?**  
The diagram is a technical claim. If it shows a managed database or automated AWS deployment that is not actually used, the team cannot defend the design with evidence. Label current, planned, and optional components clearly.

**Q15. What would you say if the tutor asks about a known gap?**  
State the expected behaviour, explain what the current implementation does, describe the risk, and show the correction or test plan. Then state exactly what evidence is still required. Do not invent a successful test or claim an open pull request has already been deployed.

## 14. Final presentation and defence checklist

Before the presentation, verify:

- The project name and description are consistent.
- The architecture diagram matches the deployed AWS design.
- The README contains working setup and deployment instructions.
- The live URL opens from outside the VM.
- A test user can log in and complete the core expense journey.
- Categories, budgets, summaries, filters, and CSV export are demonstrated with test data.
- Two-user isolation is tested.
- The exact CI run and source commit are identified.
- The workflow is configured for AWS, not merely renamed in the presentation.
- Secrets are not visible in screenshots or logs.
- Health endpoints and logs are demonstrated.
- The team knows the current limitations and does not overclaim.
- Each member can explain their own work and how it connects to the rest of the system.

## 15. The final defence mindset

The goal is not to memorise definitions. The goal is to understand why the system is divided into components, how data moves through it, what each component guarantees, what can fail, and how the team knows the deployed application behaves correctly.

For any difficult question, slow down and follow the chain: user action → HTTP request → authentication and validation → database operation → response → UI update → test evidence. If the question concerns deployment, add the tested commit, image version, AWS configuration, health check, and runtime evidence.
