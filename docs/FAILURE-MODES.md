# Failure Modes

Documents what can break, how to detect it, and how to fix it. Updated each milestone.

---

## Milestone 1: Account & Repo Setup

### FM-001: GitHub push rejected — authentication failure

**Symptoms:** `git push` returns 403 or asks for password (which won't work)

**Cause:** GitHub removed password auth in 2021. Need token or credential manager.

**Fix:**
- Windows: Git Credential Manager should pop up a browser auth flow automatically
- If not: generate a Personal Access Token (PAT) at https://github.com/settings/tokens
- Use the PAT as the password when prompted

**Debug:**
```bash
git remote -v              # verify remote URL is correct
git config credential.helper   # check which credential helper is configured
```

---

### FM-002: AWS credentials invalid in GitHub Actions (future)

**Symptoms:** Pipeline fails with `InvalidClientTokenId` or `SignatureDoesNotMatch`

**Cause:** Typo when pasting into GitHub Secrets, or keys were rotated/deleted

**Fix:**
- Go to IAM → Users → kiro-deploy → Security credentials
- Verify the access key is active (not deleted or inactive)
- Re-check GitHub Secrets — delete and re-create if needed (you can't view existing secret values)

**Debug (in CloudShell):**
```bash
aws iam list-access-keys --user-name kiro-deploy
```

---

### FM-003: Accidentally committed secrets to git

**Symptoms:** Access keys visible in git history

**Cause:** Pasted keys into a file and committed

**Fix (nuclear option — rewrites history):**
```bash
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch <file-with-secrets>' \
  --prune-empty --tag-name-filter cat -- --all
git push --force
```
Then immediately rotate the keys in IAM.

**Prevention:** Add a `.gitignore` with common secret file patterns. Never put keys in code.

---

### FM-004: Promotional credits disappeared

**Symptoms:** Billing shows $0 credit balance unexpectedly

**Cause:** Enabled Organizations, Control Tower, or Identity Center

**Fix:** None — credits are non-recoverable once forfeited. Can try contacting AWS Support.

**Prevention:** Do not enable Organizations on this account. (See DEC-007)

---
