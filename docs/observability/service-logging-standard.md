# Bond Service Logging Standard

This standard applies to every active Bond service, job, workflow, Lambda, web server, mobile runtime, Modal workload, and Unity server process that emits operational logs.

The goal is not "more logs." The goal is logs that are predictable, searchable in Grafana Loki, correlated with Tempo traces, and useful during incidents without leaking data or creating noise.

## Core Principles

1. Logs are for facts that need durable operational review. Traces are the canonical record of request shape, latency, span hierarchy, and per-request success paths.
2. Every runtime log event must be machine-readable JSON or a platform-native structured log that the collector converts to structured fields.
3. Every runtime log event must have one stable event identity. Use `event.name`, not prose as the only identifier.
4. Logs must correlate to traces whenever an active span exists. Emit `trace_id` and `span_id` fields using OpenTelemetry context.
5. Fields must be stable, low-cardinality where they are used as labels, and intentionally named. Prefer `snake_case` for log body fields.
6. Logs must never include secrets, tokens, authorization headers, cookies, raw request or response bodies, unbounded prompts, model outputs, full catalog records, or customer PII.
7. Logging code must be boring and centralized. Do not hand-build JSON strings, write ad hoc `print` or `console.log` calls in runtime paths, or invent per-service schemas.

## Required Event Shape

Every operational log event must include:

| Field | Requirement |
| --- | --- |
| `timestamp` | Emitted by the runtime/logger or collector. |
| `severity` | One of `debug`, `info`, `warn`, `error`, or `fatal`. |
| `event.name` | Stable dot-delimited event name such as `catalog.sync.completed`. |
| `message` | Short human-readable summary. This may be stable prose, but it must not be the only event identity. |
| `service.name` | OpenTelemetry resource attribute. Collector-provided is acceptable. |
| `deployment.environment.name` | OpenTelemetry resource attribute. Collector-provided is acceptable. |
| `trace_id` | Required when an active trace span exists. |
| `span_id` | Required when an active trace span exists. |

Use dot-delimited `event.name` values:

```text
service.domain.action
service.domain.action.outcome
```

Examples:

```text
catalog.sync.started
catalog.sync.completed
catalog.sync.failed
asset.processor.blender_invocation.failed
modal.model.loaded
workflow.restate_handler.failed
```

## Severity

Use severities consistently:

| Severity | Use |
| --- | --- |
| `debug` | Local or explicitly enabled diagnostic detail. Not normally emitted in production. |
| `info` | Meaningful lifecycle transitions, asynchronous job state changes, configuration summaries, and business-relevant outcomes. |
| `warn` | Unexpected but handled conditions, retryable failures, degraded external dependencies, or skipped work that needs review. |
| `error` | A failed operation requiring investigation or alert context. |
| `fatal` | Process cannot continue. Prefer `error` plus explicit exit in languages without a real fatal level. |

Do not emit one `info` log for every successful inbound HTTP request. That is trace data. Emit logs for request-boundary failures, unusual policy decisions, retries, dropped work, or domain outcomes that are not obvious from spans.

## Common Fields

Use these field names across repos.

| Field | Use |
| --- | --- |
| `operation.name` | Stable operation name when the event is tied to a command, handler, workflow, or job step. |
| `outcome` | `success`, `failure`, `skipped`, `retrying`, `timeout`, or another small controlled value. |
| `duration_ms` | Operation duration in milliseconds when logging completion of asynchronous work. |
| `attempt` | Retry attempt number. |
| `max_attempts` | Retry ceiling. |
| `request_id` | Public request correlation id, if distinct from trace id. |
| `job_id` | Background job id. |
| `workflow_id` | Restate/workflow id. |
| `tenant_id` | Tenant/org id when safe and needed for support. |
| `environment` | Business environment only when distinct from `deployment.environment.name`. |

Domain identifiers are allowed, but do not promote high-cardinality identifiers into Loki labels. Keep them as log fields.

## Error Fields

Every error log must include:

| Field | Requirement |
| --- | --- |
| `error.type` | Exception class, Go error category, status family, or stable failure type. |
| `error.message` | Sanitized error message. |
| `error.stack` | Stack trace when available and useful. |

Add these when applicable:

| Field | Use |
| --- | --- |
| `error.retryable` | Boolean. |
| `http.status_code` | Failed upstream/downstream HTTP status. |
| `rpc.system` | RPC system such as `grpc`, `http`, or `restate`. |
| `db.system` | Database system. |
| `dependency.name` | External dependency name such as `openai`, `modal`, `s3`, or `algolia`. |

Do not put raw exception objects in arbitrary fields. Normalize through the repo's logging helper.

## Request Logging

Inbound HTTP/RPC request completion logs are not required for successful requests. A complete trace with route, status, latency, service, and environment is the source of truth.

Request-bound logs are appropriate when:

1. The request fails before a trace/span is created.
2. The service makes an unusual authorization, routing, throttling, fallback, or validation decision.
3. The request triggers asynchronous work whose lifecycle continues after the response.
4. A dependency failure, retry storm, or degraded behavior needs a durable incident breadcrumb.

When request-bound logs are emitted, include:

| Field | Requirement |
| --- | --- |
| `http.method` | HTTP method. |
| `http.route` | Templated route, never raw path with ids. |
| `http.status_code` | Status code when known. |
| `duration_ms` | Duration when logging completion/failure. |

## Runtime Requirements

### Go

Use `log/slog` with JSON output and the shared OpenTelemetry trace-context handler. Do not use `log.Print*`, `log.Fatal*`, `fmt.Print*`, or hand-built JSON in runtime paths.

Fatal startup failures should be logged with structured fields and then exit:

```go
slog.Error("database connection failed",
    "event.name", "service.startup.failed",
    "error.type", "database_connection",
    "error.message", err.Error(),
)
os.Exit(1)
```

### Python and Modal

Use the shared structlog/OpenTelemetry logging setup for all runtime paths, including Modal functions. Do not use `print` for operational events. Mirroring `print` to Loki is only a temporary compatibility bridge while code is being migrated.

Every structlog event must render `event.name` explicitly and must include trace context when an active span exists.

### TypeScript, Next.js, and Node

Server-side runtime code must use the shared logging helper. Do not use `console.log`, `console.warn`, or `console.error` in server runtime paths except inside the logging helper itself.

Browser-only code may use console logging only for deliberate developer diagnostics that are not operational telemetry. Production client operational telemetry should go through the browser telemetry pipeline.

### Swift and iOS

Use OSLog with privacy annotations for local diagnostics and the approved telemetry path for operational events. Do not log raw payloads, user content, credentials, or device-local secrets.

### Unity and C#

Runtime/server Unity code should use a structured logging adapter over `ILogger`/`Debug.unityLogger` so emitted events carry `event.name`, severity, and correlation fields where available. Plain `Debug.Log*` calls are acceptable only in editor-only tooling or tests.

## Anti-Patterns

These must be removed from production/runtime paths:

1. `print`, `console.log`, `fmt.Println`, `log.Printf`, `Debug.Log`, or `NSLog` as operational logging.
2. Prose-only log lines with no `event.name`.
3. Raw payloads, prompts, generated content, file contents, request bodies, response bodies, or full catalog records.
4. High-cardinality data as labels.
5. Logs emitted in tight polling loops when nothing happened.
6. Per-success HTTP request logs duplicated by traces.
7. Error logs without `error.type` and `error.message`.
8. Logs that interpolate ids into `message` instead of structured fields.

## Migration Acceptance Checklist

A repo is logging-compliant when:

1. Runtime log emitters use one centralized logging path for the language/runtime.
2. Runtime operational events include `event.name`.
3. Error logs normalize `error.type`, `error.message`, and stack traces when useful.
4. Active trace context is attached to log events.
5. Raw console/print/debug emitters are removed from runtime paths or proven to be dev-only/test-only.
6. Successful inbound request completion logs are removed unless they carry domain value not present in traces.
7. Large payload and PII logging is removed or redacted.
8. The service can be queried in Grafana Loki by `service_name`, `deployment_environment_name`, and the normalized `event_name` label.
9. The service can jump from a relevant log event to the associated Tempo trace when trace context exists.
