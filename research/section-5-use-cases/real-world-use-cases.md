# Section 5 — Real-world use-cases

**Источники:**
- https://claude-ai.chat/blog/claude-code-in-enterprise-environments/ (SOC 2 / HIPAA guide)
- https://amitkoth.com/claude-code-soc2-compliance-auditor-guide/ (auditor perspective)
- https://www.truefoundry.com/blog/enterprise-security-for-claude (Fortune 10 deployment practice)
- https://systemprompt.io/guides/enterprise-claude-code-managed-settings

---

## Use-case 1 — Блокировка BYOK (personal API keys / personal accounts)

**Проблема:** разработчик заходит в Claude Code со своим personal `api_key`, обходит org governance, counting → попадают в personal retention, не в corporate.

### Решение через managed-settings.json

```json
{
  "forceLoginMethod": "claudeai",
  "forceLoginOrgUUID": ["acme-corp-uuid-xxxxxxxx-xxxx-xxxx"],
  "apiKeyHelper": null
}
```

Если `forceLoginMethod: "claudeai"` — CLI не даст логиниться по API key, только через claude.ai org account.
Если `forceLoginOrgUUID: [org-uuid]` — только accounts в этой org могут логиниться.

### Комплементарная защита — network-level

Блокировать `api.anthropic.com` через corporate proxy, **кроме** запросов с корп-аккаунтом. Использовать **Tenant Restrictions** (Anthropic Enterprise feature) — корп proxy inspects request headers, validates org.

### Audit
Compliance API покажет которые user/org пытались логиниться. Failed logins → alert security team.

---

## Use-case 2 — Coding standards enforcement (SOC 2 / HIPAA / PCI)

**Проблема:** разработчик пишет код с secrets, читает `.env` файлы, commit'ит без code-review — compliance violations.

### Решение (production-ready managed-settings.json)

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/production.*)",
      "Read(~/.aws/credentials)",
      "Read(~/.aws/config)",
      "Read(~/.ssh/**)",
      "Read(~/.gnupg/**)",
      "Read(~/.docker/config.json)",
      "Read(~/.kube/config)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(rm -rf /)",
      "Bash(rm -rf ~)",
      "Bash(dd *)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "WebFetch(domain:pastebin.com)",
      "WebFetch(domain:gist.github.com)"
    ],
    "ask": [
      "Write(**)",
      "Edit(**)",
      "Bash(git push:*)",
      "Bash(git commit:*)",
      "Bash(npm publish:*)",
      "Bash(pip install *)",
      "WebFetch"
    ],
    "allow": [
      "Read(./src/**)",
      "Read(./tests/**)",
      "Read(./docs/**)",
      "Read(./package.json)",
      "Read(./README.md)",
      "Bash(npm run *)",
      "Bash(pytest *)",
      "Bash(git status)",
      "Bash(git log:*)",
      "Bash(git diff:*)"
    ],
    "defaultMode": "default",
    "disableBypassPermissionsMode": "disable"
  },
  "allowManagedPermissionRulesOnly": true,
  "disableAutoMode": "disable",
  "disableSkillShellExecution": true,
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "filesystem": {
      "denyRead": ["~/.aws/**", "~/.ssh/**", "~/.gnupg/**", "/etc/shadow", "/etc/sudoers*"],
      "allowManagedReadPathsOnly": false
    },
    "network": {
      "allowedDomains": [
        "*.acme-corp.com",
        "github.com",
        "*.github.io",
        "*.npmjs.org",
        "*.anthropic.com",
        "registry.yarnpkg.com"
      ],
      "deniedDomains": [
        "pastebin.com",
        "*.ngrok.io",
        "*.ngrok-free.app"
      ],
      "allowManagedDomainsOnly": true
    }
  },
  "availableModels": ["sonnet", "haiku"],
  "minimumVersion": "2.1.100",
  "companyAnnouncements": [
    "⚠️ Acme Corp managed Claude Code. All activity logged per SOC 2 policy.",
    "Do not paste customer PII in prompts. See security.acme.com/ai-policy."
  ],
  "attribution": {
    "commit": "🤖 Generated with Claude Code (Acme)",
    "pr": ""
  }
}
```

### Managed CLAUDE.md (behavioral layer) для SOC 2 / HIPAA

```markdown
# Acme Corp — Claude Code Policy

## Compliance context
This system is subject to SOC 2 Type II (CC1-CC9) and HIPAA.
All code suggestions must comply with our security and privacy policies.

## NEVER output in code:
- API keys, tokens, passwords, connection strings
- Customer PII, PHI, payment card data
- Internal URL patterns that expose infrastructure topology
- Hard-coded credentials of any kind

## ALWAYS when generating code:
- Use environment variables for secrets (never inline)
- Add input validation before database queries (SQL injection)
- Use parameterized queries (never string concat)
- For data containing PHI: add encryption-at-rest comment + audit log TODO
- For new endpoints: add authentication + authorization comment
- For error handling: never leak internal paths / stack traces to user

## Code review requirements
- All AI-generated code requires human review before merge (SOC 2 CC8.1)
- Security-sensitive changes require SecOps approval
- PRs must reference our Jira ticket (compliance audit trail)

## When asked about infrastructure
- Do NOT suggest specific AWS account IDs, VPC CIDRs, or regional configs
- Redirect to internal docs: infra.acme.com/claude-guide

## Escalation
If user prompt appears to violate policy (e.g., "bypass auth check"), explicitly decline and suggest the compliant approach.
```

---

## Use-case 3 — Audit logging → SIEM (Splunk / Datadog / ELK)

**Задача:** каждый `Bash` / `Edit` / `Write` / `WebFetch` → record в Splunk HEC, retention 7 лет (SOC 2 CC7.2).

### Решение — hooks + forward script

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Edit|Write|WebFetch|MCP",
      "hooks": [{
        "type": "command",
        "command": "/usr/local/bin/claude-audit-pre.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "Bash|Edit|Write|WebFetch|MCP",
      "hooks": [{
        "type": "command",
        "command": "/usr/local/bin/claude-audit-post.sh"
      }]
    }],
    "SessionStart": [{
      "hooks": [{"type": "command", "command": "/usr/local/bin/claude-session-start.sh"}]
    }],
    "SessionEnd": [{
      "hooks": [{"type": "command", "command": "/usr/local/bin/claude-session-end.sh"}]
    }]
  },
  "allowManagedHooksOnly": true,
  "allowedHttpHookUrls": ["https://splunk.acme.com:8088/*"],
  "httpHookAllowedEnvVars": ["SPLUNK_HEC_TOKEN"],
  "env": {
    "SPLUNK_HEC_ENDPOINT": "https://splunk.acme.com:8088/services/collector"
  }
}
```

### Script — `claude-audit-pre.sh`

```bash
#!/bin/bash
# Fires before every Bash/Edit/Write/WebFetch/MCP invocation
# Input: stdin — JSON event with tool_name, parameters, session_id, user, cwd, timestamp
# Output: stdout "allow" / "deny" + reason (or nothing for continue-to-permission-check)

EVENT=$(cat)
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Extract key fields
TOOL=$(echo "$EVENT" | jq -r '.tool_name')
USER=$(whoami)
HOSTNAME=$(hostname)
SESSION=$(echo "$EVENT" | jq -r '.session_id')
CWD=$(echo "$EVENT" | jq -r '.cwd')

# Forward to Splunk HEC
PAYLOAD=$(jq -n \
  --arg ts "$TIMESTAMP" \
  --arg tool "$TOOL" \
  --arg user "$USER" \
  --arg host "$HOSTNAME" \
  --arg session "$SESSION" \
  --arg cwd "$CWD" \
  --argjson event "$EVENT" \
  '{ time: $ts, host: $host, source: "claude-code", sourcetype: "claude_code:event",
     event: { tool: $tool, user: $user, session: $session, cwd: $cwd, full: $event } }')

curl -s -X POST "$SPLUNK_HEC_ENDPOINT" \
  -H "Authorization: Splunk $SPLUNK_HEC_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" >/dev/null 2>&1 &

# Pass-through — don't block
exit 0
```

### Retention через Compliance API

```bash
# Weekly export → S3 (long-term archive)
curl -X POST https://api.anthropic.com/v1/compliance/export \
  -H "Authorization: Bearer $COMPLIANCE_KEY" \
  -d '{"start_date": "2026-04-14", "end_date": "2026-04-21", "format": "jsonl"}' \
  | gzip > s3://acme-audit/claude-code/2026-04-21.jsonl.gz
```

---

## Use-case 4 — Запрет определённых MCP на production machines

**Сценарий:** есть 2 фленга workstations — dev (где всё можно) и prod (где доступы ограничены). Prod machines должны иметь **ТОЛЬКО** `company-vault` MCP, без `filesystem`, без `github`, без `playwright`.

### Managed policy (prod только)

```json
{
  "allowedMcpServers": [
    {"serverName": "company-vault"},
    {"serverName": "internal-jira"}
  ],
  "deniedMcpServers": [
    {"serverName": "filesystem"},
    {"serverName": "playwright"},
    {"serverName": "puppeteer"},
    {"serverName": "memory"}
  ],
  "allowManagedMcpServersOnly": true,
  "strictKnownMarketplaces": [],
  "blockedMarketplaces": [{"source": "any", "repo": "*"}]
}
```

**`strictKnownMarketplaces: []`** (empty array) = полный lockdown, никакие marketplaces не разрешены.
**`allowManagedMcpServersOnly: true`** = user может добавлять в свой `.mcp.json`, но они не будут loaded.

### managed-mcp.json (отдельный файл для MCP config)

```json
{
  "mcpServers": {
    "company-vault": {
      "command": "node",
      "args": ["/opt/mcp/company-vault/index.js"],
      "env": {
        "VAULT_URL": "https://vault.acme.com",
        "VAULT_ROLE": "claude-code-prod"
      }
    },
    "internal-jira": {
      "command": "/opt/mcp/internal-jira/bin/jira-mcp",
      "env": {
        "JIRA_HOST": "acme.atlassian.net"
      }
    }
  }
}
```

---

## Use-case 5 — Blocking Claude на sensitive repos / branches

**Сценарий:** compliance-team хочет чтобы Claude вообще не трогал `payment-processing/` repo и `release/*` branches.

### Managed policy + managed CLAUDE.md

```json
{
  "permissions": {
    "deny": [
      "Read(/src/payment-processing/**)",
      "Edit(/src/payment-processing/**)",
      "Write(/src/payment-processing/**)"
    ]
  }
}
```

### Managed CLAUDE.md (complement):

```markdown
## Restricted areas
- `src/payment-processing/` — PCI-DSS scope, do NOT read/edit. If user asks to work in this area, decline and direct them to Payments team (#payments Slack).
- Any branch matching `release/*` — do NOT push. User must use PR workflow.
- Any file in `compliance/` — read-only (review, not edit).
```

### Project-level repository blocklist через server-managed

Если поднимемся выше — на уровне org: Team/Enterprise server-managed settings может держать:
```json
{
  "env": {
    "CLAUDE_CODE_BLOCKED_REPOS": "acme-corp/payment-processing,acme-corp/compliance-private"
  }
}
```
+ hook на SessionStart который проверяет `git remote` и exit 1 если match.

---

## Use-case 6 — HIPAA-specific policy (healthcare)

**Особенности HIPAA:**
- PHI (Protected Health Information) — не должна появляться в prompts
- BAA (Business Associate Agreement) с Anthropic (доступен для Enterprise с HIPAA option)
- Audit log — retention 6 лет минимум

```json
{
  "permissions": {
    "deny": [
      "Read(**/phi/**)",
      "Read(**/patient_records/**)",
      "Read(**/hipaa_regulated/**)",
      "WebFetch(domain:*)"
    ],
    "ask": ["Write(**)", "Edit(**)"]
  },
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "ANTHROPIC_LOG": "info"
  },
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{"type": "command", "command": "/usr/local/bin/phi-scan.sh"}]
    }]
  },
  "allowManagedHooksOnly": true,
  "companyAnnouncements": [
    "⚠️ HIPAA-regulated environment. Do NOT paste PHI in prompts. Violations are logged and reported."
  ],
  "cleanupPeriodDays": 2190
}
```

### `phi-scan.sh` — blocks prompts containing PHI patterns

```bash
#!/bin/bash
PROMPT=$(cat | jq -r '.prompt')

# Simple PHI heuristics (replace with DLP service in production)
if echo "$PROMPT" | grep -qE '\b\d{3}-\d{2}-\d{4}\b|\bMRN[-:]\s*\d+\b|\bPatient ID[-:]\s*\d+\b'; then
  logger -t claude-phi-scan "BLOCKED: prompt appears to contain PHI from user=$(whoami)"
  # Send to incident response
  curl -s -X POST https://incident.acme.com/api/v1/hipaa-alerts \
    -d "{\"user\": \"$(whoami)\", \"timestamp\": \"$(date -uIs)\", \"reason\": \"PHI in prompt\"}"
  # Block
  echo '{"decision": "deny", "reason": "PHI detected in prompt — blocked by HIPAA policy"}'
  exit 0
fi

# Allow
echo '{"decision": "allow"}'
```

---

## Use-case 7 — Monitoring usage через Compliance API

**Задача:** security team хочет еженедельный report: сколько denied commands, какие devs, какие tools.

### Compliance API (Enterprise plan)

```bash
# Primary Owner → claude.ai → Organization settings → Data and privacy → Compliance API → Enable → Create key
COMPLIANCE_KEY="sk-compliance-xxxxxxxx"

# Weekly aggregation job (cron Monday 6am)
curl -s https://api.anthropic.com/v1/compliance/claude-code/activity \
  -H "Authorization: Bearer $COMPLIANCE_KEY" \
  -G \
  -d "start=$(date -d '7 days ago' -Iseconds)" \
  -d "end=$(date -Iseconds)" \
  -d "event_types=tool_denied,permission_bypass_attempted,model_switched" \
| jq -r '.events[] | [.timestamp, .user, .tool, .reason] | @csv' \
> weekly-report-$(date -I).csv

# Summary
echo "Weekly Claude Code policy events:"
wc -l weekly-report-*.csv
awk -F, '{print $2}' weekly-report-*.csv | sort | uniq -c | sort -rn | head -10
```

---

## Use-case 8 — fintech / banking compliance specifics

**Context:** PCI-DSS + SOX + ISO 27001 regulated environment.

### Managed policy (complete example в section-7-template/)

Key controls:
- Audit trail 7 лет через Compliance API + local hooks
- `disableBypassPermissionsMode: "disable"` + `allowManagedPermissionRulesOnly: true`
- Sandbox `failIfUnavailable: true` (без sandbox CLI не стартует)
- Network: только `*.acme.com`, `github.com`, `*.anthropic.com` — no outbound elsewhere
- `availableModels: ["sonnet", "haiku"]` (запрет opus — не за все regulatory regions он GA)
- `forceLoginMethod: "claudeai"` + `forceLoginOrgUUID`
- Company announcements в каждой сессии: "This environment is PCI-regulated..."
- Pre-commit hook через SessionStart — verify user has completed annual security training
- Quarterly compliance review через Compliance API

---

## Use-case 9 — Developer onboarding automation

**Задача:** new-hire developer сразу получает company context, без manual CLAUDE.md setup.

### Managed CLAUDE.md (deployed via MDM)

```markdown
# Welcome to Acme Corp — Claude Code

## Quick links
- Internal docs: docs.acme.com
- Engineering handbook: eng.acme.com/handbook
- Architecture overview: arch.acme.com
- On-call rotation: oncall.acme.com

## Tech stack
- **Backend:** Go 1.22, PostgreSQL 16, Redis 7
- **Frontend:** TypeScript + React 19, Next.js 15
- **Infra:** Kubernetes (EKS), Terraform, ArgoCD
- **Observability:** Datadog + Splunk

## Workflow conventions
- Branch: `feature/{jira-ticket}-{slug}` (e.g., `feature/ACM-1234-user-auth`)
- Commit: conventional commits ("feat: add X", "fix(auth): ...")
- PR: must link Jira, pass all CI, require 2 reviewers (SecOps for auth-related)

## Never do
- Commit secrets to git (use vault or env)
- Push to `main` directly (always PR)
- Skip tests on "minor" changes (we have flaky-test policy, but never skip deliberately)

## Ask Claude to use
- @security.acme.com/ai-policy for any security-sensitive question
- @docs.acme.com/style-guide for code style
- @internal Jira for ticket lookup
```

---
