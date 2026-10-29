# Common changes and tasks

## Set up a development environment

Install the project's selected Python version, Pipenv, and the AWS SAM CLI.
From the **generated repository root**, run:

```sh
pipenv install --dev
pipenv run test-unit
sam validate --lint
sam build
```

`test-unit` is the existing Pipfile script with a 95% coverage requirement across
`src`. SAM validation/build require the deployment tooling and relevant AWS
configuration. Unit tests use Moto, not a deployed table.
Use synthetic data and mocked credentials for tests.

## Adapt the data model

1. Add business fields to
   `src/common/common/model/thing.py`, starting with
   `BaseThingData`. Keep dataclass required/default
   field ordering valid across inheritance.
2. Update handler parsing, persistence, and returned fields. OpenAPI's
   `additionalProperties: true` does not make the Python dataclass accept
   arbitrary fields.
3. Update OpenAPI request/response schemas, including required fields and
   status codes. Address the starter contract differences listed in
   [Features](01-features.md).
4. Update common and per-handler examples/schemas and their tests.
5. If you change key format, update `create_keys()`, `get_keys_from_id()`, and
   `get_id_from_keys()` together and plan migration of existing records.

## Add an endpoint

1. Decide the client method, path, inputs, outputs, and required OAuth scope.
   Use a new function if the operation needs its own implementation; follow
   the next section first.
2. Add the path/operation in `openapi.yaml`. Define path/header parameters,
   request-body schema, documented success/error responses, and a
   `serverlessOpsCognitoPool` security requirement. Do not omit authentication
   accidentally.
3. For a Lambda route, copy an existing `x-amazon-apigateway-integration`:
   retain `type: AWS_PROXY` and integration `httpMethod: POST`, and change the
   `Fn::Sub` invocation URI to reference the intended SAM function's ARN.
   The client method remains the operation key (`get`, `post`, and so on).
4. Ensure `template.yaml` defines that function and a corresponding
   `AWS::Lambda::Permission` allowing `apigateway.amazonaws.com` to invoke it.
   Do not add a duplicate SAM `Events` route: this API is defined by OpenAPI.
5. If introducing a scope, configure it on the existing Cognito resource server
   and app client. Add application-level ownership/tenant checks where required.
6. Update event fixtures for method, path, parameters, and body, then add handler
   tests for success, invalid input, missing items, and authorization logic.
7. Run local checks below; after a safely isolated deployment, call the endpoint
   with valid and invalid tokens and verify the documented status/body.

The existing `/health` mock is not a substitute for exercising a new Lambda route.

## Add a function

### Function code and infrastructure

1. Create `src/handlers/<FunctionName>/` using the closest starter handler as a
   pattern, with `__init__.py`, `function.py`, and `requirements.txt`.
   Use a valid Python package name.
2. Keep `handler(event, context)` as the entry point and separate business
   logic into `_main` or shared modules. For API handlers, use the existing
   Powertools event wrapper, structured logger, output dataclass, and response
   conversion pattern described in [Architecture](04-architecture.md).
3. Add an `AWS::Serverless::Function` to `template.yaml` with its `CodeUri` and
   `Handler: function.handler`. It inherits the runtime, timeout, memory, and
   table environment variable from Globals unless overridden.
4. Grant only the permissions the function needs. Existing CRUD functions use
   `DynamoDBCrudPolicy`; a read-only operation should use a narrower policy.
5. For API invocation, add a Lambda invoke permission and the OpenAPI route.
   For other triggers, define the appropriate event-source resources and
   permissions instead.

The common package has a Makefile for a layer build, but this CRUD SAM template
does not declare a `CommonLayer` resource. Do not assume a new function inherits
a deployed shared-code layer.

### Dependencies

- Local development dependencies are managed in `Pipfile`. Use
  `pipenv install "<package>==<version>"` for runtime dependencies or
  `pipenv install --dev "<package>==<version>"` for test/development tools.
  Review advisories and choose a supported version before adding a package.
- SAM packages each function using its own
  `src/handlers/<FunctionName>/requirements.txt`. Add runtime dependencies there
  too; a Pipfile entry alone does not ensure they are packaged.
- Existing handler requirements reference the local common package with
  `-e src/common/` and include Powertools. Follow the existing convention and
  check `sam build` actually includes the dependencies; local editable installs
  are not proof that a deployment artifact is complete.
- Declare dependencies of shared code in `src/common/setup.py` as appropriate.
  Keep versions consistent across local and deployed environments, and review
  dependency/lockfile changes.

### Mock data and schemas

Create `data/handlers/<FunctionName>/` with the same four example/schema pairs
as the existing functions:

- `event.json` / `event.schema.json`: invocation envelope.
- `data.json` / `data.schema.json`: business input.
- `response.json` / `response.schema.json`: decoded response body.
- `output.json` / `output.schema.json`: proxy status and string body.

Update reusable examples and schemas in `data/common/` when the model changes.
Keep API examples, dataclasses, and stored-item examples consistent. Use fake
identifiers and no real user records or tokens. Update `file:` schema references
if you relocate files.

### Tests

Create `tests/unit/handlers/<FunctionName>/` with `__init__.py` and
`test_function.py`, following the nearest existing test:

1. Load fixtures from the matching `data/handlers/` directory.
2. Use `lambda_function_context` for invocation context.
3. Use Moto AWS mocking and patch the module's table/client reference before
   executing business logic. Do not let tests contact a real AWS account.
4. Make generated IDs deterministic and assert database state, proxy status,
   and decoded response.
5. Cover malformed input, conditional-write failures, missing records, and
   relevant application authorization checks.
6. Explicitly validate fixture instances against schemas. Constructing a
   `Draft7Validator` alone does not validate an instance; some starter
   `test_validate_*` functions only construct one. For a self-contained schema:

   ```python
   jsonschema.Draft7Validator.check_schema(schema)
   jsonschema.Draft7Validator(schema).validate(instance)
   ```

   Referenced schemas additionally need reference resolution for the
   `file:./data/common/...` convention. Resolve references using the installed
   JSON Schema library's supported mechanism and test that invalid fixtures fail.

## Validate and deploy changes

Run a focused test first, substituting your new function's name:

```sh
pipenv run pytest -vv tests/unit/handlers/<FunctionName>/test_function.py
```

Then run `pipenv run test-unit`, `sam validate --lint`, and `sam build`.
Existing optional scripts `pipenv run pylint` and `pipenv run mypy` run Python
checks; the workflows do not run them automatically.

Review the API contract and packaged dependencies, commit the related code,
schemas, tests, and handbook updates, and open a PR under your review policy.
Remember that **pushing a feature branch starts deployment**: complete the
[branch-isolation setup](05-ci-cd.md) before pushing.

After deployment, smoke-test authentication and behavior with disposable data,
inspect logs/metrics, and obtain human review. Unit tests cannot prove that DNS,
API Gateway authorization, IAM, or the deployed dependency bundle works.
