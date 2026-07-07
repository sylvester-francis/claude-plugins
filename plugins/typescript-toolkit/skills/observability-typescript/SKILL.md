---
name: observability-typescript
description: Use when making a Node.js/TypeScript service observable — structured logging with pino, Prometheus RED metrics via prom-client, OpenTelemetry tracing with context propagation, correlation IDs, and health/readiness — wired without coupling business logic to telemetry.
---

# Observability — TypeScript / Node

## Overview

Keep telemetry at the edges: a request-context carrier plus middleware (Express) or a module + interceptor (NestJS) install the three pillars once; handlers stay clean. In Node the carrier is **`AsyncLocalStorage`** (OpenTelemetry uses it under the hood for context propagation) — use it to make `request_id` and `trace_id` available to the logger without threading anything.

## The load-order gotcha (do this first)

OpenTelemetry auto-instrumentation **monkey-patches modules at import time**, so the SDK must start **before** anything else is imported. Put it in its own file and load it first:

```ts
// instrumentation.ts — started via:  node --require ./dist/instrumentation.js dist/main.js
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  instrumentations: [getNodeAutoInstrumentations()], // http, express/nest, pg, ioredis...
}).start();
```
Get this wrong and spans silently never appear. The SDK sets the W3C propagator and context manager for you, so `traceparent` flows across hops automatically.

## The three pillars

- **Logging — `pino`** (`nestjs-pino` for Nest). JSON in prod, `pino-pretty` in dev. Give each request a **child logger** (`pino-http` sets `req.log`, or bind fields in `AsyncLocalStorage`). The key idiom is a **`mixin`** that injects `trace_id`/`request_id` on every line by reading the active span + the store — so logs correlate to traces with zero handler code:
  ```ts
  const logger = pino({ mixin() {
    const s = trace.getActiveSpan()?.spanContext();
    return { trace_id: s?.traceId, request_id: als.getStore()?.requestId };
  }});
  ```
- **Metrics — `prom-client`, RED.** Register default metrics; add a `Counter http_requests_total{method,route,status}` (Rate = increase; Errors = filter `5xx`) and a `Histogram http_request_duration_seconds`, recorded in one middleware/interceptor. Expose `/metrics` (`register.metrics()`). **Label by the route template** (`req.route?.path` / the Nest handler path), never the raw URL — raw paths and ids explode cardinality.
- **Tracing — OpenTelemetry.** Auto-instrumentation covers the framework, HTTP client, and DB. Add manual spans only at real boundaries: `tracer.startActiveSpan('charge', span => { ...; span.end(); })`. Sampling + OTLP endpoint via env (`OTEL_TRACES_SAMPLER`, `OTEL_EXPORTER_OTLP_ENDPOINT`).

## Correlation IDs & health

Middleware honors an inbound `X-Request-ID` or mints one (ULID), stores it in `AsyncLocalStorage`, echoes it in the response header; the pino `mixin` puts it on every log line alongside `trace_id`. `/healthz` = **liveness** (dumb 200 — no dependency checks, or a DB blip restart-loops the pod); `/readyz` = **readiness** (pings deps, 503 when degraded so the LB pulls it out). In Nest, use `@nestjs/terminus` for `/readyz`.

## Wiring

- **Express:** `app.use(requestId, pinoHttp({ logger }), metricsMiddleware)`; mount `/metrics`, `/healthz`, `/readyz`; handlers only ever touch the logger via `req.log` / the store.
- **NestJS:** `LoggerModule.forRoot()` (nestjs-pino) + a global metrics `Interceptor` + Terminus health module. Business services inject nothing telemetry-specific.

## Common mistakes

- **Initializing OTel after other imports** → auto-instrumentation never patches; no spans. Load `instrumentation.ts` first (`--require`).
- Labeling metrics by raw path / user id → cardinality blowup; use the route template.
- Dependency checks in `/healthz` → transient outage restart-loops the pod (put them in `/readyz`).
- A single global logger with no request context → logs can't be tied to a trace or user; use a child logger + `mixin`.
- Manual spans around everything — the auto-instrumentation already spans the request and DB calls.
