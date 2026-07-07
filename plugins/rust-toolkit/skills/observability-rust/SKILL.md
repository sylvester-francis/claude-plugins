---
name: observability-rust
description: Use when making a Rust service (axum/tokio) observable — the tracing crate for structured logs and spans, tracing-opentelemetry for distributed tracing, Prometheus RED metrics, correlation IDs, and health/readiness — wired as tower layers without coupling business logic.
---

# Observability — Rust

## Overview

In Rust the **`tracing` crate is the unifying abstraction** — it produces structured logs *and* spans from the same API, and `tracing-opentelemetry` bridges those spans to OpenTelemetry for distributed tracing. Install the pillars once as **tower layers**; handlers only call `tracing` macros. This makes logs, traces, and metrics line up for free.

## The three pillars

- **Logging + spans — `tracing` + `tracing-subscriber`.** Instrument handlers with `#[tracing::instrument]` (creates a span with the args as fields); emit `info!`/`error!` (span-aware). `tracing-subscriber` formats (JSON in prod via `fmt().json()`) and filters with `EnvFilter` (`RUST_LOG=info,myapp=debug`). A flat log can't show the causal path through async code; spans can.
- **Distributed tracing — `tracing-opentelemetry`.** Add an `OpenTelemetryLayer` to the subscriber so every `tracing` span becomes an OTel span; export with `opentelemetry-otlp`. Set the W3C propagator and extract/inject `traceparent` at the edges (`tower-http` `TraceLayer` on inbound; propagate on outbound clients). Because the OTel layer stamps the `trace_id` onto the span, your `tracing` logs already carry it — logs ↔ traces correlate automatically.
- **Metrics — RED.** Use the `metrics` facade + `metrics-exporter-prometheus` (or `axum-prometheus`, which wraps this). A tower layer records `http_requests_total{method,route,status}` (Rate = increase; Errors = `5xx`) and `http_request_duration_seconds` (Duration); expose `/metrics`. **Label by the matched route (`MatchedPath`)**, never the raw URI — raw paths and ids explode cardinality.

## Correlation IDs & health

`tower-http`'s `RequestIdLayer` (or a small middleware) honors/mints an `X-Request-ID` and puts it in a span field, so it rides on every log line alongside `trace_id`. `/healthz` = **liveness** (dumb `200` — no dependency checks, or a DB blip restart-loops the pod); `/readyz` = **readiness** (pings deps, `503` when degraded so the LB pulls it out).

## Wiring (axum + tower)

```rust
// subscriber: fmt (JSON) + EnvFilter + the OTel layer — one place, in main
tracing_subscriber::registry()
    .with(EnvFilter::from_default_env())
    .with(tracing_subscriber::fmt::layer().json())
    .with(tracing_opentelemetry::layer().with_tracer(otlp_tracer()?))
    .init();

let app = Router::new()
    .route("/orders/{id}", get(get_order))   // handler: #[instrument], tracing macros only
    .route("/metrics", get(metrics_handler))
    .route("/healthz", get(|| async { "ok" }))       // liveness
    .route("/readyz", get(readyz))                    // readiness: 503 until deps healthy
    .layer(TraceLayer::new_for_http())               // span per request + W3C extract
    .layer(prometheus_layer);                          // RED metrics
```
Business handlers stay telemetry-free — they just `#[instrument]` and `info!`. Swap the OTLP endpoint or the fmt layer and no handler changes.

## Common mistakes

- Using `log` instead of `tracing` in async code → no spans, no async-aware causal path.
- Labeling metrics by raw URI / id instead of `MatchedPath` → cardinality blowup.
- Dependency checks in `/healthz` → transient outage restart-loops the pod (use `/readyz`).
- Forgetting the OTel propagator / not extracting `traceparent` at the edge → traces die at the first hop.
- Building tracing/OTLP setup per-handler instead of once as layers in `main`.
