# Milestone 1: Repository Setup + AWS Account Hardening

## Objective
Establish the project foundation — version control, AWS account security, credential management, and documentation structure.

## What We Did

### 1. AWS Account Hardening
- Created IAM user `kiro-deploy` (programmatic access only) for pipeline automation
- Created IAM user `vigne-admin` (console access only) for human exploration
- Created IAM groups:
  - `pipeline-automation` → scoped AWS managed policies (EC2, EKS, VPC, IAM, S3, CloudWatch, CloudFormation, Budgets)
  - `human-admins` → AdministratorAccess
- Decision: no access keys for human user — CloudShell only (DEC-004)
- Decision: no Identity Center — would forfeit promotional credits (DEC-007)

### 2. GitHub Repository
- Created public repo: https://github.com/Vision13X/otel-learning
- Stored `kiro-deploy` credentials as GitHub Secrets:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
  - `AWS_REGION` = `us-east-1`

### 3. Documentation Structure
- Living docs: ARCHITECTURE.md, DECISIONS.md, FAILURE-MODES.md, DATAFLOW.md
- Milestone-specific notes in docs/milestones/
- Steering file for Kiro context persistence

## Key Concepts Learned

### IAM Hierarchy
```
AWS Account (root)
  └── IAM Groups (policy containers)
        ├── pipeline-automation
        │     └── kiro-deploy (user)
        └── human-admins
              └── vigne-admin (user)
```

### Credential Types
| Type | Lifespan | Use Case |
|------|----------|----------|
| Root credentials | Permanent | Account recovery only |
| IAM access keys | Until rotated/deleted | Programmatic access (APIs, CLI, pipelines) |
| Console password | Until changed | Human browser login |
| CloudShell session | Temporary (auto-expires) | CLI in browser, no keys needed |
| STS tokens (future) | 1-12 hours | Assumed roles, OIDC federation |

### GitHub Secrets
- Encrypted at rest with libsodium sealed boxes
- Exposed only to workflow runs as environment variables
- Never printed in logs (auto-masked)
- Cannot be read back after creation (write-only from UI)

## Debugging Commands

```bash
# Check who you are (run in CloudShell)
aws sts get-caller-identity

# List users in your account
aws iam list-users

# See what groups a user belongs to
aws iam list-groups-for-user --user-name kiro-deploy

# See policies attached to a group
aws iam list-attached-group-policies --group-name pipeline-automation

# Check if access keys are active
aws iam list-access-keys --user-name kiro-deploy
```

## Interview Questions

*Pretend an Oracle Principal Engineer is asking you these:*

1. **Why did you separate human and machine IAM users?**
   - CloudTrail auditability. Filter by principal to distinguish automated vs manual actions. Also: blast radius — if pipeline creds leak, revoke one user without locking out human access.

2. **Why groups over direct policy attachment?**
   - Scales to N users. Revoke an entire class of access in one action. Audit one place instead of N user policies.

3. **Your pipeline has IAMFullAccess — isn't that dangerous?**
   - Yes. For a learning project it's acceptable. In production I'd scope to `iam:CreateRole`, `iam:AttachRolePolicy` for specific resource ARNs only. The principle is: start permissive to unblock, then tighten based on what's actually needed (permission boundaries, SCPs in Orgs).

4. **Why not OIDC federation for GitHub Actions?**
   - It's the better long-term solution (no long-lived keys). Requires creating an IAM OIDC provider and role trust policy. Planned for milestone 10 (cross-account). For now, static keys in GitHub Secrets are simpler and the account is short-lived.

5. **What happens if someone accidentally enables Organizations on this account?**
   - Promotional credits are immediately and irreversibly forfeited. There's no warning in the enable flow. Recovery requires AWS Support and isn't guaranteed.
