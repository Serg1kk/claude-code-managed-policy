# Section 6 — Сравнение с другими AI IDE / coding tools

> **Язык:** [English](competitors.md) · Русский

**Источники:**
- https://cursor.com/docs/account/teams/enterprise-settings (Cursor Enterprise)
- https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies (Copilot)
- https://developers.openai.com/codex/enterprise (Codex Enterprise)
- https://developers.openai.com/codex/enterprise/managed-configuration/ (Codex requirements.toml)
- https://docs.windsurf.com/windsurf/enterprise-policies (Windsurf)
- https://codeium.com/security (Windsurf Enterprise deployment options)

---

## Big-picture табличка

| Capability | **Claude Code** | **Cursor Business/Enterprise** | **GitHub Copilot Business/Enterprise** | **OpenAI Codex Enterprise** | **Windsurf Enterprise** |
|---|---|---|---|---|---|
| **Device-level policy file** | ✅ `managed-settings.json` (JSON, OS-level path) | ⚠️ MDM controls (team IDs, extensions) + Admin Dashboard Team Rules | ❌ Policy через GitHub web UI only | ✅ `requirements.toml` + `managed_config.toml` (local TOML file) | ⚠️ ADMX/plist/JSON (но policy контролирует VS Code fork settings, не AI rules) |
| **Behavioral memory at org-level** | ✅ Managed CLAUDE.md | ⚠️ Team Rules (editor-level, enforceable optional) | ⚠️ `.github/copilot-instructions.md` (repo-level, не device-level) | ⚠️ AGENTS.md (repo), managed-config.toml (device) | ⚠️ `.windsurf/rules/*.md` repo + Team Rules |
| **MDM delivery mechanism** | ✅ Jamf / Kandji / Intune / Group Policy (official templates) | ⚠️ MDM deploys editor, не rules | ❌ Нет — policy management only через GitHub web admin | ⚠️ macOS MDM для `requirements.toml` | ✅ Group Policy ADMX/ADML + MDM + Configuration Profiles |
| **Hard-enforce tool permissions** | ✅ `permissions.deny` hardcoded on client (Bash/Read/Write/WebFetch/MCP/Agent) | ⚠️ Privacy Mode enforce + repo blocklist, но не fine-grained tool deny | ⚠️ Content exclusions + file-level deny (e.g., `.env` blocked по org policy) | ✅ `requirements.toml` `allowed_approval_policies`, `sandbox_modes`, `mcp_servers` allowlist | ⚠️ Team-wide command allowlist/denylist через Admin Portal (не client-enforced) |
| **Cannot bypass via CLI flag** | ✅ `disableBypassPermissionsMode: disable` → `--dangerously-skip-permissions` rejected | ⚠️ Privacy Mode нельзя отключить user'ом | ⚠️ Model policy enforced server-side | ✅ `requirements.toml` takes precedence over CLI `--config` | ⚠️ Admin sets max auto-exec level, user не превысит |
| **Cannot bypass via settings override** | ✅ `--setting-source local` НЕ отключает managed (bug #11872 resolved as "by design") | ⚠️ Project rules не override team rules | ❓ Не доступно public doc | ✅ cloud-managed > macOS MDM > managed_config.toml > user config | ✅ Policy overrides workspace + user level |
| **Block BYOK / personal API keys** | ✅ `forceLoginMethod: claudeai` + `forceLoginOrgUUID` | ✅ Enforce team org via SCIM | ✅ SCIM + enterprise license required | ✅ ChatGPT Business/Enterprise RBAC | ✅ SSO + SCIM + zero-data retention |
| **Model allowlist** | ✅ `availableModels: [sonnet, haiku]` (managed-set) | ⚠️ UI-level model picker, team admin toggles | ✅ Model allowlist in org policy (Copilot Policies page) | ✅ Model access per RBAC role + requirements.toml | ⚠️ UI-level toggle в Admin Portal |
| **Plugin / MCP marketplace control** | ✅ `strictKnownMarketplaces: [...]` (managed-only) | ❌ Marketplace ещё не существует | ⚠️ MCP registry (GA), через Copilot Policies page | ✅ MCP allowlist в `requirements.toml` | ⚠️ Extension allowlist via GPO ADMX |
| **Audit logging** | ✅ Compliance API (Enterprise) + local hooks | ✅ Audit dashboards, Admin API, AI Code Tracking API | ✅ GitHub Audit Log + Enterprise analytics | ✅ ChatGPT Enterprise audit log + Codex-specific events | ✅ Admin Portal analytics + OTel export |
| **Sandbox / isolation** | ✅ macOS/Linux/WSL sandbox с managed fail-closed option | ⚠️ Agent Sandbox Mode (enforce org-wide Enterprise) | ⚠️ Codespaces sandbox | ✅ Sandbox modes enforced через requirements.toml | ⚠️ VM sandbox через Cascade |
| **Telemetry enforce on/off** | ✅ `env.CLAUDE_CODE_ENABLE_TELEMETRY` managed-enforce, OTel endpoint control | ✅ Privacy Mode org-wide | ✅ Codespaces telemetry | ✅ OTel через managed config | ✅ Admin Portal OTel config |
| **On-premise / self-hosted** | ⚠️ Bedrock/Vertex/Foundry proxy для model, но CLI всё равно talks в cloud | ❌ Cloud only | ⚠️ GitHub AE (enterprise cloud, нет on-prem для agent) | ❌ Cloud only | ✅ Enterprise Hybrid (CPU/storage tenant) + Enterprise Self-hosted (GPU-enabled) tiers |
| **FedRAMP / highest compliance** | ✅ SOC 2 T2 + ISO 27001 + ISO 42001 + HIPAA BAA | ✅ SOC 2 T2 | ✅ SOC 2 T2 + FedRAMP Moderate + IP indemnity | ✅ SOC 2 T2 (ChatGPT Enterprise) | ✅ **FedRAMP High** (unique) + SOC 2 T2 |
| **Per-group / per-team policy** | ❌ Server-managed uniform per org (limitation v2.1.x) | ✅ Scope per team / SCIM group | ✅ Per org + per enterprise | ✅ Per group (requirements.toml admin-set per group) | ✅ Per team through Admin Portal |

---

## Детали per product

### Cursor Business / Enterprise

**Key mechanism:**
- **Admin Dashboard** — cloud-hosted, SAML SSO, SCIM
- **Team Rules (Enforceable + Optional)** — пишутся в web dashboard, пушатся в Cursor client через сервис
- **MDM policies** — enforce allowed team IDs, extensions на user devices (**не контент rules**)
- **Hooks for Logging/Auditing** — MDM Distribution (Enterprise) vs. **MDM & Server-side distribution** (Enterprise)
- **Privacy Mode** — Enterprise может enforce org-wide (zero retention на Cursor servers)
- **Repository Blocklist** (Enterprise only)
- **Agent Sandbox Mode** — enforce org-wide (Enterprise)
- **SCIM distribution & access gating** (Enterprise)

**Что слабее Claude Code:**
- Нет fine-grained tool deny (Bash/Read/Write/WebFetch) на client level как hard gate
- Policy распространяется через cloud admin dashboard + MDM. Нет офлайн file-based machinery.

**Что сильнее:**
- Per-team scope из коробки (SCIM group mapping)
- AI Code Tracking API для analytics
- Репозиторный blocklist (Claude Code не имеет)

**Цена:** $40/user/month Business, Enterprise custom pricing.

### GitHub Copilot Business / Enterprise

**Key mechanism:**
- **GitHub web admin** → Enterprise / Organization → Copilot → Policies page
- **Policies** controlled в GitHub UI, применяются к Copilot license holders
- **Content Exclusion** — file-level deny (через `.github/copilot-exclude.yml` или org-level settings)
- **IP indemnity** (Business + Enterprise) — GitHub assumes legal liability
- **Audit Log** — через GitHub Enterprise Audit Log + specific Copilot events
- **SAML SSO, SCIM** (Business+)
- **Knowledge Bases** (Enterprise only) — index repos для codebase-aware chat
- **Model policy** — org-admin allowlist models

**Что слабее Claude Code:**
- **Нет device-level policy file** — всё через GitHub web (admin-only)
- **Нет fine-grained tool deny** как у Claude — Copilot exclusions file-level только
- **Политики только через web** — нельзя version-control policy в git как `managed-settings.json`

**Что сильнее:**
- **IP indemnity** — Claude Code не имеет (юридическая защита при AI-генерированном коде)
- **Deep GitHub integration** — Copilot видит repo context, Actions, PRs, Issues native
- Дешевле ($19/seat/month Business, $39 Enterprise — vs Cursor $40, Claude Code Enterprise custom)

**Цена:** $19 Business, $39 Enterprise.

### OpenAI Codex Enterprise

**Key mechanism (очень близко к Claude Code!):**
- **`requirements.toml`** — admin-enforced constraints (approval policy, sandbox mode, web search, MCP allowlist)
- **`managed_config.toml`** — managed defaults, merges on top of user's `config.toml`, takes precedence over CLI `--config`
- **Cloud-managed** (ChatGPT Business/Enterprise) — `requirements.toml` deploys через admin Policies page
- **macOS MDM** (managed preferences) — highest precedence для `requirements.toml` + `managed_config.toml`
- **Per-group policies** — different requirements для different user groups ✅ (**круче чем Claude Code server-managed uniform**)
- **RBAC** — Codex Admin group, Codex Users group через ChatGPT Enterprise RBAC
- **Smart approvals** — AI proposes `prefix_rule` для allowed commands

**`requirements.toml` пример:**
```toml
enforce_residency = "us"
allowed_approval_policies = ["on-request"]
allowed_sandbox_modes = ["workspace-write", "read-only"]
mcp_servers = ["github", "company-vault"]
```

**Что слабее Claude Code:**
- Нет публичных MDM templates для Jamf/Kandji/Intune (только macOS managed preferences generic)
- Нет `managed-settings.d/` drop-in directory (модульность)
- Нет Managed CLAUDE.md equivalent (AGENTS.md — это repo-level, а не device-level enforceable behavioral layer)

**Что сильнее:**
- **Per-group policies в cloud-managed** (Codex has this, Claude Code server-managed не имеет)
- **Explicit "rules" concept** (`.rules` files) — отдельный formal механизм для sandbox escape rules
- TOML format — более читаемый для IT-админов (не JSON)

### Windsurf (Codeium) Enterprise

**Key mechanism (больше IDE-центрично, менее AI-центрично):**
- **ADMX/ADML templates** для Windows Group Policy (Windsurf читает `HKLM\Software\Policies\Windsurf\{ProductName}`)
- **Configuration Profiles** на macOS (через MDM)
- **JSON policy files** на Linux
- **Admin Portal** — feature toggles (Web Search, MCP, Deploys), analytics, RBAC
- **Team-wide command allowlist/denylist** — applies to all team members (через Admin Portal)
- **Admin-Controlled Maximum Auto-Execution Level** (Teams/Enterprise) — admin sets max, users can downgrade but not exceed
- **Enterprise Hybrid** tier — CPU-and-storage-only tenant (Anthropic-сравнимо OK только Bedrock/Vertex proxy)
- **Enterprise Self-hosted** tier — GPU-enabled tenant managed by customer
- **FedRAMP High** accreditation (unique)

**Что слабее Claude Code:**
- Policy в основном контролирует **IDE (VS Code fork) settings**, не AI behavior rules
- Нет native fine-grained tool deny для AI agent (как `permissions.deny`)
- `.windsurfrules` / `.windsurf/rules/` — это Repo-level конвенция, не centrally enforceable

**Что сильнее:**
- **FedRAMP High** (единственный в этой группе)
- **Enterprise Self-hosted** tier (полный control)
- **Admin-controlled max auto-exec level** — granular

---

## Summary framework

**What to look for in any AI IDE when evaluating for enterprise:**

### Tier 1 — must have
1. **SSO / SCIM** — обязательный минимум для корп-auth
2. **Block BYOK** — force-login в org аккаунт
3. **Audit logging** — что-то для compliance (API / dashboard)
4. **SOC 2 Type II** — базовая compliance certification

### Tier 2 — важно
5. **Hard permission deny** — fine-grained tool/command block на client level (не только privacy mode on/off)
6. **Managed policy file** — version-control policies в git, rollback через CI/CD
7. **MDM delivery** — Jamf / Kandji / Intune / Group Policy native
8. **Model allowlist** — restrict which models доступны

### Tier 3 — nice to have
9. **MCP/plugin allowlist** — для экосистемы расширений
10. **Per-group policies** — разные группы → разные правила
11. **Hooks для custom audit** — integration с Splunk/Datadog
12. **Sandbox fail-closed** — hard-gate для prod environment

### Tier 4 — edge cases
13. **IP indemnity** — для особо-regulated verticals (legal protection)
14. **On-prem / self-hosted** — для air-gapped / FedRAMP High
15. **FedRAMP certification** — federal contracts
16. **Per-team RBAC** — large orgs (1000+ devs)

---

## Ranking для типовой fintech команды (60 devs)

| Requirement | Claude Code | Cursor | Copilot | Codex | Windsurf |
|---|---|---|---|---|---|
| SSO / SCIM | ✅ | ✅ | ✅ | ✅ | ✅ |
| Block BYOK | ✅ | ✅ | ✅ | ✅ | ✅ |
| Hard tool deny | ⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐ |
| Managed policy file | ⭐⭐⭐ | ⭐ | ❌ | ⭐⭐⭐ | ⭐⭐ |
| MDM delivery | ⭐⭐⭐ | ⭐⭐ | ❌ | ⭐⭐ | ⭐⭐⭐ |
| Model allowlist | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Audit → SIEM | ⭐⭐⭐ (Compliance API + hooks) | ⭐⭐ (dashboards) | ⭐⭐⭐ (GitHub Audit Log) | ⭐⭐ | ⭐⭐ (OTel) |
| MCP control | ⭐⭐⭐ | ❌ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Per-group | ❌ (limitation) | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| FedRAMP High | ❌ | ❌ | ❌ | ❌ | ⭐⭐⭐ |
| IP indemnity | ❌ | ❌ | ⭐⭐⭐ | ? | ? |
| Self-hosted | ❌ | ❌ | ❌ | ❌ | ⭐⭐⭐ |

**Где Claude Code выигрывает для fintech:**
- Самая детальная tool-level enforcement (permissions.deny hard-gate)
- Managed CLAUDE.md + managed-settings.json two-layer architecture
- Drop-in `managed-settings.d/` modular policy composition
- Official MDM templates для всех major platforms

**Где Claude Code проигрывает:**
- Per-group policies (всем — или никому)
- Нет IP indemnity
- Нет FedRAMP High / Self-hosted tier

**Recommendation for fintech / regulated orgs:** Claude Code для большинства fintech use-cases is best-in-class через managed-settings.json + MDM, но если org требует FedRAMP High или regulated air-gap — рассмотреть Windsurf Enterprise Self-hosted как альтернативу или complement.
