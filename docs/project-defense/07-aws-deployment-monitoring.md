# Session 7 — AWS Deployment, Networking, Health Checks, and Monitoring
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. What cloud deployment means

Cloud deployment means running the application on infrastructure provided through a cloud platform rather than only on a developer's personal computer. AWS provides compute, networking, identity and access controls, storage, and monitoring services that can be combined to host the application.

The capstone handout originally names an Azure virtual machine. The tutor approved AWS as an alternative, and the team chose AWS. In the defense, explain this accurately: the written handout names Azure, while the tutor-approved implementation target is AWS. Do not edit the story so it sounds as if the original handout named AWS.

A cloud VM is still a computer. It has an operating system, CPU, memory, disk, network interfaces, firewall rules, users, and running processes. Cloud infrastructure does not automatically configure the application securely. The team remains responsible for deployment configuration, updates, access control, logs, and data protection.

## 2. What an EC2 instance is

Amazon EC2 provides virtual server instances. An instance runs an operating system and can host Docker containers, a reverse proxy, and other required services. Instance type affects available CPU, memory, and network capacity. The selected size should match the workload and cost constraints; do not claim a performance benchmark without measuring it.

An EC2 instance has an identity and a network position. It may have a public IP or DNS name, private networking, and security group rules. The public endpoint is how a browser reaches the web application when that is the chosen design. The database should not be exposed to the public internet merely because the web application needs public access.

## 3. Network boundaries and security groups

A security group is a virtual firewall attached to AWS resources. It controls allowed inbound and outbound traffic according to configured rules. For a typical web deployment, inbound web traffic may be allowed to the public entry point; administrative SSH should be restricted to trusted sources or replaced by a managed access method where possible. PostgreSQL should normally accept connections only from the backend or an approved private network.

Opening a port to the whole internet can make a service reachable by anyone, not just the team. Before the defense, know which ports are open and why. Do not assume that a port is safe simply because it belongs to a familiar service.

A private database is not just a preference. It reduces the number of places an attacker can directly reach. Even if the database port is private, database credentials and permissions still need protection.

## 4. IAM and least privilege

AWS Identity and Access Management (IAM) controls which identities can perform which actions on AWS resources. An IAM user, role, and EC2 instance role are not interchangeable concepts, and the correct choice depends on the deployment design.

Least privilege means granting only the permissions needed for the task. A deployment workflow that needs to publish one image should not automatically receive broad administrative access to the entire AWS account. Credentials should be stored in protected secret configuration and rotated if exposed.

The capstone asks for appropriate cloud IAM permissions. Be ready to explain which identity deploys the application, which identity the running server uses if applicable, and why those permissions are limited. If the current setup uses a simpler credential-based process, describe it truthfully and identify what would be improved.

## 5. Public endpoint and reverse proxy

The browser should access the application's public web endpoint, not a private PostgreSQL port. Nginx can serve frontend assets and route API requests to Flask. Flask communicates with PostgreSQL through the intended private service connection.

A public IP address or DNS name is not proof that the application is healthy. A TCP connection may work while Nginx routes the wrong path, Flask is unavailable, or the database is unreachable. Test the real public URL and the health endpoints through the same route users use.

The capstone asks for a live URL, and it says a server IP can be used rather than a custom domain. The team should supply the current working endpoint and a demonstration plan. Do not invent a URL or claim that it is live if it has not been checked.

## 6. Environment configuration on AWS

The application needs configuration for database access, Flask behavior, JWT signing, and other runtime settings. These values should be supplied to the running services securely. They should not be hard-coded into source files or baked into a public image.

The code may work locally because a developer's environment contains variables that are absent on AWS. A production deployment should fail clearly and safely when a required secret is missing, rather than silently using a weak fallback. To verify configuration, check variable names and whether values are present without printing the values themselves.

The project correction addressed a JWT environment-variable naming mismatch. This is an example of why configuration must be checked across local development, CI, Docker, and AWS. Passing tests with one variable name does not prove that the running service uses the intended secret.

## 7. Deploying an exact version

A reliable release should identify the code commit and the exact image deployed. A tag such as latest can move to a different image after another build. If the deployment pulls latest without pinning or recording the image digest, the server may run a version different from the one CI tested.

A good release record includes:
- the Git commit SHA;
- the image tag or digest built from that commit;
- the deployment time and environment;
- the result of migrations;
- the running application version;
- health-check results;
- rollback instructions and the previous known-good version.

If a deployment also pulls the Git repository on the server, the team must explain how that source version relates to the container image. Mixing a source checkout from one commit with an image from another can make troubleshooting confusing.

## 8. Database persistence and migrations on AWS

Containers may be replaced during deployment. Persistent database data must not depend only on a disposable container layer. Depending on the architecture, the database may use a persistent volume or a managed database service. The team must know which design the project actually uses.

Migrations need special care on an existing database. Initialization SQL often runs only when a database volume is first initialized; restarting a container with an existing volume does not guarantee the SQL will run again. The current correction workflow applies migrations 002–004 before restarting the application services. Migration 004 protects the category-budget relationship.

The CI migration job passed for the recorded correction commit, but CI is a separate environment. It does not prove a populated AWS database has been updated. The team must collect deployment logs and verify the schema on AWS. Before a real production migration, back up the data and know how to recover if the migration fails.

## 9. Health checks versus monitoring

A health endpoint gives a quick signal about a service. A simple application health endpoint might answer whether the API process is responding. A database health endpoint can test whether the application can reach PostgreSQL. These endpoints should avoid exposing passwords, connection strings, or detailed internal error messages.

Health is not the same as full monitoring. A single successful request does not show long-term availability, CPU and memory pressure, disk exhaustion, error rates, or unusual login activity. Monitoring may include service metrics, container status, application logs, cloud metrics, alerts, and deployment records.

The team should distinguish:
- **Liveness:** Is the process running and able to respond?
- **Readiness:** Is it ready to serve normal requests, including required dependencies?
- **Dependency health:** Can it reach services such as PostgreSQL?
- **Monitoring:** Can the team observe trends and be alerted to failures over time?

An overly deep health check can mark the whole service unhealthy during a brief database problem; an overly shallow check can report healthy while the app cannot serve meaningful requests. Choose and document the intended meaning.

## 10. Logs and incident response

Logs help diagnose failures. For example, if the public health route fails, the team can check the Nginx response, backend logs, database connection errors, and container status. Logs should be available to authorized maintainers and must not reveal tokens or secrets.

An incident procedure should answer: how do we identify the affected version, check service health, inspect recent changes, restore the previous version, protect database state, and tell the team what happened? A rollback of application code does not automatically roll back a destructive database migration, so the plan must account for schema compatibility and data recovery.

## 11. Cost and resource awareness

Cloud resources can incur costs even when the application has little traffic. Instance type, storage, public IPv4 usage, data transfer, and other services may affect the bill. Stop or remove temporary resources when no longer needed, while protecting persistent data and following team policy.

The defense should not claim the deployment is “free” unless all applicable charges and credits are understood. Explain the selected resources and the reason for their size rather than inventing cost or performance numbers.

## 12. Hard defense questions

**Q: Why use AWS when the PDF says Azure?**  
The handout names Azure VM deployment, but the tutor approved AWS as an alternative. The team used AWS under that approval. The written requirement and approved implementation choice should be described honestly.

**Q: What is an EC2 instance?**  
A virtual server in AWS that provides compute resources and a network environment for running the application.

**Q: Why not open PostgreSQL to the internet?**  
The browser does not need direct database access. Keeping PostgreSQL private reduces the attack surface; Flask is the intended data access boundary.

**Q: What is a security group?**  
An AWS virtual firewall that controls network traffic to and from associated resources.

**Q: What is least privilege?**  
Give each identity only the permissions it needs, reducing damage if a credential is misused.

**Q: Does a public IP mean the app is deployed successfully?**  
No. It only identifies a network endpoint. The actual application routes, API, database connection, health checks, and user flows must be tested.

**Q: How do you know which version is live?**  
Tie the running image tag/digest to the source commit and deployment record, then verify the deployed service. Do not rely on a moving tag alone.

**Q: What happens if migration 004 fails?**  
The release should stop before restarting incompatible application code, preserve diagnostic logs, protect the existing data, and follow the documented recovery plan. Never improvise a destructive rollback on production data.

**Q: Is a health endpoint the same as monitoring?**  
No. Health is a point-in-time signal; monitoring tracks service behavior and trends over time and can alert the team.

## 13. Evidence checklist for AWS claims

Before saying a deployment is verified, gather the current public URL, timestamped health responses through Nginx, container/service status, logs showing the tested image/version, migration result on the target database, proof that required secrets are configured without exposing their values, security group rules, and a successful browser journey. If any item is unavailable, label it as pending rather than implying it passed.

## 14. Capstone alignment

The handout asks for a live URL, architecture diagram, cloud infrastructure, health checks, logs, basic server/container monitoring, secure environment variables, appropriate IAM, Docker image build and deployment, and deployment instructions. The tutor approved AWS as the alternative to the Azure VM named in the original PDF. The presentation should explain the actual AWS resources used and show evidence, not merely describe what AWS could do.
