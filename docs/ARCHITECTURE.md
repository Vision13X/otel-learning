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

## Next Milestone (2): EC2 + Docker Compose

```
GitHub ──push──► GitHub Actions ──deploy──► EC2 Instance
                                              │
                                    ┌─────────┴─────────┐
                                    │  Docker Compose    │
                                    │  - 15 microsvcs   │
                                    │  - OTel Collector  │
                                    │  - Prometheus      │
                                    │  - Grafana         │
                                    │  - Jaeger          │
                                    └───────────────────┘
```

*Updated each milestone. Previous states preserved in git history.*
