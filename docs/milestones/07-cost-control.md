# Milestone 7: Cost Control Pipeline

## Objective
Ensure the project never exceeds $70/month through automated financial guardrails.

## Design Principles

1. **Hard guarantees over soft alerts** — pipeline FAILS (exit 1), not "sends a warning email"
2. **No assumptions about cost** — fetch real pricing from AWS API, don't hardcode
3. **Detect nightly destroy failure** — orphan check blocks new deploys if resources leaked
4. **Defense in depth** — multiple layers, any one of which can stop the bleed

## The Layers

```
Layer 1: Budget gate on deploy     → blocks if MTD >= $70
Layer 2: Nightly full destroy      → ensures $0/hr overnight  
Layer 3: Validate-destroy          → proves nothing survived
Layer 4: Orphan check next morning → blocks if something leaked
Layer 5: $20/day sanity cap        → catches wrong instance types
```

## Cost Math

| Resource | Hourly | Daily (16 hrs) | Weekly | Monthly (worst) |
|----------|--------|----------------|--------|-----------------|
| EKS control plane | $0.10 | $1.60 | $11.20 | $48.00 |
| t3.large × 2 | $0.1664 | $2.66 | $18.62 | $79.87 |
| NAT Gateway | $0.045 | $0.72 | $5.04 | $21.60 |
| **Total 24/7** | **$0.31** | **$4.98** | **$34.86** | **$149.47** |
| **With nightly destroy (16h/day)** | — | **$4.26** | **$29.82** | **~$85** |
| **Our actual usage (~4 days)** | — | — | — | **~$17-25** |

## Nightly Destroy Schedule

- **19:30 UTC (1:00 AM IST):** cost-control.yml fires, full destroy
- **20:00 UTC (1:30 AM IST):** validate-destroy.yml fires, confirms everything dead

## Pricing Mechanism

1. Call AWS Pricing API with instance type + region + OS filters
2. Extract on-demand hourly price from response
3. If API fails → fallback lookup table (common types hardcoded)
4. If instance type unknown → assume $1/hr (conservative, triggers sanity cap)

## Testing Strategy

- `test-lifecycle.yml` — full deploy + destroy cycle, validates both work
- `validate-destroy.yml` — audits account for orphaned resources (no deploy needed)
- Orphan check in cost-gate — catches leaked resources before spending more

## Interview-Ready Statements

1. "Cost control isn't an alert you ignore — it's a hard gate. The pipeline exits non-zero and the deploy doesn't happen."

2. "I designed for the worst case: if nightly destroy fails silently, the morning deploy's orphan check catches it before creating more resources."

3. "At scale, this becomes a shared reusable workflow. Each team calls cost-gate with their resource spec. Budget is federated per team."

4. "The Pricing API gives real-time cost data, but it's eventually consistent. The fallback table and $20/day sanity cap compensate for API failures."
