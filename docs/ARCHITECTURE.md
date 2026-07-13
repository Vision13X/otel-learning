# Architecture — Current State

## Milestone 1: Repository + AWS Account

```
┌─────────────────────────────────────────────────────────┐
│                      GitHub                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Vision13X/otel-learning                          │  │
│  │  - Secrets: AWS_ACCESS_KEY_ID                     │  │
│  │  - Secrets: AWS_SECRET_ACCESS_KEY                 │  │
│  │  - Secrets: AWS_REGION (us-east-1)                │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
                          │ (future: GitHub Actions)
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    AWS Account                           │
│                                                         │
│  IAM Users:                                             │
│  ┌─────────────────┐    ┌──────────────────┐           │
│  │  kiro-deploy    │    │  vigne-admin     │           │
│  │  (programmatic) │    │  (console only)  │           │
│  │  Group:         │    │  Group:          │           │
│  │  pipeline-      │    │  human-admins    │           │
│  │  automation     │    │  (AdminAccess)   │           │
│  └─────────────────┘    └──────────────────┘           │
│                                                         │
│  Region: us-east-1                                      │
│  Resources deployed: none yet                           │
└─────────────────────────────────────────────────────────┘
```

## Milestone 2: EC2 + Docker Compose (CURRENT)

```
Internet (your browser)
    │
    ├── :8080 → frontend-proxy (Envoy) → web store
    ├── :3000 → Grafana (dashboards)
    ├── :9090 → Prometheus (metrics)
    └── :16686 → Jaeger (traces)

┌─────────────────────────────────────────────────────────┐
│  EC2: t3.large, 30GB EBS, us-east-1                     │
│  IAM Role: otel-demo-ec2-role (SSM access)              │
│  Security Group: otel-demo-sg                           │
│                                                         │
│  Docker Compose (3 files merged):                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │  15 microservices + load generator                │  │
│  │       │ OTLP (gRPC :4317)                        │  │
│  │       ▼                                          │  │
│  │  OTel Collector                                   │  │
│  │       ├── metrics → Prometheus                   │  │
│  │       ├── traces → Jaeger                        │  │
│  │       └── logs → (OpenSearch, disabled)          │  │
│  │                                                  │  │
│  │  Grafana ← queries Prometheus + Jaeger           │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Next Milestone (3): Explore Prometheus, Grafana, Traces

Deep dive into the running observability stack — PromQL, Grafana dashboards, distributed tracing.

*Updated each milestone. Previous states preserved in git history.*
