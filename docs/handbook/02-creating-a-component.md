# Creating a component via Backstage

Choose **Create** in Backstage and select **AWS Serverless CRUD API (Python)**.
The platform needs GitHub publishing, catalog registration, the template's
custom form fields/actions, and catalog entities for the choices below.

## Page 1: Component Info

Paths below are relative to the **newly generated repository root**, not the
template source repository. Catalog selections generally contribute the entity's
name rather than its display title; Account ID uses the exception noted below.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Name | Unique component and repository name. | GitHub repository name; Component and API `metadata.name`, `spec.providesApis`, and repository references in `catalog-info.yaml`; `stack_name` in `samconfig.toml`; `info.title` in `openapi.yaml`; headings in `README.md` and `docs/handbook/00-index.md`; `site_name` in `mkdocs.yml`; package metadata in `src/common/setup.py`. `Component` in `cfn-parameters.json` and `org:component` in `cfn-tags.json` resolve from the repository-name workflow variable rather than a literal form value. |
| Description | Describes the repository, component, and SAM application. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; top-level `Description` in `template.yaml`; introductory text in `README.md`. |
| Owner | Owning group selected from your groups. | Component and API `spec.owner` in `catalog-info.yaml` use `group:<selected-group-name>`; GitHub repository Settings → Custom properties → `owner` contains the selected group's entity name. |
| Domain | Catalog domain, also used for deployment metadata and tagging. | `Domain` in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; GitHub Settings → Custom properties → `domain`. There is no direct domain relationship in the generated `catalog-info.yaml`. |
| System | Catalog system filtered by the chosen domain. | Component and API `spec.system` in `catalog-info.yaml`; `System` in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; the `README.md` heading; GitHub Settings → Custom properties → `system`. |
| Environment | Catalog environment filtered by the chosen system; narrows the cloud-account picker. | Scaffolder selection only: the chosen environment is not written to a generated file. `.github/workflows/main.yml` and `.github/workflows/branch.yml` independently set `jobs.deploy.environment: production`. |
| Account ID | Catalog cloud-account resource related to the environment; its `metadata.title` must contain the AWS account ID. | GitHub Settings → Secrets and variables → Actions → repository secret `DEPLOYMENT_ACCOUNT_ID`; both workflow files reference it in deployment steps. The account ID is not written into application files. GitHub shows the secret's name but does not reveal its saved value; confirm the source catalog entity's title before creation. |

Hidden defaults are lifecycle `production`, namespace `default`, and GitHub
organization `ServerlessOpsIO`. Namespace is collected but is not currently
passed into generated catalog metadata.

Inspect lifecycle in both entities' `spec.lifecycle` in `catalog-info.yaml`.
The GitHub organization is visible in the repository URL,
`metadata.annotations.github.com/project-slug`, and the API definition's `$text`
URL in `catalog-info.yaml`. The hidden namespace value is not written as
`metadata.namespace`; it remains a scaffolder input only.

Choosing an Environment does not configure a matching GitHub environment:
the generated workflows both explicitly use `production`.

## Page 2: Software configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Hostname | Fully qualified API DNS name for TLS, the custom domain, and DNS record. | Literal `Hostname` in `cfn-parameters.json`. In `template.yaml`, `SiteCertificate.Properties.DomainName` and `RestApi.Properties.Domain.DomainName` reference this parameter; the API's `Domain.Route53` configures its DNS record. In `openapi.yaml`, `servers` and security scopes use `Fn::Sub` expressions referencing `${Hostname}`, resolved during deployment rather than replaced with a literal hostname during scaffolding. |
| Path Prefix | Prefix before the collection path, such as `/v1`; defaults to `/v1`. Include the leading slash and avoid a trailing slash. | Collection keys under `paths` in `openapi.yaml`; `resource`, `path`, and `requestContext.path` in `data/handlers/<FunctionName>/event.json`; rendered route examples in `docs/handbook/01-features.md`. It does not change `/health` or become a separate stack parameter. |
| Collection Name | Collection path segment and source/model naming input. Use a valid Python module name, for example `thing`; hyphens produce invalid imports. | Paths, schema names, and scopes in `openapi.yaml`; function logical IDs and `CodeUri` paths in `template.yaml`; `src/handlers/<FunctionName>/`, `tests/unit/handlers/<FunctionName>/`, and `data/handlers/<FunctionName>/` directory names; the model filename and `COLLECTION_NAME` constant in `src/common/common/model/<collection>.py`; collection example/schema filenames and namespaced keys in `data/common/`. |
| API Description | Describes the API separately from the component. | `info.description` in `openapi.yaml` and the API entity's `metadata.description` in `catalog-info.yaml`, not the Component entity's description. |
| Python Version | Selects 3.13 or 3.14 (default 3.14) for Lambda, local dependency metadata, and CI. | `Globals.Function.Runtime` in `template.yaml`; `[requires].python_version` in `Pipfile`; `python_requires` and Python classifier in `src/common/setup.py`; Python/pip commands in `src/common/Makefile`; `python_version` under the Python setup step in both `.github/workflows/main.yml` and `.github/workflows/branch.yml`. |

For example, prefix `/v1` and collection `thing` produce `/v1/thing` and
`/v1/thing/{id}`. These are URL paths, not DynamoDB table names.

For collection `thing`, the generated model is
`src/common/common/model/thing.py` and the create function is
`src/handlers/CreateThingItem/function.py`; matching test and mock-data
directories are `tests/unit/handlers/CreateThingItem/` and
`data/handlers/CreateThingItem/`. The template capitalizes the collection name
for class and function names, while the model module and key namespace retain
the supplied collection name.

These values are copied into multiple files at creation time, not maintained by
a single live configuration source. If you rename a collection or change the
Python version later, update all relevant locations together. For hostname
changes, also review DNS/TLS and Cognito scopes in
[Configuration](03-configuration.md) and [Features](01-features.md).

## Page 3: Copilot Cloud Agent

**Launch Copilot agent?** defaults to false. Leaving it disabled generates the
service without launching an agent.

If enabled, choose whether to initialize Spec Kit:

- With Spec Kit disabled, supply a non-empty **Agent prompt** describing the
  work to perform. After publication and catalog registration, the
  `github:copilot:agent:launch` action launches the agent.
- With Spec Kit enabled, supply a **Spec Kit constitution** describing the
  project and its principles. The scaffolder initializes Spec Kit before
  publication, then the `github:copilot:speckit:constitution` action creates
  the constitution task.

Both paths request GitHub credentials under `USER_GITHUB_TOKEN` with the
additional `read:user` scope. They require platform support and appropriate
GitHub access. Do not put secrets in the prompt or constitution.

## Review and create

Review the form summary, especially repository name, account, hostname, and
collection name, before submitting. The scaffolder then:

1. Fetches the selected owner, domain, system, and account catalog entities.
2. Copies shared files, CRUD infrastructure and handbook, starter functions and
   tests, and GitHub Actions workflows.
3. Publishes a **public** repository with `main` as the default branch and sets
   `DEPLOYMENT_ACCOUNT_ID`.
4. Registers the generated Component and OpenAPI API entity in Backstage.
5. Optionally launches Copilot.

The results link to the repository, `catalog-info.yaml`, catalog component,
and optional agent task. Default branch protection is disabled and the required
approving review count is zero in the publisher configuration; apply your
review policy before development.

## After creation

- Confirm catalog ownership and relationships.
- Complete [deployment configuration](03-configuration.md) before assuming the
  initial push can deploy successfully.
- Review [branch deployment isolation](05-ci-cd.md) before pushing additional branches.
- Open the component's **Docs** tab, or read `docs/handbook/00-index.md` in the repository.
- Adapt the model and contract using [Common tasks](07-common-tasks.md), and request
  human review of the generated handbook.
