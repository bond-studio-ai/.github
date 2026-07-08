# Logging Audit - 2026-07-08

This audit covers active local Bond repositories under `/Users/chris.seltzer/Developer/Bond`, cross-checked against live GitHub repository state. Archived and decommissioned repositories were excluded, including the old prototype monolith and inactive iOS/mobile repos. `ios-studio-prototype` is the only active iOS repo in scope.

The audit is a prioritization pass, not a compliance verdict. Counts identify likely raw emitters and schema gaps so implementation work can be batched by runtime and blast radius.

## High-Priority Cleanup Clusters

| Repo | Main issue |
| --- | --- |
| `service-video-reconstruct` | Heavy raw `print`/console-style logging and little structured event identity. |
| `service-scan-processor` | Heavy raw logging in a large codebase, likely including runtime paths. |
| `service-avery` | High log volume, many request-completion candidates, mixed structured and unstructured patterns. |
| `web-app` | Many client/server console emitters; needs server/runtime separation and structured server logs. |
| `service-on-demand-reconstruction` | High raw logging volume and mixed logger patterns. |
| `service-image-generation` | High raw and logger volume with request-completion duplication risk. |
| `service-catalog`, `service-catalog-feed`, `service-catalog-ingestion` | Mixed catalog-service logging shape; structured events exist but are not uniform. |
| `service-room`, `service-room-design` | Raw logging plus request/log duplication candidates. |
| `model-compositor` | Raw logging, no structured logger baseline. |
| Modal workloads such as `service-3d-generation`, `service-comfyui`, `service-spatiallm`, `service-vggt-slam` | Modal stdout-oriented logging needs migration to structured OpenTelemetry-compatible logging. |
| Unity repos | `Debug.Log*`/console-style logging needs runtime-vs-editor separation and a structured server/runtime adapter. |

## Runtime Batches

1. Python/Modal services: replace runtime `print` emitters, normalize structlog output, add trace/error processors, and remove temporary print mirroring.
2. Go services and Restate workflows: standardize `slog` wrappers, remove `log.Fatal*`/`fmt.Print*`, normalize `event.name`, and preserve trace context.
3. TypeScript/Next.js/web: migrate server runtime console calls to a shared logger and keep browser dev diagnostics out of operational telemetry.
4. C#/Unity server/runtime code: add or reuse a structured logging adapter, then leave editor-only diagnostics explicitly marked as editor-only.
5. iOS: verify OSLog privacy and client telemetry paths, then remove operational `print`/`NSLog` usage from production code.
6. Infrastructure and CLI repos: keep human CLI output where appropriate, but ensure automation/runtime failures use structured logs where they are ingested by Loki.

## Acceptance Target

Every migrated repo must satisfy the checklist in `docs/observability/service-logging-standard.md` before being marked complete.

