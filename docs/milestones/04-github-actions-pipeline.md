# Milestone 4: GitHub Actions — CI/CD Pipelines

## Objective
Build a pipeline system that deploys and destroys infrastructure with financial guardrails.

## What We Built

### Workflows Created

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `deploy.yml` | Manual / push to main | Creates EKS cluster + installs OTel Demo via Helm |
| `destroy.yml` | Manual | Tears down cluster on demand |
| `cost-control.yml` | Nightly cron (1 AM IST / 19:30 UTC) | Full destroy overnight + emergency kill at $70 |
| `cost-gate.yml` | Called by deploy.yml | Reusable validation: orphan check, pricing, budget |
| `validate-destroy.yml` | Cron (1:30 AM IST) + manual | Proves nothing is running after destroy |
| `test-lifecycle.yml` | Manual | End-to-end: deploy → verify → destroy → validate |

### Pipeline Architecture

```
deploy.yml ──calls──► cost-gate.yml (validate)
                          │
                  ┌───────┴────────┐
                  │ Gate 1: Orphans │ (cluster/EC2/NAT/ELB already exist?)
                  │ Gate 2: Pricing │ (AWS Pricing API → real $/hr)
                  │ Gate 3: Budget  │ (MTD + daily cost >= $70?)
                  │ Gate 4: Sanity  │ (> $20/day = wrong instance)
                  └───────┬────────┘
                          │ approved
                          ▼
                  deploy (create EKS + helm install)

Nightly 1 AM IST:
  cost-control.yml → full destroy → $0/hr

1:30 AM IST:
  validate-destroy.yml → proves nothing survived
```

### Key Design Decisions

1. **Cost-gate is separate from cost-control** — GitHub Actions doesn't allow `schedule` + `workflow_call` in the same file. Reusable gate is in `cost-gate.yml`, scheduled jobs in `cost-control.yml`.

2. **Full nightly destroy (not scale-down)** — User prefers $0 overnight over fast morning startup. 15 min rebuild is acceptable.

3. **Live pricing from AWS Pricing API** — No hardcoded costs. Falls back to a lookup table if API fails. Catches if someone changes instance type to something expensive.

4. **Orphan check before deploy** — If nightly destroy failed, blocks new deploys until the leak is fixed. Prevents cost accumulation.

5. **Sanity cap at $20/day** — Even if budget math passes, if a single day costs > $20, something is wrong (wrong instance type). Hard block.

6. **Monthly hard cap: $70** — Cannot be exceeded under any scenario. Deploy blocked when approached, emergency destroy when crossed.

## GitHub Actions Concepts Learned

- **`workflow_dispatch`** — manual trigger button in GitHub UI
- **`workflow_call`** — one workflow calling another (reusable workflow)
- **`needs:`** — job dependency (cost-gate must pass before deploy runs)
- **`secrets: inherit`** — pass secrets to called workflows
- **`schedule` + `cron`** — GitHub as the scheduler (like CloudWatch Events → Lambda)
- **`exit 1`** — pipeline fails, blocks downstream jobs
- **Environment variables** — `env:` block at workflow level
- **Job outputs** — pass data between jobs

## Issues Encountered

### Helm schema error
- `components.frontendProxy` not valid in newer chart version
- Fix: install with defaults, patch service type after

### EKS auth — cluster creator gets access, nobody else
- Cluster created by `kiro-deploy` pipeline user
- `vision-admin` (CloudShell) couldn't access until explicitly granted
- Fix: EKS access entry + policy association

### GitHub Actions: schedule + workflow_call conflict
- A workflow with `schedule` trigger cannot also be `workflow_call`
- Fix: split into `cost-control.yml` (schedule) and `cost-gate.yml` (workflow_call)

### Pricing API failure
- AWS Pricing API requires specific filter combinations, fragile
- Fix: fallback lookup table for common instance types

## Interview-Ready Statements

1. "I built a CI/CD system where cost is a first-class gate — the pipeline fetches live pricing from the AWS Pricing API and blocks deploy if projected spend would breach the monthly cap."

2. "I designed for failure: orphan detection catches when nightly destroy fails, preventing cost accumulation. A validation workflow runs 30 minutes after destroy to prove the account is clean."

3. "I separated the cost gate as a reusable workflow so any future deploy pipeline (EKS, EC2, Lambda) can call it with their instance type and get the same financial guardrails."

4. "The system has defense in depth: budget gate blocks new deploys, nightly cron destroys everything, validation proves it's dead, and the $70 hard cap is the final backstop."
