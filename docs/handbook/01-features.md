# Features

## OpenAPI and API Gateway

`openapi.yaml` defines the API contract and its API Gateway integrations.
`template.yaml` includes it through `AWS::Include` in `RestApi.DefinitionBody`,
so editing it changes both the published API definition and deployed routes.
The Backstage API entity also references this file.

| Method and path | Implementation |
| --- | --- |
| `GET /health` | API Gateway mock integration; no Lambda |
| `POST /v1/thing` | `CreateThingItemFunction` |
| `GET /v1/thing/{id}` | `GetThingItemFunction` |
| `PUT /v1/thing/{id}` | `UpsertThingItemFunction` |
| `DELETE /v1/thing/{id}` | `DeleteThingItemFunction` |

Lambda routes use proxy integrations. The API Gateway integration method is
`POST` even for client-facing `GET`, `PUT`, and `DELETE` operations because it
invokes Lambda. A request validator enables body and parameter validation.
Response declarations describe the contract; they do not validate Lambda outputs.
The file includes CloudFormation `Ref` and `Fn::Sub` expressions, so it is a
deployment-aware definition rather than a fully resolved standalone OpenAPI file.

## Starter code, tests, and data

Each CRUD operation has a handler, deployment dependency file, unit tests, mock
events and responses, and JSON schemas. A shared dataclass model and key helpers
live in `src/common/common/model/thing.py`.

The starter model has an `id` but no domain-specific fields. Adapt the model,
OpenAPI schemas, mock data, and tests together before exposing a real contract.
Create generates a UUID; read retrieves an item; delete removes an existing
item. Despite its name, the upsert handler's conditional update requires an
existing item.

The starter contract needs review: POST's request schema requires `id` even
though create generates it; its response schema describes `request_id` but
requires `id`, while the handler returns `id`. PUT declares `200` but the
handler returns `201`. Resolve these differences when adapting the API.

See [Architecture](04-architecture.md) for code and fixture anatomy and
[Common tasks](07-common-tasks.md) for customization steps.

## Pipenv for Python virtualenv management

Pipenv manages an isolated Python virtual environment for local development,
keeping this project's dependencies separate from other projects and your
system Python. The generated `Pipfile` specifies the selected Python version,
runtime packages, development tools, and named test/check scripts. It installs
the local `common` package in editable mode so shared-code changes are available
without reinstalling that package.

From the generated repository root, use `pipenv install --dev` to create the
environment and install runtime and development dependencies. Run commands
inside it with `pipenv run`, such as `pipenv run test-unit`, or enter an
interactive environment with `pipenv shell`. Install the selected Python version
and Pipenv first.

Pipenv records resolved dependency versions in `Pipfile.lock`; keep the lockfile
under version control for repeatable installs. This development environment is
separate from Lambda deployment packaging: SAM uses each function's
`requirements.txt`, so Pipfile changes alone do not update deployed dependencies.
See [Common tasks](07-common-tasks.md#dependencies) for dependency management.

## CI/CD and feature branches

GitHub Actions supplies separate main and non-main branch workflows. Both run
unit tests, validate and build with SAM, and deploy after a successful build.
Branch deployments need deliberate stack and hostname isolation; both workflows
currently select the `production` GitHub environment.
See [CI/CD pipeline](05-ci-cd.md) before pushing a feature branch.

## DNS and TLS

The stack creates a DNS-validated ACM certificate, a regional API Gateway
custom domain, and a Route 53 record for the configured hostname. It obtains
the hosted zone ID through an existing SSM parameter. The hosted zone and SSM
parameter are prerequisites, not resources created by this service.
See [Configuration](03-configuration.md).

## Cognito authentication and authorization

The API uses an existing Cognito User Pool through `CognitoUserPoolArn`; it does
not create a pool, resource server, app client, or users.
The `serverlessOpsCognitoPool` OpenAPI security scheme reads the `Authorization`
header. Every starter operation, including `/health`, has a scope requirement:

| Operation | Required scope |
| --- | --- |
| Health | `https://<hostname>/health.read` |
| Read item | `https://<hostname>/thing.read` |
| Create, update, delete | `https://<hostname>/thing.write` |

The hostname here is the deployed `Hostname` parameter. Configure matching
resource-server scopes and allow them on the Cognito app client. Obtain an
**access token** containing the required scope using the client's configured
OAuth flow, then send it in the `Authorization` header. Machine clients may use
client credentials if enabled; browser clients should use authorization code
with PKCE and must not embed a client secret.

API Gateway verifies the token and operation scope. This does not provide
per-item ownership or tenant authorization: add those checks in application
logic if needed. Do not commit tokens or app-client secrets.
