# Milestone 3: Explore Prometheus, Grafana, Traces

## Objective
Understand the observability stack — what each component does, how data flows, how to read dashboards and traces, and how to design useful monitoring.

## Key Concepts Learned

### The Observability Stack (and what each thing actually does)

| Component | Job | Analogy |
|-----------|-----|---------|
| OTel SDK | Embedded in each microservice, creates spans/metrics | CloudWatch agent, but inside the app code |
| OTel Collector | Receives telemetry, routes it to backends | A message router — receives from many, sends to specific destinations |
| Prometheus | Time-series database — stores metrics | CloudWatch Metrics store |
| Jaeger | Trace database — stores spans, renders waterfalls | AWS X-Ray |
| Grafana | Visualization — queries Prometheus/Jaeger, renders dashboards | CloudWatch Dashboards |

### How Data Gets Into Prometheus

Two models:
- **Pull (classic):** Prometheus scrapes `/metrics` endpoints every 15s
- **Push (our setup):** Collector pushes via OTLP to `prometheus:9090/api/v1/otlp`

Our setup uses push. Prometheus's `prometheus.yml` only scrapes itself — all service data arrives via the collector's push.

**Trade-off of push:** Prometheus can't tell if a service is down (no failed scrape = no signal). Mitigated with heartbeat metrics or external health checks.

### Span Metrics — RED for Free

The collector has a `span_metrics` connector that transforms traces into metrics automatically:

```
Every span that passes through:
  → Increment calls_total{service_name, status_code} by 1
  → Record duration in histogram bucket
```

This gives you Rate, Error, Duration (RED) per service without developers writing any metric code. They just create traces — the collector derives metrics from them.

### Percentiles (p50/p95/p99)

- p50: median — normal user experience
- p95: 95% of requests are faster than this — most users' worst experience
- p99: the unlucky 1%

Averages hide outliers. Percentiles expose them. Use p99 for SLOs.

### OTel Semantic Conventions

Standard metric names defined by the OTel spec:
- `http.server.request.duration` — HTTP server latency
- `rpc.server.call.duration` — gRPC server latency
- `db.client.operation.duration` — database call latency

Auto-instrumentation emits these automatically. You don't choose names — the spec does.

---

## What We Built: Layer 1 Dashboard

### The Problem with Demo Dashboards

The pre-built dashboards are:
- **No hierarchy** — collector internals next to business metrics
- **No context** — numbers without baselines or thresholds
- **Component-centric** — organized by tech (Postgres, Collector) not by user impact
- **Not first-responder friendly** — can't answer "is the business OK?" in 5 seconds

### Our Design: Layered Drill-Down

```
Layer 1: "Is the business okay?" (3 numbers: error rate, latency, throughput)
    ↓ (something's red)
Layer 2: "Which service is broken?" (per-service table sorted by worst)
    ↓ (click the bad one)
Layer 3: "What inside that service is broken?" (traces, DB metrics, resource usage)
```

### Dashboard Panels We Created

1. **Overall Error Rate** — single stat, green/yellow/red thresholds
2. **Frontend p95 Latency** — end-user experience in one number
3. **Total Request Rate** — traffic volume
4. **Error Rate by Service** — table, sorted worst first
5. **Latency by Service (p95)** — timeseries graph
6. **Request Rate by Service** — traffic distribution
7. **Revenue Path Health** — frontend → cart → checkout → payment error rates

### PromQL Patterns Used

```promql
# Error rate for all services:
sum(rate(traces_span_metrics_calls_total{status_code="STATUS_CODE_ERROR", span_kind="SPAN_KIND_SERVER"}[5m]))
/ sum(rate(traces_span_metrics_calls_total{span_kind="SPAN_KIND_SERVER"}[5m])) * 100

# p95 latency per service:
histogram_quantile(0.95, sum(rate(traces_span_metrics_duration_milliseconds_bucket{span_kind="SPAN_KIND_SERVER"}[5m])) by (service_name, le))

# Request rate per service:
sum(rate(traces_span_metrics_calls_total{span_kind="SPAN_KIND_SERVER"}[5m])) by (service_name)
```

---

## Jaeger: Distributed Tracing

### How to Read a Trace

- **Indentation** = "called by" (parent → child relationship)
- **Bar width** = time spent (wider = slower)
- **Gap between parent and child** = waiting time (queuing, connection pool, scheduling)
- **Same indent level** = sequential steps by the same parent

### What a Checkout Trace Shows

```
checkout/PlaceOrder
  ├── cart/GetCart              ← get what's in the cart
  ├── product-catalog (x2)     ← look up item details
  ├── currency/Convert (x3)    ← convert prices
  ├── shipping/get-quote       ← get shipping cost
  ├── payment/Charge           ← take the money
  ├── shipping/ship-order      ← ship it
  ├── cart/EmptyCart            ← clear the cart
  └── email/confirmation       ← send receipt
```

### Performance Issue Found via Tracing

- Checkout calls currency service → 8 second span
- Currency's actual processing → 41 microseconds
- Gap = waiting time (sequential calls, resource contention on single EC2)
- Fix: parallelize independent calls, ensure adequate resource allocation

**Key insight:** Metrics tell you WHAT's wrong (checkout is slow). Traces tell you WHERE (currency calls dominate). Service metrics tell you WHY (resource starvation).

---

## How Spans Are Created (Microservice Instrumentation)

### Three Levels

| Level | Who does it | Code required |
|-------|-------------|---------------|
| Auto-instrumentation | OTel middleware/libraries | 2 lines of setup (wrap HTTP/gRPC server/client) |
| Manual spans | Developer | `tracer.Start(ctx, "name")` + `defer span.End()` |
| Attributes/events | Developer | `span.SetAttributes(...)` / `span.AddEvent(...)` |

### How Auto-Instrumentation Works

```go
// One-time setup: wrap the gRPC server
grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))
```

Now every incoming gRPC call automatically gets a span. No per-function code.

### How Trace Context Propagates

```
Checkout (client middleware):
  1. Creates span
  2. Injects into gRPC headers: traceparent: 00-<trace-id>-<span-id>-01
  3. Sends request

Cart (server middleware):
  1. Reads traceparent header
  2. Creates child span (same trace-id, parent = checkout's span-id)
  3. Runs handler
```

It's header propagation. Same concept as X-Forwarded-For — an agreed-upon header format both sides understand.

### Parallel Calls

Same trace-id, same parent span-id, different span-ids. Jaeger renders them overlapping horizontally. Sequential calls render stacked vertically.

---

## gRPC vs REST

| | REST (HTTP/JSON) | gRPC (HTTP/2 + Protobuf) |
|---|---|---|
| When to use | External-facing, browser clients, simple integrations | Internal service-to-service at scale |
| Speed | Slower (text parsing) | Faster (~10x less serialization CPU) |
| Contract | Informal (docs) | Strict (.proto files, compile-time errors) |
| Pattern in OTel Demo | Browser → frontend | frontend → all backend services |

---

## Debugging Commands

```bash
# What services does Prometheus know about?
curl -s http://localhost:9090/api/v1/label/service_name/values

# What metrics exist?
curl -s http://localhost:9090/api/v1/label/__name__/values

# Query a specific metric
curl -s 'http://localhost:9090/api/v1/query?query=traces_span_metrics_calls_total{service_name="checkout"}'

# Check Grafana datasources
curl -s http://localhost:3000/api/datasources -u admin:admin

# Create/update a dashboard via API
curl -X POST http://localhost:3000/api/dashboards/db -H 'Content-Type: application/json' -u admin:admin -d @dashboard.json
```

---

## Collector Config (How Routing Works)

```yaml
exporters:
  otlp_grpc/jaeger:
    endpoint: "jaeger:4317"           # traces go here
  otlp_http/prometheus:
    endpoint: "http://prometheus:9090/api/v1/otlp"  # metrics go here
  opensearch:
    http:
      endpoint: "http://opensearch:9200"  # logs go here

service:
  pipelines:
    traces:
      exporters: [otlp_grpc/jaeger, span_metrics]  # span_metrics = connector that generates RED metrics
    metrics:
      exporters: [otlp_http/prometheus]
    logs:
      exporters: [opensearch]
```

---

## Interview-Ready Statements

1. "I built a Layer 1 dashboard using RED methodology — rate, error, duration — derived automatically from span metrics via the OTel Collector's spanmetrics connector."

2. "Traces showed me that checkout's 32-second latency was dominated by sequential downstream calls. The services themselves responded in microseconds — the latency was queuing and resource contention."

3. "I use a layered drill-down approach: golden signals on top, per-service RED in the middle, component deep-dive at the bottom. You wake someone up because p99 breached the SLO, not because a buffer count spiked."

4. "Auto-instrumentation handles 80% of span creation via middleware hooks on HTTP/gRPC/DB libraries. Developers only manually instrument business-specific logic that the framework can't see."

5. "Push-based metric collection removes the need to manage scrape targets at scale, but you lose the implicit health signal of a failed scrape. We compensate with heartbeat metrics and monitoring the collector itself."

6. "gRPC for internal service mesh because of binary serialization efficiency and compile-time contract enforcement. REST for external-facing APIs where browser compatibility and debuggability matter."
