# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the `Parameters` in
`template.yaml`. The file is named **`cfn-parameters.json`**, without a leading
underscore.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Hostname` | Hostname from Backstage | API custom domain and certificate name. Also used to construct OAuth scope identifiers. |
| `DnsZoneId` | `/org/dns/ZoneId` | **SSM parameter name**, not the literal hosted zone ID; CloudFormation resolves it using `AWS::SSM::Parameter::Value<String>`. |
| `CognitoUserPoolArn` | `$secrets.COGNITO_USER_POOL_ARN` | Existing Cognito User Pool used by the API authorizer. |
| `Domain` | Selected catalog domain name | Application domain metadata. |
| `System` | Selected catalog system name | Application system metadata. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Repository-derived component identifier provided through the workflow environment. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Branch slug used for the API Gateway stage. |

The `$env` and `$secrets` expressions are intentionally unquoted in the starter
file. It is deployment-action input, **not strict JSON** for direct consumption
by `json.load`, `jq`, or the SAM CLI. The same applies to expressions in
`cfn-tags.json`. The workflow passes `env_json`, `vars_json`, and `secrets_json`
to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution.
Consult that action's configuration when changing expression syntax or adding
environment-specific values.

For a manual deployment, supply resolved CloudFormation parameter values through
SAM's parameter overrides; do not pass this file as though it were the AWS CLI's
parameter-list format. Do not replace references with committed credentials.

## Stack tags: `cfn-tags.json`

This file is also named without a leading underscore. It defines:

| Tag | Value |
| --- | --- |
| `org:domain` | Selected domain name |
| `org:system` | Selected system name |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

Keep these aligned with catalog metadata and your organization's inventory and
cost-allocation conventions. Add tags through the existing deployment action;
CloudFormation propagates stack tags to resources that support them.
These metadata tags are not runtime configuration or access controls.

## Deployment prerequisites

- A target AWS account and deployment region with the required IAM permissions.
  Starter handlers explicitly create DynamoDB clients in **`us-east-1`**; use
  that region or adapt all handlers and tests before deploying elsewhere.
- A Route 53 hosted zone authoritative for the hostname, with its ID stored in
  the deployment account/region's SSM parameter `/org/dns/ZoneId` (or your chosen
  parameter name). DNS validation must be able to create certificate records.
- An existing Cognito pool and OAuth app-client/resource-server configuration
  with the scopes described in [Features](01-features.md).
- GitHub Actions secrets `AWS_CICD_ACCOUNT_ID`, `DEPLOYMENT_ACCOUNT_ID`, and
  `COGNITO_USER_POOL_ARN`, available to the relevant jobs. The scaffolder only
  explicitly creates `DEPLOYMENT_ACCOUNT_ID`; configure the others through your
  organization's repository or environment settings.
- AWS OIDC trust and build/deployment roles expected by the shared
  `gha-assume-aws-credentials` action. Account IDs alone do not grant deployment
  access. Verify role names, trust restrictions, region, and artifact storage
  against the shared action's requirements.
- GitHub environment `production` settings appropriate for **both** workflows,
  including any required approvals and branch restrictions.

## SAM and runtime settings

`samconfig.toml` sets the default stack name to the component name, disables
changeset confirmation, and requests `CAPABILITY_NAMED_IAM`. It does not define
separate branch environments. Verify the final stack name used by the shared
deployment action before deploying.

`template.yaml` globals set the selected Python runtime, a five-second timeout,
128 MB memory, `DDB_TABLE_NAME` from the stack's table, and
`POWERTOOLS_SERVICE_NAME` from the stack name. Change resource settings in SAM
when workloads need different sizing or permissions.

The hostname is not branch-qualified by default. See
[CI/CD pipeline](05-ci-cd.md) before deploying multiple branches to avoid stack,
custom-domain, and DNS collisions.
