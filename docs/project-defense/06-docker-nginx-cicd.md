# Session 6 — Docker, Nginx, GitHub Actions, and CI/CD
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. Why deployment engineering matters

An application can work on one developer's laptop but fail on another computer or server because of different runtimes, dependencies, ports, configuration, or operating-system packages. Deployment engineering makes the application easier to build, run, test, and update consistently.

The capstone is deliberately more than a React and Flask exercise. It also evaluates architecture, source control, automated tests, container images, deployment, health checks, environment configuration, monitoring, and documentation. These pieces explain how software moves from a code change to a running service.

## 2. What Docker is

Docker packages an application and its runtime dependencies into a container image. An image is a template; a container is a running instance of that image. An image can be versioned and promoted between environments, helping the team reduce “works on my machine” differences.

A container is not a full virtual machine. Containers share the host operating-system kernel while isolating processes and resources. They still need secure configuration, resource limits where appropriate, network rules, and updates to the underlying image.

The frontend, backend, and database can be separate services because they have different responsibilities and lifecycles. Separating services helps isolate failures and configuration, but it also creates network and startup dependencies that must be handled.

## 3. Dockerfile versus Docker Compose

A Dockerfile describes how to build one image: base image, dependencies, source files, and startup command. Docker Compose describes how multiple services work together: which images or build contexts to use, environment variables, ports, networks, volumes, health checks, and dependencies.

A Compose file is not the same as a Dockerfile. A Dockerfile builds one service's image; Compose coordinates the application stack. Neither automatically guarantees the application is production-secure.

The PostgreSQL volume is important because containers can be replaced. Persistent database data should live in a volume or managed database rather than only inside the writable layer of a disposable container. Removing a container should not accidentally mean losing the user's records.

## 4. Ports and networks

A port is a communication endpoint. A container may listen on one port internally and map it to a host port. Services in a Compose network can often reach one another by service name, while the browser reaches the public host through exposed ports and Nginx.

A common deployment mistake is using localhost incorrectly. Inside the backend container, localhost refers to that container, not necessarily the database container or the host. The backend normally connects using the database service hostname and internal port. The browser cannot use an internal Compose-only hostname unless its network environment can resolve it.

The database should not be exposed publicly without a deliberate need and strong restrictions. For a standard web deployment, browsers contact the public reverse proxy; Flask connects privately to PostgreSQL.

## 5. Startup order and health checks

Starting a container does not prove its service is ready. PostgreSQL may have a running process while it is still initializing or rejecting connections. A health check tests readiness using a defined command or endpoint.

Compose dependency ordering can control when one service starts relative to another, but “started” and “healthy” are different states. A deployment that immediately runs migrations before PostgreSQL is ready can fail intermittently.

The correction workflow uses a PostgreSQL health wait before running database migrations and restarting application services. CI checks the intended order. That is valuable evidence, but a real AWS cold-start test is still needed to prove the deployed environment behaves the same way.

## 6. Nginx as reverse proxy

Nginx is a web server and reverse proxy. In this architecture, it can serve frontend assets and route API requests to Flask. It is the public entry point for web traffic, while the database remains behind the application boundary.

The routing rules must match the API's actual paths. If Flask exposes a health endpoint at one path but Nginx forwards or maps a different path, the application may be healthy internally but fail through the public route. Therefore, health checks should be tested through Nginx as well as directly against Flask.

Nginx can also handle TLS termination, request size limits, and security headers if configured. Do not claim those features are active simply because Nginx is installed; verify the configuration and deployed behavior. A reverse proxy does not replace backend authentication or authorization.

## 7. Continuous Integration and Continuous Deployment

**Continuous Integration (CI)** automatically checks changes when code is pushed or a pull request is updated. Typical checks include linting, automated tests, frontend build, backend tests, secret scanning, database migration checks, and Docker image builds.

**Continuous Deployment/Delivery (CD)** moves a tested version toward an environment. Continuous delivery prepares a releasable version but may require approval; continuous deployment can automatically release changes when conditions pass. The exact workflow in this repository determines which term is accurate.

CI and CD solve different problems. A green CI run means the tested commit passed the configured checks. It does not automatically prove that the latest code is live on AWS, that the right secrets exist, that a populated database migrated safely, or that a browser journey works in production.

## 8. Why commit identity matters

A release must be traceable to the exact source revision and container image. A moving image tag such as latest can be overwritten by a later build. If a deployment pulls latest while another process has published a newer image, the server may run a version different from the commit that was tested.

A safer release process uses immutable image tags tied to a commit SHA or records the image digest, deploys that exact artifact, and records which version is running. A rollback should return to a known previously tested image and configuration.

The team should be able to answer: Which commit did CI test? Which image was built from it? Which image is on AWS? How do we know the running version? How would we roll back? Without these links, “deployment succeeded” is difficult to prove.

## 9. Database migrations in a release

Schema changes must be applied in a deliberate order. If the new application code expects a column or constraint that the database does not yet have, deployment can break. Conversely, changing the schema too early can affect the old application version if the migration is not backward-compatible.

The current correction branch contains migrations 002, 003, and 004. Migration 004 protects the budget-category relationship with ON DELETE RESTRICT. The CD workflow and CI check are intended to ensure PostgreSQL is healthy, migrations are applied, and only then are application services restarted.

The migration CI checkpoint passed in its test environment, but this is not proof that an existing AWS volume was migrated successfully. The team must verify a fresh database and a populated existing database. Back up before production migrations, stop deployment on migration failure, and document recovery/rollback steps.

## 10. Environment variables and secrets

Containers need configuration such as database host, database name, credentials, Flask settings, and JWT secret. These values should be supplied at runtime rather than hard-coded into the image or committed repository.

An example environment file should contain placeholders, not actual production credentials. A secret scan can detect some accidental exposures, but it cannot prove that every secret is safe or that the deployed environment has the correct configuration. If a credential was committed, rotate it.

Separate development, test, and production configuration. Never use a development fallback that silently accepts a missing production secret. Make the application fail safely if a required production setting is absent or obviously a placeholder.

## 11. CI test coverage and its limits

A workflow can pass even if some tests are not present. A frontend lint/build job proves that the configured static checks and build succeeded; it does not prove that every button works. If the frontend package has no test script, an optional test step may skip functional tests while the overall workflow still passes.

Likewise, Docker image builds prove that the images can be built, not that the running containers can communicate correctly in the deployed environment. Migration tests prove only the database state and behavior actually exercised by those tests.

Read the workflow YAML, inspect the logs, and identify exactly which commands ran. State evidence precisely: “CI built the frontend and backend images” is different from “we tested every user journey on AWS.”

## 12. Logs and troubleshooting

Container logs help diagnose startup errors, failed connections, migration failures, and application exceptions. They should not contain passwords or bearer tokens. A good operational workflow checks the status of each service, its health check, recent logs, network configuration, and environment variable names without printing secret values.

When a service fails, ask:
- Is the container running?
- Is its health check passing?
- Is the expected port listening?
- Can the backend reach PostgreSQL by the correct service hostname?
- Did the database migration complete?
- Is Nginx routing to the right upstream and path?
- Does the application have the required runtime configuration?
- Does the deployed image correspond to the tested commit?

## 13. Hard defense questions

**Q: What is the difference between an image and a container?**  
An image is the built template; a container is a running instance of it.

**Q: Why use Compose?**  
It defines and coordinates the multi-service application, including networking, environment, volumes, and health/dependency settings.

**Q: Why does a healthy container not prove the application works?**  
The process can be alive while the API, database connection, routes, or browser requests are failing. The health check itself may also test only a narrow condition.

**Q: What does a green CI pipeline prove?**  
Only that the configured checks passed for the tested commit. It does not automatically prove production deployment or complete functional coverage.

**Q: Why can latest be risky?**  
It is a moving label. The image available under that label may change, so the deployed artifact may not match the tested commit unless the process records and pins the exact artifact.

**Q: Why run migrations before restarting the new application?**  
The new code may depend on schema changes. Migration ordering and compatibility must be deliberate, and failures must stop the release safely.

**Q: Does Nginx protect the API from unauthorized users?**  
Not by itself. It routes traffic and can add transport/security controls, but Flask must enforce application authorization.

**Q: Why can initialization SQL fail to update an existing database?**  
Initialization scripts often run only when the database directory is first initialized. An existing persistent volume can retain the old schema, so explicit migrations are needed.

## 14. What to demonstrate

Show the Dockerfiles, Compose service definitions, Nginx routes, CI workflow, migration sequence, and one CI run's actual logs. Identify the tested commit SHA and distinguish build/test evidence from AWS runtime evidence. Do not claim a live migration or browser test unless it was actually performed.

## 15. Capstone alignment

The capstone explicitly calls for Docker Compose, GitHub Actions CI/CD, Docker image build and deployment, Nginx reverse proxy, application health checks, environment-based secrets, source-control branches and pull requests, and deployment documentation. This session explains how those requirements form a release process. The tutor approved AWS as the alternative to the Azure VM named in the handout.
