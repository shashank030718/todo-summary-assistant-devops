Failure and Rollback Strategy

1. A faulty version is deployed. How do you roll back?

I would deploy versioned Docker images tagged with the Git commit SHA rather than relying only on the `latest` tag. If a deployment introduces a problem, I would identify the last known healthy version, redeploy that image, and verify the application health endpoint and key functionality.

I would also review deployment logs and application logs to identify the root cause. Database schema changes would need a separate rollback or forward-fix plan because reverting an application image alone may not reverse database changes.

2. The application crashes after deployment. What happens?

Docker's `restart: unless-stopped` policy attempts to restart a stopped application container. I would check container status, application logs, resource usage, database connectivity, and the health endpoint.

If the application remains unhealthy, I would roll back to the last known healthy version and investigate the issue before attempting another deployment. A container restart policy does not replace application health checks or deployment verification.

3. The CI/CD tool is unavailable. Can you still deploy?

Yes. I can use AWS Systems Manager Session Manager or Run Command to access the EC2 instance without relying on SSH access from the CI/CD runner. I would deploy a previously built and verified image from the container registry, using the appropriate configuration and secrets.

Manual deployment would be a documented break-glass procedure. Afterward, I would record the deployed version and investigate the CI/CD outage.

4. Secrets are leaked. What steps do you take?

I would immediately revoke or rotate the exposed credentials, including database passwords, API keys, webhook URLs, or cloud credentials as applicable. I would check access logs and audit events for unauthorized activity.

I would remove secrets from the active configuration and store them in AWS Secrets Manager or Systems Manager Parameter Store, with access restricted through IAM. I would also review repository history and pipeline logs, remove exposed secrets where appropriate, and prevent future secret commits.

Deleting a secret from a repository does not invalidate it; rotation is essential.

5. The EC2 instance fails. How do you recover?

I would inspect the EC2 instance status checks, system logs, and AWS monitoring information. If recovery is not possible, I would launch or recover an instance using the documented deployment procedure, install or configure Docker, and deploy the known-good application version.

I would restore the required IAM permissions, security group configuration, application environment variables, and connectivity to RDS. I would then verify application health and database access.

Infrastructure as Code and documented configuration would make recovery more repeatable.

6. The RDS database becomes unavailable. What is the impact and recovery plan?

The application may fail to read or write todos while the database is unavailable. I would check RDS status, connectivity, security groups, database logs, and application connection-pool errors.

I would restore database availability first. If the issue is an infrastructure failure, Multi-AZ failover may help if Multi-AZ is configured. If data is corrupted or lost, I would evaluate restoring from automated backups or a snapshot to a recovery point.

After recovery, I would verify database connectivity, data integrity, and application functionality. I would also review backup retention, recovery objectives, and whether Multi-AZ is appropriate for the workload.

Deployment and Rollback Principles

- Build and tag Docker images with immutable Git commit identifiers.
- Deploy only versions that pass the CI checks.
- Verify application health after deployment.
- Preserve a known-good image for rollback.
- Keep database credentials outside source control.
- Document recovery steps and test them periodically.