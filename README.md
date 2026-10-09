todo Summary Assistant — DevOps Assessment

## 1. Project Overview

Todo Summary Assistant is a full-stack application containerized with Docker and deployed on AWS. This repository adds DevOps automation and operational configuration around the existing application.

Assessment focus: repeatable builds and deployments, cloud hosting, database connectivity, security, monitoring, and operational recovery.

### Technology Stack

| Area | Technology |
|---|---|
| Backend | Java 21, Spring Boot, Maven |
| Frontend | React, Node.js, npm |
| Database | MySQL on Amazon RDS for the EC2 deployment |
| Containers | Docker, Docker Compose |
| CI/CD | GitHub Actions |
| AWS deployment | EC2, Systems Manager (SSM), IAM OIDC role for GitHub Actions |
| Monitoring design | Prometheus and Grafana |

## 2. Architecture

```text
Developer
   |
   | git push to main
   v
GitHub Repository
   |
   v
GitHub Actions
   |-- Backend build (Java 21 / Maven)
   |-- Frontend build (Node.js / npm)
   |
   | on successful push to main
   v
AWS IAM OIDC role
   |
   v
AWS Systems Manager Run Command
   |
   v
EC2 instance
   |-- React frontend / Nginx (port 80)
   |-- Spring Boot backend (port 8080)
   |
   v
Amazon RDS for MySQL
```

Prometheus and Grafana are assessment monitoring components. Their configuration, deployment, dashboard, and alert status should be verified separately before claiming monitoring is complete.

## 3. Repository Structure

```text
.
├── Backend/
│   └── todo-summary-assistant/
│       ├── src/
│       ├── pom.xml
│       ├── mvnw
│       └── Dockerfile
├── Frontend/
│   └── todo/
│       ├── src/
│       ├── package.json
│       ├── package-lock.json
│       └── Dockerfile
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── monitoring/
│   ├── prometheus.yml
│   └── alert-rules.yml
├── docker-compose.yml
├── .env.example
├── FAILURE_AND_ROLLBACK.md
├── MONITORING_AND_OPERATIONS.md
└── README.md
```

The list describes the intended submission structure. Keep only paths that actually exist in the repository, and update this section if your final structure differs.

## 4. Prerequisites

For local development:

- Git
- Java 21
- Node.js 22 and npm
- Docker Desktop with Docker Compose

For AWS deployment:

- An EC2 instance with Docker and Docker Compose installed
- An Amazon RDS for MySQL database reachable from EC2
- An EC2 instance profile and Systems Manager connectivity
- A GitHub Actions IAM role configured for OpenID Connect (OIDC)
- The required GitHub repository variable and AWS permissions

## 5. Environment Configuration and Secrets

Configure runtime values using environment variables or a secret store. Do not commit `.env` files containing real credentials, database passwords, API keys, webhook URLs, or AWS credentials.

The EC2 Compose configuration uses these variables:

| Variable | Purpose |
|---|---|
| `SPRING_DATASOURCE_URL` | JDBC URL for the RDS MySQL database |
| `SPRING_DATASOURCE_USERNAME` | Database username |
| `SPRING_DATASOURCE_PASSWORD` | Database password |
| `COHERE_API_KEY` | Optional integration key, if used by the application |
| `SLACK_WEBHOOK_URL` | Optional Slack webhook, if used by the application |
| `CORS_ALLOWED_ORIGINS` | Allowed frontend origin(s) |
| `AWS_ROLE_ARN` | GitHub repository variable for the GitHub Actions deployment role |

For EC2, set the database variables in a protected environment configuration or use AWS Secrets Manager / Systems Manager Parameter Store. Restrict access to the instance and secrets.

For GitHub Actions, configure `AWS_ROLE_ARN` as a repository variable. Configure OIDC trust so that only the intended repository and branch/workflow can assume the role. Do not store long-lived AWS access keys in the workflow.

The exact environment variables required for local development may differ from those used on EC2. Check `application.properties`, Docker Compose, and the application code before running the project.

## 6. Run Locally

### Backend

From the repository root:

**Windows PowerShell**

```powershell
cd Backend/todo-summary-assistant
.\mvnw.cmd -B package -DskipTests
```

**Linux/macOS**

```bash
cd Backend/todo-summary-assistant
chmod +x mvnw
./mvnw -B package -DskipTests
```

The command compiles and packages the backend while skipping tests. To run the test suite, use the Maven test/package command appropriate to the project.

### Frontend

```bash
cd Frontend/todo
npm ci
npm run build
```

The frontend build expects the environment and API configuration used by the application. Review the frontend service configuration before running it against a different backend URL.

### Run with Docker Compose

Review `docker-compose.yml` and `.env.example` first, then configure the required values locally. Start only after confirming the Compose file matches your intended local environment:

```bash
docker compose config -q
docker compose up -d --build
docker compose ps
```

The local Compose configuration may use a containerized MySQL service, while the EC2 deployment uses Amazon RDS. Do not use the local database configuration for production deployment.

## 7. Docker Design

The backend and frontend are containerized separately.

- **Backend:** built from the Spring Boot/Maven project.
- **Frontend:** built from the React project and served by Nginx.
- **Configuration:** runtime settings are supplied through environment variables.
- **Networking:** services communicate over a Docker network.
- **Persistence:** production database data resides in Amazon RDS rather than a database container.

For final assessment compliance, verify that both Dockerfiles use multi-stage builds, run as a non-root user where applicable, and that `.dockerignore` files exclude build output, local dependencies, secrets, and unnecessary files. Document the final image and security choices only after verifying the actual Dockerfiles.

## 8. CI/CD Pipeline

The GitHub Actions workflow is intended to run on pushes and pull requests targeting `main`.

### Backend build

- Checks out the repository.
- Sets up Java 21.
- Uses Maven to build the Spring Boot backend.

### Frontend build

- Checks out the repository.
- Sets up Node.js 22.
- Installs locked dependencies using `npm ci`.
- Builds the React frontend using `npm run build`.

### Deployment

On a push to `main`, after both build jobs succeed, the workflow assumes an AWS IAM role using GitHub OIDC and invokes AWS Systems Manager Run Command on the EC2 instance.

**Current limitation to resolve before claiming end-to-end CI/CD success:** a reported deployment run failed during `git pull --ff-only origin main` because EC2 had a locally modified `docker-compose.yml` and untracked monitoring files. The EC2 Compose configuration uses RDS, while the repository's local Compose configuration includes a MySQL service. These configurations must be separated and the deployment workflow updated to use the intended EC2 configuration before the pipeline can be considered fully verified.

The assessment also asks for container image tagging by commit SHA/version, publishing to a registry, and a post-deployment health check. Verify and implement these stages if they are not present in the final workflow.

### Deployment verification

After deployment, verify the containers and application:

```bash
sudo docker compose ps
curl -fsS http://localhost:8080/actuator/health
curl -fsS http://localhost/api/todos
```

Run the health check from the appropriate host or container context. If the application does not expose the actuator endpoint in the deployed configuration, use a supported health endpoint instead. A successful health endpoint alone does not prove that every user workflow works; test creating and retrieving a todo.

## 9. AWS Deployment and Security

### EC2 and RDS

- EC2 hosts the frontend and backend containers.
- The backend connects to MySQL on Amazon RDS through JDBC environment variables.
- RDS should not be publicly accessible.
- The RDS security group should allow MySQL traffic only from the EC2 security group.
- EC2 should expose only required inbound ports. Restrict SSH access to trusted addresses if SSH is enabled.
- Use an EC2 instance role for AWS access and Systems Manager.
- Use GitHub OIDC to assume the deployment role instead of hardcoded AWS access keys.

### Security checklist

- [ ] No secrets or credentials are committed.
- [ ] `.env` is ignored by Git; only safe placeholders appear in `.env.example`.
- [ ] RDS is private and its security group only allows the application tier.
- [ ] IAM permissions follow least privilege.
- [ ] GitHub OIDC trust is limited to the intended repository/ref.
- [ ] Docker images and containers are configured according to the assessment's non-root requirement.
- [ ] Only necessary ports are exposed.

## 10. Monitoring and Operations

The assessment requires Prometheus and Grafana monitoring for application and infrastructure health.

Recommended metrics:

- Request rate, HTTP error rate, and request latency
- Application health, JVM memory, and garbage collection
- EC2 CPU, memory, disk usage, and uptime
- Container restarts and uptime
- RDS availability, CPU, storage, and database connections

Important logs:

- Spring Boot application logs
- Nginx access and error logs
- Docker container logs
- GitHub Actions build/deployment logs
- AWS Systems Manager command results
- Relevant EC2/RDS and AWS audit logs

Useful alert examples:

1. Backend unavailable for a sustained evaluation period.
2. Elevated HTTP 5xx error rate.
3. Sustained high API latency.
4. Critically low disk space or available memory.
5. RDS unavailable or critically low storage.

Use thresholds and evaluation periods to avoid alerts for brief, harmless spikes. Confirm that Prometheus targets, alert rules, and Grafana dashboards are actually working before marking monitoring complete.

See `MONITORING_AND_OPERATIONS.md` for the operational design and `monitoring/` for the actual configuration files, if present.

11. Failure and Rollback

The recovery approach is to retain a known-good application image, deploy versioned images, verify health after release, and roll back to the last known-good version if a release fails. Container restart policies can help recover a stopped process but do not replace health checks or rollback procedures.

For database incidents, inspect connectivity and RDS health, use configured backups/snapshots for data recovery, and evaluate Multi-AZ failover if Multi-AZ is configured. Rotate any leaked credentials immediately and review relevant logs.

Detailed scenarios should be documented in `FAILURE_AND_ROLLBACK.md`.

12. Troubleshooting

### Git pull fails because local changes would be overwritten

Inspect the working tree before changing anything:

```bash
git status --short
git diff -- docker-compose.yml
```

Do not use destructive Git commands blindly on a running server. Keep the EC2/RDS deployment configuration separate from the local development Compose file and make the deployment process repeatable.

### Frontend returns 502 Bad Gateway

Inspect the frontend and backend containers:

```bash
sudo docker compose ps
sudo docker logs --tail 100 todo-frontend
sudo docker logs --tail 100 todo-backend
```

Check that the frontend proxy uses the backend service name and port, and that both services share the intended Docker network. If a container's IP changes and Nginx has cached an old address, restarting the frontend can refresh the connection; investigate and fix DNS resolution behavior for a durable solution.

### Backend cannot connect to RDS

Check the JDBC URL and credentials without printing secrets, confirm RDS availability, and verify security group rules and network connectivity. Do not make RDS publicly accessible as a workaround.

13. Assessment Deliverables Checklist

Update this checklist to reflect the final, verified repository state.

- [ ] Backend Dockerfile and `.dockerignore`
- [ ] Frontend Dockerfile and `.dockerignore`
- [ ] GitHub Actions workflow for build, image publication, deployment, and health verification
- [ ] AWS architecture diagram and `aws/aws-setup.md`
- [ ] `.env.example` with placeholders only
- [ ] README with reproducible instructions
- [ ] `FAILURE_AND_ROLLBACK.md`
- [ ] `MONITORING_AND_OPERATIONS.md`
- [ ] Prometheus configuration and alert rules
- [ ] Grafana dashboard JSON or screenshots
- [ ] Successful end-to-end deployment and post-deployment health check

14. Live Application and Source

- **Live application:** http://13.201.98.243/
- **GitHub repository:** https://github.com/shashank030718/todo-summary-assistant-devops

The live URL and repository are provided for assessment review. Confirm that the endpoint remains reachable and that the repository reflects the final submitted version before sharing the links.

15. Cost Management

AWS resources may incur charges. After assessment and review, stop or delete resources that are no longer needed, following any required retention or submission constraints. Take care not to delete RDS data or other resources that must be retained.
