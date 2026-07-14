---
inclusion: auto
---

# Session Facts — DO NOT ASSUME, ASK IF MISSING

## AWS Account
- Account ID: 835050295936
- Region: us-east-1

## IAM Users
- Pipeline user: kiro-deploy (programmatic only, keys in GitHub Secrets)
- Human user: vision-admin (console + CloudShell, NO access keys)
- CloudShell identity ARN: arn:aws:iam::835050295936:user/vision-admin

## EKS Cluster
- Cluster name: otel-learning
- Instance type: t3.large
- Node count: 2
- Cluster creator: kiro-deploy (via GitHub Actions)
- CloudShell user access: GRANTED (access entry + AmazonEKSClusterAdminPolicy)

## GitHub
- Username: Vision13X
- Repo: Vision13X/otel-learning
- Secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION

## IPs / URLs
- EC2 instance (Milestone 2): TERMINATED
- EKS LoadBalancer URL: UNKNOWN — ask user

## Rules for Me
- NEVER hardcode account IDs, ARNs, or IPs without confirmed source
- If a value is marked UNKNOWN, ASK the user before proceeding
- If deploying a pipeline step that references a specific resource, parameterize it
- Use GitHub Secrets or workflow inputs for anything environment-specific
- ALWAYS mark milestones as complete and write milestone notes when boundary is crossed
- Detect milestone completion by: user says "done", "next", or moves to new topic
- When marking a milestone complete: update PROJECT_PLAN.md status + create docs/milestones/XX-*.md
