# Security Misconfiguration Lab Report

## Misconfigurations Identified

| # | Misconfiguration | Risk Level | Remediation | Status |
|---|-------------------|------------|--------------------------------------|--------|
| 1 | Public S3 bucket object (`test-file.txt` in `my-insecure-test-bucket-1787643884`) | Critical | Enabled Block Public Access, enabled AES256 default encryption | ✅ Fixed |
| 2 | SSH open to 0.0.0.0/0 (`insecure-ssh-sg` / `sg-046e4caee31b25674`) | Critical | Revoked 0.0.0.0/0 rule, restricted to single trusted IP (85.170.119.34/32) | ✅ Fixed |
| 3 | IAM role with AdministratorAccess (`InsecureAppRole`) | Critical | Detached AdministratorAccess, applied least-privilege inline policy (S3 GetObject/PutObject + scoped Secrets Manager access) | ✅ Fixed |

## Evidence

**Misconfiguration 1 — Public S3 Object**
- Before: `curl` to object URL returned `200` with object content readable
- After: `curl` to same URL returned `403`

**Misconfiguration 2 — Open SSH**
- Before: `describe-security-groups` showed ingress rule `0.0.0.0/0` on TCP/22
- After: `describe-security-groups` shows only `85.170.119.34/32` on TCP/22

**Misconfiguration 3 — Over-privileged IAM Role**
- Before: `list-attached-role-policies` showed `AdministratorAccess`
- After: `list-attached-role-policies` returns empty; `list-role-policies` shows inline policy `AppLeastPrivilegePolicy` scoped to specific S3 bucket and Secrets Manager path

## Detection Methods
- AWS Config rules (automated): `s3-bucket-public-read-prohibited`, `restricted-ssh`
- Manual review (IAM policies)

## Detection Gap Finding
- AWS Config rule `s3-bucket-public-read-prohibited` returned **COMPLIANT** even while the bucket contained a publicly-readable object (confirmed via `curl` → `200`).
- Root cause: this managed rule evaluates only bucket-level policy and bucket ACL, not individual object ACLs — the object's `public-read` ACL fell outside its detection scope.
- The `restricted-ssh` rule correctly flagged the open security group as **NON_COMPLIANT**.
- Implication: object-level public grants can bypass this specific Config rule entirely, even with all other detection in place.

## Lessons Learned
1. Default-deny is critical (Block Public Access)
2. Regular Config rule evaluation catches misconfigurations — but rule scope matters; a "passing" rule doesn't guarantee no exposure
3. IAM Access Analyzer should be enabled for ongoing monitoring
4. Object-level permissions (ACLs) need separate detection coverage from bucket-level checks

## Recommendations
1. Enable Security Hub for centralized findings
2. Implement infrastructure as code to enforce secure baselines
3. Conduct quarterly security reviews
4. Supplement `s3-bucket-public-read-prohibited` with S3 Access Analyzer or object-ACL auditing via S3 Inventory to close the detection gap identified above
