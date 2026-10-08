# DEV CI/CD and Integration Testing

## Repository model

Separate repositories are used for Auth, Product, Cart, Order and shared API integration tests. Each microservice deploys independently from its `develop` branch. Integration tests are maintained in `kosaraju3333/ecommerce-integration-tests`.

## CI: build, test, scan, publish

The Auth workflow `Auth DEV CI` is representative:

1. Trigger: push to `develop` (including merges).
2. Checkout using `actions/checkout`.
3. Set up Python 3.13 and install requirements.
4. Execute unit tests with `python -m pytest -v`.
5. Run Trivy filesystem scan, severity `HIGH,CRITICAL`, with failure on applicable findings and `ignore-unfixed: true`.
6. Derive short Git commit SHA as the Docker image tag.
7. Build image, authenticate to AWS using GitHub OIDC, log in to ECR and push image.
8. Invoke reusable DEV CD workflow with `image_tag`.

**Verification needed:** The Auth image scan stage was commented out in the shared workflow. Do not claim image scanning is enforced until it is enabled and verified.

## CD: SSM-based deployment

The reusable workflow (`workflow_call`) assumes an IAM role and runs an SSM shell script on private EC2. The Auth deployment sequence is:

```text
SSM SendCommand
  → mkdir /opt/ecommerce/auth
  → read ecommerce/dev/auth from Secrets Manager
  → write auth.env and chmod 600
  → ECR Docker login and pull image:<commit-sha>
  → run Alembic migration
  → stop/remove old ecommerce-auth container
  → docker run --network ecommerce-network --restart unless-stopped
  → curl --fail http://localhost:8001/health
  → inspect SSM completion status
```

Other services follow the same general deployment pattern; their exact migration and runtime commands should be verified against their own workflows.

## Cross-repository integration tests

After successful deployment, a separate GitHub Actions job:

1. Uses `actions/create-github-app-token@v2` with an App ID and private key stored as an Actions variable/secret.
2. Gets a short-lived installation token scoped to `ecommerce-integration-tests`.
3. Calls `gh workflow run dev-integration-tests.yml --repo kosaraju3333/ecommerce-integration-tests --ref develop`.
4. The target repository runs shared API integration tests.

The test suite covers service health, authentication, product listing, cart operations and checkout. It uses a real DEV test customer/product and can modify inventory, so dedicated test fixtures and serial execution matter.

**Current behavior:** Trigger success is not the same as test success. The microservice workflow does not currently poll or fail based on the remote test run's conclusion. The integration-test repository retains the authoritative test result.

## Branch and release plan

| Branch | Intended environment | Status |
|---|---|---|
| `feature/*` | PR development and validation | Workflow varies by repo |
| `develop` | DEV | Implemented automatic deploy |
| `staging` | STAGING | Planned |
| `master` | PROD | Planned |

Manual QA approvals, code freeze, artifact promotion and production rollout policies are future phases, not current DEV capabilities.

## Operational failure cases

| Failure | Where to investigate |
|---|---|
| Unit tests fail | GitHub Actions test logs |
| Trivy blocks build | Scanner report and dependency remediation |
| ECR push fails | OIDC trust, IAM ECR permissions, repository/tag |
| SSM SendCommand denied | IAM permissions and correctly resolved instance ID |
| SSM execution fails | Command invocation stdout/stderr, EC2 IAM role, NAT/connectivity |
| DB migration fails | Alembic output, database connectivity, secrets |
| Health check fails | Container logs, port mapping, ALB target health |
| Integration tests fail | Separate integration-test repository's Actions run |

## Security notes for public repositories

Never publish GitHub App private keys, AWS credentials, JWT signing keys, DB passwords or raw `.env` files. Use placeholder account IDs and instance IDs in examples. Review screenshots and GitHub Actions logs before publishing.
