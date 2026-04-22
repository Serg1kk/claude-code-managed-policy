# Acme Corp — Managed Claude Code Policy

> This policy (managed CLAUDE.md) applies to ALL Acme Corp developers.
> On-disk paths:
> - macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`
> - Linux/WSL: `/etc/claude-code/CLAUDE.md`
> - Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`
>
> This file is managed by IT / Security via MDM. **Do not edit locally**: changes will be overwritten on the next MDM sync.

---

## Compliance context

Acme Corp operates in **SOC 2 Type II + PCI DSS + SOX-regulated** environment.
This Claude Code installation is **managed** by Acme IT / Security.
All activity is logged and reviewed per our Security Policy (sec.acme.com/ai-policy).

## Tech stack quick reference

- **Backend:** Go 1.22 / Java 21 + Spring Boot / Node.js 22
- **Frontend:** TypeScript + React 19, Next.js 15, Tailwind CSS 4
- **Databases:** PostgreSQL 16, Redis 7, MongoDB 7
- **Infra:** Kubernetes (EKS 1.30), Terraform 1.9, ArgoCD, Istio 1.24
- **Observability:** Datadog APM + Splunk Cloud + Grafana + PagerDuty
- **CI/CD:** GitHub Actions + ArgoCD + Spinnaker
- **Security:** Vault (HashiCorp) + Snyk + Trivy + SonarQube

## NEVER do (hard requirements — policy-enforced)

- ❌ Output API keys, tokens, passwords, connection strings in code
- ❌ Hard-code credentials — **always use Vault / env vars**
- ❌ Read `.env`, `./secrets/`, `~/.aws/`, `~/.ssh/` — **blocked at client level**
- ❌ Push to `main` / `master` / `release/*` directly — **must use PR workflow**
- ❌ Force-push (`git push --force`) — **blocked**
- ❌ Use `curl` / `wget` for arbitrary URLs — **blocked except allowlist**
- ❌ Install npm / pip packages without review — **requires `ask` approval**

## NEVER paste in prompts

- ❌ Customer PII (names + DOB + SSN / national IDs)
- ❌ Payment card numbers (PCI DSS scope)
- ❌ Personal Health Information (PHI — HIPAA scope for Healthcare BU)
- ❌ Internal infrastructure secrets (AWS account IDs, VPC CIDRs, Vault paths)
- ❌ Full database connection strings
- ❌ Client confidential data marked "Internal" or above
- ❌ Active session tokens or OAuth codes

If unsure — redact first. Use placeholders: `{{USER_ID}}`, `{{CARD_NUMBER}}`, `{{API_KEY}}`.

## ALWAYS when generating code

### Security
- Use environment variables for all configuration secrets (never inline)
- Add input validation before database queries (SQL injection prevention)
- Use parameterized queries — never string concatenation
- Add authentication middleware to all new endpoints
- Log errors but NEVER leak stack traces / internal paths to user-facing errors

### Code quality
- Include JSDoc / docstrings for public APIs
- Add unit tests for new business logic (80%+ coverage target)
- Follow conventional commits format: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`
- Reference Jira ticket in commit messages: `feat(ACM-1234): add user auth`

### PCI-specific (when touching payment-processing/)
- All cardholder data must flow through `payments-service`
- Never log card numbers (even masked) outside `payments-service`
- Use ONE-TIME-USE tokens, never raw PAN
- PR requires SecOps approval (automated via CODEOWNERS)

### PHI-specific (when touching healthcare-bu/)
- All PHI must be encrypted at rest (AES-256) and in transit (TLS 1.3)
- Access logs must include `actor`, `patient_id`, `fields_accessed`, `purpose`
- PR requires Compliance Officer approval

## Workflow conventions

### Branch naming
- `feature/ACM-{ticket}-{slug}` — feature work
- `fix/ACM-{ticket}-{slug}` — bugfix
- `chore/ACM-{ticket}-{slug}` — maintenance
- `release/v{major}.{minor}` — release branches (protected)

### PR requirements
- Minimum 2 approvers (1 SecOps for security-sensitive)
- All CI checks pass (linting, tests, SAST via Snyk, secret scan via TruffleHog)
- Jira ticket linked
- Description includes: what changed, why, testing notes, rollback plan for non-trivial changes

### Commit messages
```
feat(ACM-1234): add OAuth2 PKCE flow

- Implements RFC 7636 for native clients
- Adds PKCE challenge + verifier to auth endpoints
- Updates docs/auth.md

Closes ACM-1234
```

## Restricted code areas

Claude Code is **blocked** (via `permissions.deny`) from reading/editing:

| Path | Reason |
|---|---|
| `src/payment-processing/` | PCI-DSS scope — requires SecOps review, use @payments Slack |
| `compliance/regulated/` | Compliance-regulated — read-only for engineering |
| `infra/vault/` | Vault infrastructure — Platform team only |
| `secrets/`, `./.env*` | Contains secrets |

If user asks to work in restricted area — **decline and redirect:**
> "This area requires SecOps review. Please contact @security-team in Slack or file a ticket via sec.acme.com/help."

## When in doubt

- Security / privacy questions → `@security-team` in Slack, or sec.acme.com/ai-policy
- Architecture questions → Engineering Handbook at eng.acme.com/handbook
- Runtime / incident → PagerDuty on-call via oncall.acme.com
- Claude Code issues / policy feedback → `#claude-code-support` Slack

## Useful links (internal)

- Engineering Handbook: eng.acme.com/handbook
- Security Policy: sec.acme.com/ai-policy
- Architecture diagrams: arch.acme.com
- Runbooks: runbooks.acme.com
- Slack: `#claude-code-support`, `#security-team`, `#engineering`
- Incident response: incident.acme.com

## Policy version

- Managed policy version: **v1.0 (2026-04-21)**
- Maintained by: Acme IT Security (`security-engineering@acme-corp.com`)
- Source repo (internal): `https://github.acme.com/acme/claude-code-policy`
- Next review: 2026-07-21
