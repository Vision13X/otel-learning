# Decision Log

Format: Decision → Why → Alternatives Considered → Trade-offs

---

## DEC-001: Separate IAM users for human vs automation

**Decision:** Two IAM users — `kiro-deploy` (programmatic) and `vigne-admin` (console only)

**Why:** CloudTrail auditability. When things go wrong, you can immediately filter by principal and see whether the pipeline or the human caused the issue.

**Alternatives:**
- Single user with both access types (simpler, less auditable)
- IAM Identity Center / SSO (best practice, but requires Organizations which forfeits promotional credits)

**Trade-offs:** Slightly more setup. Two sets of permissions to manage. Worth it for the audit trail.

---

## DEC-002: IAM Groups over inline policies

**Decision:** Policies attached to groups (`pipeline-automation`, `human-admins`), users added to groups.

**Why:** Scalable, auditable, revocable at the group level. Industry standard.

**Alternatives:**
- Inline policies on users (quick and dirty, doesn't scale, hard to audit)
- Direct policy attachment on users (slightly better, still per-user management)

**Trade-offs:** Extra step to create groups. Pays off immediately when you add a second user or need to revoke.

---

## DEC-003: AdministratorAccess for human user, scoped policies for pipeline

**Decision:** `vigne-admin` gets AdministratorAccess. `kiro-deploy` gets scoped managed policies.

**Why:** The human user is exploring and learning — permission walls waste learning time. The pipeline user is where least-privilege matters because it's automated and credential-bearing.

**Alternatives:**
- Scope both users (safer, but frustrating for exploration)
- Admin both users (dangerous for automation credentials)

**Trade-offs:** If `vigne-admin` credentials leaked, it's full account access. Mitigated by: no access keys generated for this user (console + CloudShell only).

---

## DEC-004: No local AWS credentials — CloudShell for human CLI access

**Decision:** `vigne-admin` has no access keys. All CLI work done via CloudShell in the browser.

**Why:** Zero risk of credential leakage from local machine. Temporary credentials that expire when you close the tab.

**Alternatives:**
- Local `~/.aws/credentials` with named profiles (convenient, risk of leakage)
- IAM Identity Center SSO (best of both worlds, but requires Organizations)

**Trade-offs:** Can't run `aws` commands from local terminal or Kiro. Acceptable for this project.

---

## DEC-005: GitHub Secrets for pipeline credentials (not AWS Secrets Manager)

**Decision:** Store `kiro-deploy` access keys in GitHub repository secrets.

**Why:** GitHub Actions natively reads repository secrets. Secrets Manager costs $0.40/secret/month and adds unnecessary complexity — the pipeline runs in GitHub's infra, not in AWS.

**Alternatives:**
- AWS Secrets Manager ($0.80/month for 2 secrets, adds API call to pipeline)
- OIDC federation (no long-lived secrets at all, more complex setup)

**Trade-offs:** Long-lived access keys exist in GitHub. Mitigated by: scoped permissions on `kiro-deploy`, can rotate keys anytime, keys never on local machine.

---

## DEC-006: us-east-1 as deployment region

**Decision:** All resources deployed to `us-east-1`.

**Why:** Broadest AWS service availability. Most documentation and examples use it. No concerns about latency for a learning project.

**Alternatives:**
- Nearest region (lower latency for console/CloudShell)
- Multi-region (overkill for learning)

**Trade-offs:** None meaningful for this use case.

---

## DEC-007: Skip IAM Identity Center / AWS Organizations

**Decision:** Do not enable Organizations or Identity Center.

**Why:** AWS explicitly states that enabling Organizations forfeits promotional/free-tier credits immediately. Account has $100 + $100 in credits that would be lost.

**Alternatives:**
- Enable anyway and accept credit loss (bad idea)
- Use a separate paid account for Identity Center (overkill)

**Trade-offs:** Can't use SSO or centralized access management. Acceptable for single-account learning project.

---

## DEC-008: Public GitHub repository

**Decision:** Repository is public.

**Why:** Free unlimited GitHub Actions minutes for public repos. No secrets in code (they're in GitHub Secrets). The project is a learning artifact, not proprietary.

**Alternatives:**
- Private repo (limited free Actions minutes — 2000/month)

**Trade-offs:** Code is visible to anyone. No sensitive data in the repo, so no risk.

---
