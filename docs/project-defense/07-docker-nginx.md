# Session 7 — Docker Compose, Nginx, Networking, and Health Checks

## 1. Why containerise an application?

A web application depends on more than source code. It needs a compatible runtime, libraries, environment variables, network access, and sometimes operating-system packages. If each developer configures these differently, the app may work on one machine and fail on another.

A Docker image packages an application and the files/runtime it needs. A container is a running instance of that image. Containers make the runtime more repeatable, but they do not automatically make the application secure, scalable, or correct.

A container is also not the same thing as a virtual machine. A VM provides a virtualised operating-system environment. Containers share the host kernel and isolate application processes through operating-system mechanisms. In this capstone, AWS EC2 provides the VM, and Docker runs the application's containers on that VM.

## 2. Image versus container

An **image** is the packaged template built from a Dockerfile. A **container** is a running instance created from an image.

A frontend image can contain the built React assets and a web server to serve them. A backend image can contain Python, Flask, and required packages. PostgreSQL is usually run from an official database image. Nginx can run from its own image/configuration.

Why separate images? Each service has a different responsibility and runtime. The backend can be rebuilt without putting Python into the frontend container, and the database can use its maintained database image. The services still need compatible configuration and communication paths.

An image build passing proves the image can be built under that build environment. It does not prove the container can start with production secrets, connect to the database, or serve a correct live application.

## 3. What Docker Compose does

Docker Compose defines how multiple containers work together. A Compose file can specify services, image/build context, environment variables, ports, networks, volumes, health checks, and dependency conditions.

In Expense Tracker, the main services are frontend, backend, PostgreSQL, and Nginx.

- **frontend:** serves the React build.
- **backend:** runs Flask and connects to PostgreSQL.
- **postgres:** stores persistent application data.
- **nginx:** exposes the public web port and routes requests to frontend or backend.

Compose provides internal service discovery. Containers on the Compose network can normally reach a service using its service name, such as backend or postgres. Those names are for container-to-container traffic; a browser on a user's laptop should not call http://backend:5000 directly.

## 4. Ports and network exposure

A container port is the port on which a service listens inside its container. A published port maps a host port to a container port. For example, publishing host port 80 to Nginx makes the public web service reachable through the VM's port 80, subject to AWS security-group and host firewall rules.

Only necessary ports should be published. The browser needs the public web entry point; it generally does not need direct access to Flask or PostgreSQL. Publishing PostgreSQL to the public internet increases risk and should be avoided unless there is a carefully controlled, explicit operational reason.

Docker's internal network is not a replacement for application authentication. A compromised service may still reach other services it can network to. Use least privilege, restrict host ports, protect credentials, and avoid running unnecessary services.

## 5. Persistent volumes: why database data must survive

Containers can be replaced. A database's records must not disappear simply because its container is recreated.

A Docker volume stores data outside the container's writable layer. The PostgreSQL service uses a named volume for its data directory. When the container is recreated with the same volume, PostgreSQL can retain its data.

This creates an important distinction:

- Recreating a container changes the running instance.
- Rebuilding an image changes the packaged application.
- Deleting a persistent volume can delete the stored database data.

Do not run destructive volume commands on production without understanding their effect and having a backup/recovery plan. A volume is not itself a backup; it may still be lost if the host or storage is damaged.

## 6. Environment variables and secrets

Services need configuration: database name, host, username, password, secret keys, environment mode, and sometimes API base URLs. Environment variables allow the same image to run in different environments with different configuration.

But environment variables are not automatically secure. They can leak through repository files, shell history, process inspection, logs, or misconfigured deployment tooling. Never commit a real .env file or put production credentials into a Dockerfile or frontend build.

The reviewed main Compose file contains fixed PostgreSQL credentials in the service configuration. This must be reconciled before claiming production secrets are fully managed. Use a local example file with harmless placeholders, provide real values outside Git, restrict access, and rotate credentials if they were used in an exposed environment. The backend must receive the same intended database credentials that PostgreSQL was configured with.

## 7. Startup order and health checks

A dependency rule can control startup order, but “container started” does not always mean “service ready.”

PostgreSQL may take time to initialise or recover. A backend that connects before the database is ready can fail. Compose health checks can test whether PostgreSQL accepts connections and can mark it healthy. A dependency condition can then delay backend startup until the database is healthy.

A health check is only as useful as its test command. For example, a PostgreSQL readiness command checks whether the server is accepting connections. A backend /health route may prove that Flask responds, but if it does not query PostgreSQL it does not prove database connectivity. The application can provide separate liveness and readiness checks when appropriate.

- **Liveness:** is the process running and able to respond?
- **Readiness:** can the service handle real work, including required dependencies?
- **Dependency health:** is a dependency such as PostgreSQL accepting requests?

Do not assume a restart policy or health check repairs every failure. A container may keep restarting because of bad configuration. Logs and clear failure behaviour are still needed.

## 8. Nginx as reverse proxy

A reverse proxy receives a request from the client and forwards it to the appropriate upstream service. In this project, Nginx can serve the frontend for normal page paths and forward /api requests to Flask.

The intended public routing is:

- Browser requests the application page → Nginx → frontend.
- Browser requests /api/expenses → Nginx → Flask backend.
- Flask queries PostgreSQL over the internal network.
- The response returns through Flask and Nginx to the browser.

This pattern gives the browser one public origin and hides internal service names. It can simplify configuration and reduce the need to expose Flask directly.

Nginx routing is based on matching rules, so exact paths matter. The reviewed main configuration explicitly maps /api/health to the backend /health route and then has a general /api/ rule. That does not automatically prove that every desired health path, such as /api/health/db, maps correctly. Test the exact public path through Nginx.

## 9. What proxy headers do

Nginx commonly forwards headers such as Host, X-Real-IP, X-Forwarded-For, and X-Forwarded-Proto. They provide information about the original request and client address/protocol.

The backend must only trust forwarded headers when they come through a trusted proxy configuration. If the application trusts arbitrary client-supplied forwarded headers, a caller may spoof information used for logging, URL generation, or security decisions. Rate limiting behind a proxy also depends on correct client-IP handling.

## 10. Docker Compose is not CI/CD

Compose describes how to run the services together in an environment. CI/CD describes how code changes are checked, packaged, published, and deployed.

A typical flow is:

1. Developer pushes a branch or opens a pull request.
2. GitHub Actions checks source code and tests.
3. Docker builds images.
4. After the required checks and merge conditions, a publish workflow creates image tags.
5. The deployment process pulls the intended version on AWS EC2.
6. Containers are restarted or updated.
7. Health checks and browser tests confirm the running service.

The exact flow must match the repository's workflow. A Dockerfile alone does not publish an image. A successful image push does not prove the VM pulled it. A deployment command does not prove the app is healthy.

## 11. Migration and persistent-volume traps

PostgreSQL container initialisation scripts in the official image normally run when the database data directory is first initialised. If a named volume already contains a database, adding another SQL file to the init directory does not cause that script to run automatically.

That is why existing databases need an explicit migration mechanism. The deployment should wait for PostgreSQL readiness, apply migrations in order, stop if a migration fails, and only then restart the application as intended. Test on both a fresh database and a copy or safe staging instance with existing data.

The reviewed main Compose file mounts migration 001 in the PostgreSQL initialisation directory. Later migrations in the repository need an explicit application path; do not assume they run just because their SQL files exist. A newer correction branch proposes applying migrations during deployment. Verify which branch and workflow are actually deployed before describing this as complete.

## 12. Logs and troubleshooting

Each container should be diagnosable through its logs. When the site is unavailable, check in a logical order:

1. Is the AWS VM reachable?
2. Is the EC2 security group allowing the required public port?
3. Are the containers running?
4. Are their health checks passing?
5. Can Nginx reach the frontend and backend services?
6. Can Flask reach PostgreSQL?
7. Are environment variables present and consistent?
8. Did a migration fail?
9. Is the correct image/version running?
10. Does the public route return the expected response?

Avoid debugging by repeatedly restarting everything without recording evidence. Capture timestamps and relevant logs, redact secrets, identify the failure layer, and make one controlled change at a time.

## 13. Connections to the other sections

- **Frontend:** built static assets are served by the frontend container and reached through Nginx.
- **Backend:** Flask runs in a container and uses the internal database network.
- **Database:** the PostgreSQL volume preserves records beyond container replacement.
- **Security:** secrets, ports, proxy trust, and access boundaries must be configured correctly.
- **Testing:** build tests, health checks, migration checks, and end-to-end tests cover different risks.
- **CI/CD and AWS:** images are built/published and a specific version is deployed to EC2.

## 14. Difficult defence questions and answers

**Q1. What is the difference between a Docker image and a container?**  
An image is a packaged template. A container is a running instance created from that image.

**Q2. Why does PostgreSQL need a volume?**  
Containers can be replaced. A named volume stores database files outside the container's writable layer so the data can survive container recreation. It is not a substitute for backup.

**Q3. Why not expose port 5432 publicly?**  
The browser and public users do not need direct database access. Keeping PostgreSQL internal reduces the attack surface and forces requests through the backend's authentication and business rules.

**Q4. What is a reverse proxy?**  
It receives client requests and forwards them to the appropriate upstream service. Here Nginx routes normal web requests to the frontend and API requests to Flask.

**Q5. Does depends_on guarantee that the database is ready?**  
A basic startup dependency may only establish ordering. A health-based dependency can wait for a defined readiness check. The check itself must meaningfully verify the required state.

**Q6. What is the difference between a liveness and readiness check?**  
Liveness asks whether the process is alive. Readiness asks whether it can handle requests, including required dependencies. A live Flask process can be unready if PostgreSQL is unavailable.

**Q7. Why do migrations not rerun automatically on an existing volume?**  
Database image initialisation scripts normally run only when the data directory is first created. Existing data volumes need an explicit, versioned migration process.

**Q8. Does a successful Docker build prove the app runs?**  
No. It proves the image built. Runtime configuration, ports, secrets, dependency readiness, migrations, and application behaviour still need testing.

**Q9. Why use service names like backend and postgres?**  
Compose provides internal DNS for services on its network. Containers can use service names, while the user's browser should use the public host and /api path.

**Q10. If Nginx is public, does it make the application secure?**  
No. It is a traffic-routing boundary. Flask still needs authentication, validation, and authorisation; secrets and database permissions must also be protected.

**Q11. Why can a health endpoint pass while the app is broken?**  
It may only prove that one process responds. It may not check PostgreSQL, login, expense creation, summaries, or CSV export. Use health checks and user-journey tests together.

**Q12. What happens if a migration fails halfway through deployment?**  
The workflow should fail visibly and avoid restarting the application against an incompatible schema. The team needs a backup/recovery strategy and evidence from staging that the failure path is safe.

**Q13. Why are fixed database credentials a problem if the repository is only for a school project?**  
A public repository can be copied or scanned, and development credentials can accidentally be reused in production. Environment-based configuration and least privilege are good practice even in a capstone.

**Q14. What should you do if the browser can reach Nginx but API calls fail?**  
Inspect the browser request path and status, Nginx location rules and logs, backend container health, service networking, and Flask logs. Test through the same public URL used by the browser.

## 15. Docker and Nginx defence checklist

Be able to explain each Compose service, each published port, the database volume, environment variables, health checks, the internal network, and the Nginx route rules. Demonstrate the public health path and an API request. Know which migrations are automatically initialised and which must be explicitly applied. Never claim a secret or deployment configuration is secure without inspecting the active configuration.
