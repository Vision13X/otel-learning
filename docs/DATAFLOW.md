# Data Flow

How telemetry and deployment data flows through the system. Updated each milestone.

---

## Current State (Milestone 1): No Data Flow Yet

No services deployed. This document will track:

1. **Deployment flow:** How code gets from GitHub to running infrastructure
2. **Telemetry flow:** How observability data moves from services to dashboards

---

## Milestone 2 (Preview): EC2 + Docker Compose

```
┌─────────────────────────────────────────────────────────────────┐
│  EC2 Instance (Docker Compose)                                  │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ Service A│  │ Service B│  │ Service C│  ... (15 services)   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                     │
│       │              │              │                            │
│       │   OTLP (gRPC, port 4317)   │                            │
│       └──────────────┼──────────────┘                           │
│                      ▼                                          │
│            ┌──────────────────┐                                 │
│            │  OTel Collector  │                                  │
│            │  - Receives all  │                                  │
│            │    telemetry     │                                  │
│            └───┬────────┬────┘                                  │
│                │        │                                       │
│         metrics│        │traces                                 │
│                ▼        ▼                                       │
│     ┌────────────┐  ┌─────────┐                                │
│     │ Prometheus │  │ Jaeger  │                                 │
│     │ (scrapes)  │  │ (store) │                                 │
│     └─────┬──────┘  └────┬────┘                                │
│           │               │                                     │
│           └───────┬───────┘                                     │
│                   ▼                                             │
│            ┌────────────┐                                       │
│            │  Grafana   │                                        │
│            │ (visualize)│                                        │
│            └────────────┘                                       │
│                   │                                             │
│            port 3000 exposed                                    │
└───────────────────┼─────────────────────────────────────────────┘
                    │
              browser access
              (your laptop)
```

### Key Protocols:
- **OTLP (OpenTelemetry Protocol):** Services → Collector. gRPC on port 4317, HTTP on 4318.
- **Prometheus scrape:** Collector exposes `/metrics` endpoint, Prometheus pulls from it.
- **Jaeger receiver:** Collector pushes traces to Jaeger via OTLP or Jaeger-native protocol.
- **Grafana data sources:** Queries Prometheus (PromQL) and Jaeger (trace IDs) for visualization.

*This will be fleshed out with actual config details when we deploy in Milestone 2.*
