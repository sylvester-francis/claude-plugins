---
name: observability-python
description: Use when making a Python service (FastAPI/Django/Flask) observable — structured logging with structlog, Prometheus RED metrics via prometheus_client, OpenTelemetry tracing with context propagation, correlation IDs via contextvars, and health/readiness.
---

# Observability — Python

## Overview

Keep telemetry at the edges: middleware installs the three pillars once; handlers stay clean. The request-scoped carrier is **`contextvars`** (works across `async`), used to expose `request_id`/`trace_id` to the logger without threading parameters.

## The three pillars

- **Logging — `structlog`** (or stdlib `logging` + a JSON formatter). JSON in prod. Bind request-scoped fields once; the key idiom is a **processor that injects `trace_id`/`request_id`** on every event by reading the active OTel span + a `contextvar`, so logs correlate to traces with no handler code:
  ```python
  def add_trace(_, __, event: dict) -> dict:
      span = trace.get_current_span().get_span_context()
      if span.is_valid:
          event["trace_id"] = format(span.trace_id, "032x")
      if rid := request_id_var.get(None):
          event["request_id"] = rid
      return event
  structlog.configure(processors=[..., add_trace, structlog.processors.JSONRenderer()])
  ```
- **Metrics — `prometheus_client`, RED.** A `Counter("http_requests_total", ["method","route","status"])` (Rate = increase; Errors = filter `5xx`) and a `Histogram("http_request_duration_seconds", ...)`, recorded in one middleware; expose `/metrics` (`make_asgi_app()`, or `prometheus-fastapi-instrumentator`). **Label by the route template** (`request.scope["route"].path` / the matched pattern), never the raw path — raw paths and ids explode cardinality.
- **Tracing — OpenTelemetry.** Use the auto-instrumentation for your framework (`opentelemetry-instrumentation-fastapi`/`django`/`flask` + the DB/HTTP-client instrumentors), an OTLP exporter, and the SDK's W3C propagator so `traceparent` flows across hops. Add manual spans only at real boundaries (`tracer.start_as_current_span("charge")`). Bootstrap with `opentelemetry-bootstrap -a install` + `opentelemetry-instrument python app.py`, or wire the instrumentors in code.

## Correlation IDs & health

A middleware honors an inbound `X-Request-ID` or mints one (ULID), sets the `contextvar`, echoes it in the response header; the log processor puts it (and `trace_id`) on every line. `/healthz` = **liveness** (dumb 200 — no dependency checks, or a DB blip restart-loops the pod); `/readyz` = **readiness** (pings deps, 503 when degraded so the LB pulls it out).

## Wiring

- **FastAPI/ASGI:** an ASGI middleware for request-id + RED metrics; instrument via `FastAPIInstrumentor.instrument_app(app)`; mount `/metrics`, `/healthz`, `/readyz`. Handlers only ever call `logger.info(...)`.
- **Django:** a middleware for request-id/metrics; `DjangoInstrumentor().instrument()`; `django-prometheus` or a `/metrics` view.

## Common mistakes

- Labeling metrics by raw path / user id → cardinality blowup; use the matched route template.
- Dependency checks in `/healthz` → transient outage restart-loops the pod (put them in `/readyz`).
- Threading a logger through call chains instead of a `contextvar` + processor → uncorrelated logs.
- Blocking the event loop with synchronous logging/exporters in an async app (use async-safe handlers / batch exporters).
- Manual spans everywhere — the framework auto-instrumentation already spans the request and DB calls; add spans at boundaries only.
