# Session 7 — Docker, Nginx, GitHub Actions, and AWS Deployment

## 1. What deployment means

Deployment makes a version of the application available in an environment where users can access it. It is not the same as writing code, committing code, building a container image, or passing CI.

A deployed system needs the right application version, runtime configuration, network routes, database, secrets, persistent storage, health checks, and a way to recover if an update fails.

The team has stated that the tutor approved AWS as the alternative to the cloud platform named in the original workflow. This guide therefore explains the intended deployment on AWS. It does not claim that the current Azure-labelled workflow is already configured for AWS or that a live deployment has been independently verified.

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

The reviewed `.github/workflows/cd.yml` still names an Azure VM and Azure-specific secret variables. It also describes pulling images and running Compose, while the current Compose file and migrations must be examined together to confirm that schema changes are actually applied before the application restarts.

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

**Q8. Why is the current Azure-labelled workflow important if the tutor approved AWS?**  
Because the source configuration must match the actual target. Azure secret names and host assumptions do not magically become AWS settings. The workflow or the documented deployment method must be updated and verified.

**Q9. What is the risk of deploying `latest`?**  
It can point to different image content over time, making rollbacks and evidence less precise. Immutable tags or digests make it easier to identify the exact artifact that was tested.

**Q10. What would you do if a migration failed during deployment?**  
Stop the release, preserve logs, avoid restarting into an incompatible schema, inspect the migration's partial effects, and follow a tested recovery plan. Restore from backup if necessary. Do not rerun blindly without understanding whether the migration is safe to repeat.

**Q11. Is a persistent volume the same as a backup?**  
No. A volume protects against some container lifecycle events; a backup is a separate recovery mechanism and must be tested.

**Q12. How would you prove the public app is the version that passed CI?**  
Record the tested commit, build images from that commit, deploy immutable image tags/digests, and expose or record the deployed version. Then compare the live deployment record to the CI run.

## 12. Explain it in one minute

“Docker packages the frontend and backend and Compose connects them to PostgreSQL and Nginx. Nginx routes public requests, PostgreSQL uses a persistent volume, and health checks provide limited signals about service availability. GitHub Actions CI checks code and builds; CD publishes and deploys a specific version. Our agreed cloud target is AWS, but the workflow source reviewed still contains Azure-specific configuration, so it must be aligned before we claim automated AWS deployment. We prove a release with the exact commit or image digest, migration logs, public health checks, and a real user journey.”
