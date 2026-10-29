# Observability

## Function logging

Handlers use AWS Lambda Powertools `Logger(utc=True)` and
`@LOGGER.inject_lambda_context` for structured logs with invocation context.
`POWERTOOLS_SERVICE_NAME` is set to the CloudFormation stack name.
CloudWatch receives Lambda logs in `/aws/lambda/<physical-function-name>`;
find the physical name in the stack resources or Lambda console.

Get and Delete log the full incoming event at **INFO**, so those entries are
emitted at the logger's default INFO level. Create and Upsert log events at
**DEBUG**, and all four handlers log outgoing proxy outputs at DEBUG.
For temporary debugging, configure the Powertools `LOG_LEVEL` environment
variable through SAM and redeploy, after sanitizing the logged data.
Add meaningful, sanitized business logs as the service evolves.

Full API events can contain authorization headers, personal information, and
request bodies. Redact or remove full-event logging **before production use**,
including the existing INFO calls, and sanitize outputs before enabling DEBUG.
Never log tokens, credentials, or app-client secrets.
The template does not specify Lambda log-group retention; set an appropriate
retention policy for your service.

## API Gateway logging

`RestApiLogGroup` creates `/aws/api-gateway/<stack-name>` with **seven-day**
retention. Access logs include extended request ID, source IP, caller/user,
request time, HTTP method/resource path/protocol, status, response length, and
request ID.

Method settings enable INFO execution logging for all paths and methods,
enable detailed metrics, and set `DataTraceEnabled: false` so full request and
response payload tracing is not enabled by the template. Execution logs are
separate from the configured access log group.

API Gateway needs the account/region-level CloudWatch logging role configured
for REST APIs. This template does not define `AWS::ApiGateway::Account` or that
logging role; `RestApiIamRole` grants DynamoDB query access and is **not** a
CloudWatch logging role. Confirm the platform has configured API logging.

Use access logs for route/status-level investigation and Lambda logs for handler
behavior. API Gateway request IDs and Lambda `aws_request_id` are different
identifiers; record both explicitly in sanitized application logs if you need
cross-service correlation.

## Metrics

AWS provides standard Lambda metrics such as Invocations, Errors, Duration,
Throttles, and ConcurrentExecutions. API Gateway metrics include Count, Latency,
IntegrationLatency, 4XXError, and 5XXError; the template enables detailed method
metrics. DynamoDB also exposes capacity, throttling, and latency metrics.

Filter by the actual deployed function/table names and API stage, especially
when investigating branch deployments. Watch error rates and throttles alongside
latency, not just invocation counts.


The starter does not emit custom Powertools business metrics or enable X-Ray
tracing. Add these intentionally based on your service's reliability targets.


## Datadog dashboard

The template created the **test-dd-dashboard7 API overview** Datadog
dashboard for the production deployment from `main`. Find it in Datadog's
dashboard list by name. It contains:

- **Monitor status**: the current state of every Datadog monitor tagged
  `service:test-dd-dashboard7`, including the alarms below when they
  were created.
- **API Gateway**: request count, 4xx and 5xx error rates, requests and errors
  over time, and overall, maximum, and integration latency.
- **Lambda**: invocations, errors, and throttles, plus per-function invocations,
  errors and throttles, duration, and concurrent executions.
- **DynamoDB**: throttled requests and system errors, consumed read and write
  capacity, and request latency by operation.

The API Gateway widgets filter on the API name and `main` stage. The Lambda and
DynamoDB widgets filter on the production stack's `org:stackName` tag, so they
show no data if Datadog does not import that tag.

Edit the dashboard in Datadog as your service evolves.


## Datadog alarms

The template created these Datadog monitors for the production deployment from
`main`. The API Gateway monitors identify the API by the `apiname` tag, which
is the component name, and the `main` stage. The Lambda and DynamoDB monitors
select the production stack's functions and table by the `org:stackName` tag
that the deployment workflow applies, and alert separately for each function or
table.

| Alarm | Triggering condition | What it usually means |
|---|---|---|
| API Gateway 5xx error rate is elevated | 5xx responses exceed 5% of requests over 10 minutes (warning at 2%) | A Lambda function, its permissions, or DynamoDB is failing |
| API Gateway 4xx error rate is elevated | 4xx responses exceed 20% of requests over 10 minutes (warning at 10%) | Clients are failing authentication or request validation, or sending malformed requests |
| Lambda function error rate is elevated | A function's errors exceed 5% of its invocations over 10 minutes (warning at 2%) | The handler is raising unhandled exceptions, for example from a bad deploy or failing DynamoDB calls |
| Lambda function is throttled | A function is throttled at least once in 10 minutes | The account's concurrency limit or the function's reserved concurrency was reached; throttled API requests fail with 5xx responses |
| DynamoDB table is returning system errors | More than 5 system errors in 10 minutes (warning at any) | DynamoDB itself is failing; check the AWS Health Dashboard |
| DynamoDB table is throttled | At least one throttled request in 10 minutes | Traffic spiked well above the on-demand table's previous peak, or requests are concentrated on a hot partition key |

All monitors evaluate 15 minutes behind real time to allow for CloudWatch
metric delivery to Datadog, and they do not alert when the API receives no
traffic. With very low traffic, a single failed request can exceed a threshold.

Find the monitors in Datadog by the `service:test-dd-dashboard7` tag,
and tune their thresholds and notification targets there as the service's
traffic and reliability targets become clear.

Branch deployments use their own API Gateway stages and are not covered by
these monitors.

The Lambda and DynamoDB monitors depend on Datadog importing the
`org:stackName` resource tag. If they show no data after the service has
received traffic, check in Datadog Metrics Explorer that
`aws.lambda.invocations` and `aws.dynamodb.consumed_read_capacity_units` carry
the `org:stackname:<stack-name>` tag.

DynamoDB user errors, such as validation failures, are reported only for the
whole account and region, not per table, so these monitors do not cover them.
Those failures appear in the Lambda error and API Gateway monitors instead.


## Availability in Datadog

The Datadog AWS integration and log shipping are **configured automatically in
all AWS accounts**. Developers do not need to enable the integration or configure
log forwarding for generated services. Use Datadog to inspect the service's
AWS metrics and shipped Lambda and API Gateway logs.

Baseline tagging is also automatic: `cfn-tags.json` supplies `org:domain`,
`org:system`, and `org:component`, and `ServerlessOpsIO/gha-deploy-aws-sam` adds
additional deployment tags. Use the existing tags to locate and distinguish
resources; extra tagging is only needed for service-specific requirements.

After deployment, verify that a sanitized test log and recent invocation/API
metrics appear in Datadog.

In Datadog Logs Explorer, locate the deployed service or function/log-group
identifier and narrow the time range, then search the request ID from an error.
Use Metrics Explorer to inspect the imported AWS metrics and account/region/stage
dimensions. Exact field names and tags depend on the integration's pipelines.
If data is missing, first confirm it exists in CloudWatch and check the Datadog
time range and resource/tag filters. Treat persistent missing data as a problem
with the existing collection or shipping configuration, not a requirement to
set up a new integration.

## Request investigation checklist

1. Note the timestamp, HTTP method, route, status, and any returned request ID.
2. Find the API Gateway access entry and determine whether the failure happened
   during authorization/validation or during Lambda integration.
3. Inspect the corresponding Lambda invocation and database errors, without
   copying sensitive payloads into tickets.
4. Compare error/latency/throttling metrics with recent deployments.
5. Check Datadog ingestion separately if CloudWatch data is present but missing
   from Datadog.
