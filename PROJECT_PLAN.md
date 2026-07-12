# OTel Learning Project — Master Plan

## Goal
Compress a week of learning (Prometheus, OpenTelemetry, Grafana, K8s, CI/CD) into under 2 days for a Thursday interview. Build iteratively, understand every component, not just deploy.

## Principles
- Iterative, not one-shot. Understand before moving on.
- EC2 first, then EKS (feel the pain, then understand what orchestration solves)
- Self-managed first, then managed (Prometheus/Grafana before AMP/AMG)
- Every design decision includes: why, alternatives, trade-offs, failure modes, debug commands
- Repository is a learning notebook with architecture diagrams, decision log, failure docs
- End of each milestone: interview questions from Oracle Principal Engineer perspective
- Pipeline starts simple, evolves to production-grade (lint, test, security, smoke test)
- Final state: plug-and-play cross-account deployment

## User Background
- Cloud automation / platform engineer with extensive cloud experience
- New to: app observability, K8s, OTel, Prometheus, Grafana, CI/CD pipelines
- Has: AWS root account (new, $100 free credits + $100 more), Git installed
- Needs to install: AWS CLI, kubectl, Helm, Terraform, Docker Desktop

## AWS Account State
- Brand new root account, payment method attached
- No IAM users created yet
- Plan: Create `kiro-deploy` IAM user for pipelines (least privilege)

## Milestones

### Milestone 1: Repo Setup + AWS Account Hardening
- Create repo structure with docs/learning notebook
- Create IAM user `kiro-deploy` with scoped permissions
- Set up AWS CLI locally
- Connect credentials to GitHub Secrets
- Understand: IAM, least privilege, credential management

### Milestone 2: EC2 + Docker Compose — OTel Demo
- Single EC2 instance running OTel Demo via Docker Compose
- All 15 microservices + collector + Prometheus + Grafana on one box
- Understand: containers, networking, OTel data model, docker-compose

### Milestone 3: Explore Prometheus, Grafana, Traces
- Navigate Grafana dashboards
- Understand Prometheus scrape configs, PromQL basics
- Trace a request through Jaeger/Tempo
- Understand: metrics vs traces vs logs, scraping, dashboards, data flow

### Milestone 4: GitHub Actions — Simple Deploy Pipeline
- Pipeline that deploys/updates the EC2 setup
- Start simple: lint → deploy
- Understand: CI/CD basics, secrets, triggers, workflow syntax

### Milestone 5: EKS Cluster + Migration
- Deploy EKS cluster
- Migrate OTel Demo from Docker Compose to Helm on EKS
- Understand: K8s concepts, why orchestration exists, Helm, service discovery

### Milestone 6: IaC — Terraform for Everything
- Codify what was built manually (VPC, EC2, EKS, IAM)
- Understand: state management, modules, plan/apply, drift detection

### Milestone 7: Cost Control Pipeline
- Scheduled GitHub Action: scale down at night, scale up on demand
- AWS Budgets via IaC with SNS alerts at 50%, 80%, 100%
- Understand: scheduled actions, budget APIs, guardrails

### Milestone 8: CloudWatch Monitoring (Full Pipeline)
- CFT stack deploying CloudWatch Alarms for observability services
- Full pipeline: lint → validate → unit test → security scan → deploy → smoke test
- Understand: observability of observability, CFT, alarm design

### Milestone 9: Self-Managed → AMP/AMG Migration
- Replace self-hosted Prometheus with Amazon Managed Prometheus
- Replace self-hosted Grafana with Amazon Managed Grafana
- Understand: managed vs self-hosted trade-offs, remote write, IAM roles

### Milestone 10: Plug-and-Play Cross-Account
- Parameterize everything (account ID, region, cluster name)
- Single workflow dispatch deploys entire stack to a fresh account
- Understand: multi-account strategy, parameterization, reusability

## Repository Structure
otel-learning/ ├── docs/ │ ├── ARCHITECTURE.md │ ├── DECISIONS.md │ ├── FAILURE-MODES.md │ ├── DATAFLOW.md │ └── milestones/ │ ├── 01-repository-setup.md │ ├── 02-ec2-otel-demo.md │ ├── 03-explore-observability.md │ └── ... ├── infra/ # Terraform (milestone 6) ├── k8s/ # K8s manifests/Helm values (milestone 5) ├── cloudwatch/ # CFT templates (milestone 8) ├── .github/workflows/ # CI/CD pipelines (milestone 4+) ├── PROJECT_PLAN.md # This file └── README.md



## Kiro Steering
Once in the workspace, set up `.kiro/steering/` with project conventions so Kiro maintains consistency across sessions.

## Current Status
- [ ] Milestone 1: Not started
- [ ] Milestone 2: Not started
- [ ] ...

## Next Action
Start Milestone 1: Create repo structure, create IAM user, install AWS CLI, configure credentials.