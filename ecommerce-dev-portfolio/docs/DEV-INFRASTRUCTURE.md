# DEV Infrastructure — AWS eCommerce

## 1. Scope and goals

Host an Angular eCommerce SPA and four independently deployable FastAPI services in an isolated AWS DEV environment, with HTTPS access, managed MySQL persistence, secret retrieval, CI/CD automation and centralized logs. This document reflects implemented DEV work and calls out unverified configuration separately.

## 2. Request path

**Frontend:** Browser → Route 53 name `shop-dev.spontansolutions.com` → CloudFront distribution → private S3 origin protected with Origin Access Control (OAC) → Angular SPA.

**Backend:** Browser → `api-dev.spontansolutions.com` → HTTPS listener on ALB (ACM certificate) → path-based target group → Dockerized FastAPI service on private Ubuntu EC2.

API path families: `/api/auth`, `/api/products`, `/api/cart`, `/api/orders`. The ALB HTTP listener redirects to HTTPS. Exact listener priorities, health-check paths and target-group port settings should be exported from AWS before publishing a runbook.

## 3. Network

- DEV VPC with private application subnet(s).
- Application EC2 launched in a private subnet; a NAT gateway was configured for required outbound access.
- EC2 deployment access is through AWS Systems Manager (SSM), not direct inbound SSH from CI.
- Security groups restrict application, ALB and database connectivity (verify and document actual ingress rules before publishing).
- Containers share the Docker user-defined network `ecommerce-network`.

**Verify before publishing:** exact VPC/subnet CIDRs, AZ coverage, public/private subnet inventory, NAT topology, endpoint use, security-group rules, and whether RDS is in distinct DB subnets. Avoid implying the environment is multi-AZ HA without proof.

## 4. Compute and services

A single Ubuntu EC2 VM runs four Docker containers:

| Container | Service | Host port |
|---|---|---:|
| `ecommerce-auth` | Identity, login and registration | 8001 |
| `ecommerce-product` | Product catalog and stock | 8002 |
| `ecommerce-cart` | Customer cart | 8003 |
| `ecommerce-order` | Checkout and order history | 8004 |

Each container is started with `--network ecommerce-network`, `--restart unless-stopped`, and a service-specific environment file. Container replacement currently uses stop/remove/run, so a brief service interruption may occur. Health checks use HTTP requests to the service's `/health` endpoint where configured.

## 5. Database

Amazon RDS for MySQL hosts four logical databases:

- `ecommerce_auth`
- `ecommerce_products`
- `ecommerce_cart`
- `ecommerce_orders`

Services connect using database configuration sourced from Secrets Manager. Auth deployment uses `alembic upgrade head` before container replacement. Do not assume all other services run migrations until their CD scripts are verified.

**Verify:** backup retention, encryption, Multi-AZ setting, automated snapshots, parameter groups, least-privilege database users, and restore testing.

## 6. Secrets and IAM

- AWS Secrets Manager stores runtime credentials/configuration; an Auth secret is named `ecommerce/dev/auth`.
- The EC2 IAM role grants service-appropriate secret access; avoid placing secrets in GitHub workflow source.
- GitHub Actions uses OIDC to assume a DEV deployment IAM role, rather than storing static AWS keys.
- The deploy job issues `ssm:SendCommand` to the target EC2 host using `AWS-RunShellScript`.
- A GitHub App installation token authorizes cross-repository integration-test dispatch.
- Service `.env` files are stored on EC2 under `/opt/ecommerce/<service>/` and restricted to mode `600`.

**Security note:** Restrict IAM resources and GitHub OIDC trust conditions to the minimum needed. Review secret exposure via shell tracing, logs and Docker inspection.

## 7. Observability and operations

CloudWatch logging has been configured for application logs. Deployment workflows capture SSM command status and perform HTTP health checks. The complete monitoring/alerting stack (dashboards, alarms, incident routing, ELK/New Relic) is not claimed as implemented here.

Suggested DEV operator checks:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker network inspect ecommerce-network
docker logs --tail 100 ecommerce-auth
curl --fail http://localhost:8001/health
```

Run these on the DEV EC2 host through an approved administrative channel such as SSM Session Manager. For deployment diagnosis, inspect the SSM command invocation status and stdout/stderr in the GitHub Actions workflow.

## 8. Known DEV limitations

- Four services share a single EC2 host (single-host failure domain).
- Stop/remove/run deployment is not zero-downtime.
- Cross-repository integration tests run asynchronously after deployment; their result does not gate the source workflow.
- DEV infrastructure was initially provisioned manually; Terraform-based recreation is a future milestone.
- Full STAGING/PROD environment isolation, approvals, HA/DR and rollback remain future work.

## 9. Evidence checklist

- [ ] Redacted VPC/subnet diagram or AWS resource inventory
- [ ] Redacted ALB listener/target-group screenshot
- [ ] CloudFront distribution and private S3 origin screenshot
- [ ] Successful GitHub Actions build/deploy/trigger graph
- [ ] Successful integration-test run screenshot
- [ ] CloudWatch log group/event screenshot
- [ ] Service health check and Docker container status
- [ ] RDS database list with endpoint/credentials hidden
