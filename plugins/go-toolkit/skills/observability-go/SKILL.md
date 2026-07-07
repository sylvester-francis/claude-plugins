---
name: observability-go
description: Use when making a Go service observable — structured logging with slog, Prometheus RED metrics, OpenTelemetry tracing with context propagation, correlation IDs, and health/readiness — wired as middleware without coupling business logic to telemetry.
---

# Observability — Go

## Overview

**`context.Context` is the only coupling.** Business logic never imports a metrics client, a tracer, or a concrete logger — it takes a `ctx` and at most calls `slog.InfoContext(ctx, ...)`. Everything else (spans, RED metrics, request IDs, trace correlation) is installed once as an HTTP middleware chain in `main`. Telemetry is cross-cutting infrastructure, not calls sprinkled through handlers.

## The three pillars

- **Logging — `log/slog`.** `JSONHandler` in prod, `TextHandler` in dev; a `slog.LevelVar` so verbosity is runtime-tunable. The key idiom is a **wrapping `slog.Handler`** that reads request-scoped fields off `ctx` (`request_id`, `trace_id` from the active span) and adds them as attributes — so every `...Context` call is auto-correlated to its trace and request without the caller knowing. Always use the `InfoContext`/`ErrorContext` variants so `ctx` reaches the handler.
- **Metrics — Prometheus, RED.** `client_golang` + `promauto`; expose `promhttp.Handler()` at `/metrics`. One middleware records all of RED from a `CounterVec http_requests_total{method,route,status}` (Rate = its increase; Errors = filter `status=~"5.."`) and a `HistogramVec http_request_duration_seconds` (Duration). **Label by the route template (`/users/{id}`), never the raw path** — raw paths and user/trace IDs explode cardinality.
- **Tracing — OpenTelemetry.** OTLP exporter → Collector, `BatchSpanProcessor`, a `Resource` with `service.name`. Sample with `ParentBased(TraceIDRatioBased(x))` so downstreams honor the caller. Set a global W3C propagator, wrap the server with `otelhttp.NewHandler` (extracts `traceparent`, starts the span) and every client with `otelhttp.NewTransport` (injects it) — that's how the trace survives a hop. Add custom spans only at meaningful boundaries; `defer tp.Shutdown(ctx)` to flush.

## Correlation IDs & health

Middleware honors an inbound `X-Request-ID` or mints one, stores it in `ctx`, echoes it in the response header; the log handler harvests it (and `trace_id`) so logs/traces/metrics line up. `/healthz` = **liveness** (dumb 200 if the process serves — no dependency checks, or a DB blip restart-loops the pod); `/readyz` = **readiness** (pings deps, 503 when degraded so the LB pulls it out). Both bypass the noisy middleware.

## The snippet that ties it together

```go
type contextHandler struct{ slog.Handler }
func (h contextHandler) Handle(ctx context.Context, r slog.Record) error {
    if id, ok := ctx.Value(requestIDKey).(string); ok { r.AddAttrs(slog.String("request_id", id)) }
    if sc := trace.SpanContextFromContext(ctx); sc.HasTraceID() {
        r.AddAttrs(slog.String("trace_id", sc.TraceID().String())) // logs <-> traces
    }
    return h.Handler.Handle(ctx, r)
}

slog.SetDefault(slog.New(contextHandler{slog.NewJSONHandler(os.Stdout, nil)}))
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{}, propagation.Baggage{}))

mux.Handle("/metrics", promhttp.Handler())
mux.HandleFunc("/healthz", live)   // liveness: 200 if process is up
mux.HandleFunc("/readyz", ready)   // readiness: 503 until deps healthy

// wrapped last = outermost = runs first
var h http.Handler = appRoutes()          // handlers: ctx-only, telemetry-free
h = loggingMW(h); h = metricsMW(h)        // RED
h = otelhttp.NewHandler(h, "server")      // span + W3C propagation
h = requestIDMW(h); h = recoverMW(h)      // id in ctx; panic -> 500
```

## Why it holds up

Chain order is deliberate: `recover` outermost catches panics from everything; `requestID`/`otelhttp` before `metrics`/`logging` so the id and `trace_id` are in `ctx` when those layers record. A new endpoint inherits all three pillars for free; swap the OTLP backend or log sink and no handler changes.

## Common mistakes

- Threading a `*slog.Logger` through every function instead of the context handler.
- Labeling metrics by raw path / user id → cardinality explosion; label by route template.
- Dependency checks in `/healthz` → a transient outage restart-loops the pod (put them in `/readyz`).
- Custom spans everywhere — `otelhttp` already gives you the request span; add spans at boundaries only.
- Forgetting `otelhttp.NewTransport` on clients → the trace dies at the first outbound call.
