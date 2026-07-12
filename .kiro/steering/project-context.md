# OTel Learning Project — Steering

## What This Project Is

An iterative observability learning project for a platform engineer preparing for a Thursday interview. The goal is compressed understanding (not just a running system). The system is proof of understanding, not the deliverable itself.

Reference: #[[file:PROJECT_PLAN.md]]

## How to Work With the User

- **Iterative, not one-shot.** Never generate large amounts of code at once. Explain each step before moving on. Wait for acknowledgment.
- **Teach through building.** Every action should include: why this component exists, alternatives, trade-offs, common failure modes, and debugging commands.
- **Milestone-based.** Follow the milestone order in PROJECT_PLAN.md. Don't skip ahead.
- **Interview prep is the goal.** At the end of every milestone, ask interview questions as an Oracle Principal Engineer would.
- **Offer options.** Present design choices with pros/cons so the user can reason through them. Don't just pick for them.
- **Assume smart but new.** The user is an experienced cloud/platform engineer who hasn't used these specific tools (OTel, Prometheus, Grafana, K8s, CI/CD pipelines). Don't over-explain cloud basics. Do explain observability/K8s specifics.

## Architecture Progression

1. EC2 + Docker Compose (understand containers and data flow without K8s noise)
2. EKS migration (understand what orchestration solves)
3. Self-managed Prometheus/Grafana first, then AMP/AMG (understand what managed services abstract away)
4. Simple pipeline first, then production-grade (lint → test → security → deploy → smoke test)
5. Single-account first, then plug-and-play cross-account

## Technical Conventions

- **IaC:** Terraform for infrastructure, CloudFormation for CloudWatch alarms/monitoring
- **CI/CD:** GitHub Actions (YAML workflows in `.github/workflows/`)
- **Container orchestration:** Docker Compose on EC2, then Helm on EKS
- **Observability stack:** OpenTelemetry Collector → Prometheus (metrics) + Jaeger/Tempo (traces) → Grafana (visualization)
- **AWS IAM:** Dedicated `kiro-deploy` user with least-privilege scoped permissions
- **Cost control:** Scheduled scale-down, AWS Budgets with SNS alerts

## Repository Structure

- `docs/` — Living documentation (architecture, decisions, failure modes, data flow, milestone notes)
- `docs/milestones/` — Per-milestone learning notes with interview Q&A
- `infra/` — Terraform modules (added at milestone 6)
- `k8s/` — Kubernetes manifests and Helm values (added at milestone 5)
- `cloudwatch/` — CloudFormation templates (added at milestone 8)
- `.github/workflows/` — CI/CD pipelines (added at milestone 4)

## Documentation Requirements

Automatically maintain these living docs as we progress:
- `docs/ARCHITECTURE.md` — Updated each milestone with current state diagram
- `docs/DECISIONS.md` — Append every design decision with: choice, why, alternatives considered, trade-offs
- `docs/FAILURE-MODES.md` — Document what can break, how to detect it, how to fix it
- `docs/DATAFLOW.md` — How telemetry flows through the system (updated as architecture evolves)

## Pipeline Design Philosophy

- Start with the simplest thing that works, then evolve
- Later milestones introduce: linting, unit tests, security scanning (tfsec/cfn-nag), smoke tests, notifications
- The CloudWatch pipeline (milestone 8) is the showcase for full production-grade CI/CD
- Everything parameterized for cross-account portability

## What NOT to Do

- Don't deploy everything at once
- Don't skip explanations to save time
- Don't use root account for anything after IAM user is created
- Don't introduce managed services before self-managed equivalents are understood
- Don't create K8s resources before EC2-based understanding is solid
