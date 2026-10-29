# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |
| `.github/workflows/branch.yml` (`Branch`) | Push to any branch except `main` | `workflow_dispatch` |

There is no `pull_request` trigger. A push to a feature branch starts its
workflow without needing a PR. Manual dispatch is also supported; select the
intended workflow and ref carefully.

Both workflows run on `ubuntu-latest` and grant jobs `contents: read` and
`id-token: write`. The latter allows OIDC-based AWS role assumption rather
than committing long-lived AWS credentials.

## Build job

The build job executes these steps in order:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-setup-python@v1` prepares the selected Python version.
3. `pipenv run test-unit` runs unit tests with coverage reporting and a
   **95% minimum** across `src`.
4. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
5. `aws-actions/setup-sam@v3` installs SAM.
6. `sam validate --lint` validates infrastructure with CloudFormation linting.
7. `sam build` builds the deployable application.
8. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

Integration/end-to-end tests, Python lint/type-check scripts, and TechDocs
generation are not steps in these workflows. Add them explicitly if your project
requires them; do not assume the build exercises the live API.

## Deploy job

`deploy` depends on `build` and only proceeds after it succeeds. It uses the
GitHub environment named **`production`** in both workflows.

1. The workspace action runs with `checkout_artifact: true` to retrieve the
   build artifact.
2. The credential action receives both `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID` for build/deployment account access.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys with the target account ID
   and serialized workflow environment, variables, and secrets.

The shared actions implement artifact handling, credential assumptions, and
parameter/tag expression resolution. Their internals are external to this
repository. Confirm the expected IAM roles, region, artifact storage, and
stack-name behavior against the shared actions' configuration.

See [Configuration](03-configuration.md) for parameter files, tags, secrets,
DNS/Cognito prerequisites, and SAM defaults. GitHub environment protection
rules can require deployment approval or restrict which branches deploy.

## Feature branch builds and deployments

Branch support means a non-main push executes **both build and deploy**, not
just CI checks. `CodeBranch` is resolved from `$env.GITHUB_REF_SLUG_CS` and names
the API Gateway stage. That alone does not isolate an entire deployment:

- `samconfig.toml` has a component-wide default stack name.
- `cfn-parameters.json` has one fixed hostname from scaffolding.
- Both workflows select the same GitHub environment, `production`.

Before enabling branch deployments, establish and verify a platform policy:

1. Give each deployed branch a distinct stack name and hostname through the
   shared deployment action's supported configuration.
2. Ensure the branch hostname belongs to the configured hosted zone and that
   matching Cognito resource-server scopes and app-client permissions exist.
   Scope identifiers depend on the hostname.
3. Use an appropriate account/environment and OIDC trust policy for branch
   workloads, with approval rules that prevent unintended production changes.
4. Confirm the branch slug, final stack name, custom domain, and table in the
   deployment output before sending test traffic.
5. Define teardown for branch stacks and any external resources when a branch
   is no longer needed. There is no branch-deletion cleanup workflow.

Without this policy, deployments may target the same stack or collide on custom
domains and DNS records. Do not assume a branch push creates a safe preview
environment out of the box.

## Reviewing a deployment

Inspect the GitHub Actions build and deploy logs, then check the deployed
CloudFormation stack's status and resources. Smoke-test `/health` with an access
token containing its scope, then test CRUD behavior against disposable data.
Health uses a mock integration and does not establish that Lambda or DynamoDB
is healthy. Use [Observability](06-observability.md) to correlate application
requests, errors, and metrics.
