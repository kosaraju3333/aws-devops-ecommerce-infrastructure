# Resume-ready Project Description

**Project: AWS DevOps CI/CD Automation for Microservices-Based eCommerce Platform**

**Stack:** AWS EC2, VPC, ALB, ACM, Route 53, S3, CloudFront, ECR, RDS MySQL, Secrets Manager, SSM, CloudWatch, IAM/OIDC; Docker; GitHub Actions; Trivy; Python/FastAPI; Angular; Pytest; Alembic.

- Deployed an Angular eCommerce SPA through Amazon S3 and CloudFront with OAC, and four Dockerized FastAPI microservices on private EC2 behind an HTTPS Application Load Balancer with path-based routing.
- Implemented GitHub Actions CI/CD pipelines for Auth, Product, Cart and Order services, including Python unit tests, Trivy filesystem scans, commit-based Docker image tagging and Amazon ECR publishing.
- Automated deployments to private EC2 using AWS Systems Manager and GitHub OIDC role assumption; integrated Secrets Manager, database migrations and application health checks.
- Built cross-repository GitHub Actions automation using GitHub App installation tokens to trigger API integration tests after successful DEV deployments.
- Integrated Amazon RDS MySQL with service-specific logical databases and configured CloudWatch application logging for DEV troubleshooting.

**Interview-ready explanation:** “I built a working DEV environment for a microservices eCommerce application and automated the path from a merge into `develop` to a Docker image in ECR, deployment over SSM to private EC2, and a separate API integration-test workflow. I intentionally kept DEV on one EC2 instance and planned STAGING/PROD, approval gates, HA and Terraform as later phases.”

**Accuracy note:** Do not state that remote integration-test failures automatically fail the microservice CI/CD workflow; only the cross-repository trigger has been implemented.
