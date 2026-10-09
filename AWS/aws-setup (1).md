#AWS Setup — Todo Summary Assistant

Purpose

This document describes the intended AWS deployment for the Todo Summary Assistant DevOps assessment. It is based on the assessment requirements and the deployment details shared during project troubleshooting. Confirm the current AWS console configuration before treating every item below as implemented.

## Architecture

- **Amazon EC2:** runs the frontend and Spring Boot backend as Docker containers.
- **Amazon RDS for MySQL:** production database; the EC2 backend connects using a JDBC URL and credentials supplied at runtime.
- **GitHub Actions:** builds the backend and frontend and invokes deployment through AWS Systems Manager (SSM).
- **AWS IAM / GitHub OIDC:** allows the workflow to assume a role using short-lived credentials rather than hardcoded AWS access keys.
- **Prometheus and Grafana:** required by the assessment for metrics, dashboards, and alerts. Verify their final deployment status and configuration separately.
- See `architecture-diagram.png` for the target architecture.

## EC2 instance and application

The project has been deployed on an EC2 instance running Amazon Linux 2023. Docker and Docker Compose are used to run the application containers.

The frontend is served through Nginx on port 80. The frontend proxies API requests to the Spring Boot backend on port 8080 over the Docker network. The backend connects to the RDS MySQL endpoint.

For the deployment, use a dedicated EC2 Compose configuration that references the RDS environment variables. Do not accidentally deploy the local-development Compose file if it defines a MySQL container. Keep the local and EC2 configurations clearly separated and document which file each workflow uses.

Expected runtime variables for the EC2 application:

- `SPRING_DATASOURCE_URL`
- `SPRING_DATASOURCE_USERNAME`
- `SPRING_DATASOURCE_PASSWORD`
- `COHERE_API_KEY` (if the feature is enabled)
- `SLACK_WEBHOOK_URL` (if the feature is enabled)
- `CORS_ALLOWED_ORIGINS`

Provide these values using a protected environment file with restricted permissions, AWS Secrets Manager, or Systems Manager Parameter Store. Do not commit real values to Git.

## RDS and network security

1. Create or use an Amazon RDS for MySQL database in the appropriate VPC.
2. Keep RDS **not publicly accessible**.
3. Configure the RDS security group to allow inbound TCP port `3306` only from the EC2 application's security group.
4. Configure the EC2 security group to expose only required ports. Port `80` is needed for the web application. If SSH on port `22` is enabled, restrict it to trusted administrator IP addresses.
5. Do not expose the backend port `8080` publicly unless there is a documented requirement; prefer internal Docker networking or appropriately restricted access.
6. Confirm the EC2 subnet/routes and security group rules allow the backend to reach the RDS endpoint.
7. Store database credentials outside source control and grant access only to the required instance/application identity.

## IAM and Systems Manager

### EC2 instance role

Attach an EC2 instance profile with the permissions needed for Systems Manager management and any explicitly required AWS access. Use least privilege. Confirm that the SSM Agent is running and the instance is registered as a managed node.

### GitHub Actions OIDC role

The workflow uses the repository variable `AWS_ROLE_ARN` and AWS region `ap-south-1`. Configure the IAM role's OIDC trust policy to trust GitHub's OIDC provider and restrict the subject claim to the intended repository and branch/environment.

Grant only the permissions required by the workflow, including the appropriate SSM send-command and command-result permissions, scoped to the intended EC2 instance and document where practical. Avoid long-lived AWS access keys in GitHub secrets.

## CI/CD deployment flow

1. A push or pull request to `main` triggers the CI workflow.
2. The backend job sets up Java 21 and builds with Maven.
3. The frontend job sets up Node.js 22, runs `npm ci`, and runs `npm run build`.
4. For a push to `main`, the deployment job assumes the AWS role through OIDC.
5. The workflow sends an SSM Run Command to EC2.
6. The remote deployment updates the intended application revision and runs Docker Compose using the dedicated EC2/RDS configuration.
7. The deployment must fail if the remote command fails or the post-deployment health check fails.

**Known issue to resolve:** a previous deployment failed because `git pull --ff-only origin main` was blocked by a locally modified `docker-compose.yml` and untracked monitoring files on EC2. Keep the RDS-specific deployment configuration separate from the local MySQL Compose configuration, update the workflow to use the dedicated EC2 file, and verify a complete successful run before claiming end-to-end CI/CD is complete.

The assessment also calls for image tagging by commit SHA/version, publishing images to a registry, and a post-deployment health check. Verify these are implemented in the final workflow; do not describe them as complete if the current workflow does not do them.

## Post-deployment verification

Run checks appropriate to the actual deployment configuration:

```bash
sudo docker compose -f docker-compose.ec2.yml ps
curl -fsS http://localhost:8080/actuator/health
curl -fsS http://localhost/api/todos
```

If your final deployment uses a different Compose filename or the actuator endpoint is unavailable, adjust the commands to match the actual configuration. Verify the frontend in a browser and test creating and retrieving a todo. Review container logs if any check fails.

## Monitoring

The assessment requires Prometheus and Grafana. The intended monitoring setup should include:

- Application metrics such as request rate, error rate, latency, and backend health.
- Host metrics such as CPU, memory, disk usage, and uptime.
- Container restart and uptime signals.
- At least two useful alert rules, such as backend unavailable and sustained high latency or resource pressure.
- A Grafana dashboard JSON or screenshots as evidence.

Only mark this section complete after confirming that Prometheus targets are up, alert rules are loaded, and the dashboard is available. If monitoring is not running in the final environment, state that clearly in the submission.

## Operational security checklist

- [ ] RDS is private and accepts MySQL connections only from the EC2 security group.
- [ ] No passwords, API keys, webhook URLs, or AWS credentials are committed.
- [ ] GitHub Actions uses OIDC and short-lived credentials.
- [ ] IAM policies follow least privilege.
- [ ] SSH, if enabled, is restricted to trusted IPs.
- [ ] The backend port is not unnecessarily exposed to the public internet.
- [ ] Dockerfiles satisfy the assessment's multi-stage and non-root requirements.
- [ ] Deployment performs a health check and fails on errors.
- [ ] Backups and recovery procedures for RDS are documented.

## Cost and cleanup

AWS resources may incur charges. After assessment and review, stop or delete resources that are no longer needed, following any required retention constraints. Verify whether RDS data must be retained before deleting or modifying database resources.
