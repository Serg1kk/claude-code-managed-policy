# Section 6 — Comparison with other AI IDE / coding tools

> **Language:** English · [Русский](competitors.ru.md)

**Sources:**
- https://cursor.com/docs/account/teams/enterprise-settings (Cursor Enterprise)
- https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies (Copilot)
- https://developers.openai.com/codex/enterprise (Codex Enterprise)
- https://developers.openai.com/codex/enterprise/managed-configuration/ (Codex requirements.toml)
- https://docs.windsurf.com/windsurf/enterprise-policies (Windsurf)
- https://codeium.com/security (Windsurf Enterprise deployment options)

---

## Big-picture table

| Capability | **Claude Code** | **Cursor Business/Enterprise** | **GitHub Copilot Business/Enterprise** | **OpenAI Codex Enterprise** | **Windsurf Enterprise** |
|---|---|---|---|---|---|
| **Device-level policy file** | ✅ `managed-settings.json` (JSON, OS-level path) | ⚠️ MDM controls (team IDs, extensions) + Admin Dashboard Team Rules | ❌ Policy via GitHub web UI only | ✅ `requirements.toml` + `managed_config.toml` (local TOML file) | ⚠️ ADMX/plist/JSON (but policy controls VS Code fork settings, not AI rules) |
| **Behavioral memory at org-level** | ✅ Managed CLAUDE.md | ⚠️ Team Rules (editor-level, enforceable optional) | ⚠️ `.github/copilot-instructions.md` (repo-level, not device-level) | ⚠️ AGENTS.md (repo), managed-config.toml (device) | ⚠️ `.windsurf/rules/*.md` repo + Team Rules |
| **MDM delivery mechanism** | ✅ Jamf / Kandji / Intune / Group Policy (official templates) | ⚠️ MDM deploys editor, not rules | ❌ None: policy management is only via GitHub web admin | ⚠️ macOS MDM for `requirements.toml` | ✅ Group Policy ADMX/ADML + MDM + Configuration Profiles |
| **Hard-enforce tool permissions** | ✅ `permissions.deny` hardcoded on client (Bash/Read/Write/WebFetch/MCP/Agent) | ⚠️ Privacy Mode enforce + repo blocklist, but no fine-grained tool deny | ⚠️ Content exclusions + file-level deny (e.g., `.env` blocked via org policy) | ✅ `requirements.toml` `allowed_approval_policies`, `sandbox_modes`, `mcp_servers` allowlist | ⚠️ Team-wide command allowlist/denylist via Admin Portal (not client-enforced) |
| **Cannot bypass via CLI flag** | ✅ `disableBypassPermissionsMode: disable` → `--dangerously-skip-permissions` rejected | ⚠️ Privacy Mode cannot be disabled by user | ⚠️ Model policy enforced server-side | ✅ `requirements.toml` takes precedence over CLI `--config` | ⚠️ Admin sets max auto-exec level, user cannot exceed it |
| **Cannot bypass via settings override** | ✅ `--setting-source local` does NOT disable managed (bug #11872 resolved as "by design") | ⚠️ Project rules cannot override team rules | ❓ Not available in public docs | ✅ cloud-managed > macOS MDM > managed_config.toml > user config | ✅ Policy overrides workspace + user level |
| **Block BYOK / personal API keys** | ✅ `forceLoginMethod: claudeai` + `forceLoginOrgUUID` | ✅ Enforce team org via SCIM | ✅ SCIM + enterprise license required | ✅ ChatGPT Business/Enterprise RBAC | ✅ SSO + SCIM + zero-data retention |
| **Model allowlist** | ✅ `availableModels: [sonnet, haiku]` (managed-set) | ⚠️ UI-level model picker, team admin toggles | ✅ Model allowlist in org policy (Copilot Policies page) | ✅ Model access per RBAC role + requirements.toml | ⚠️ UI-level toggle in Admin Portal |
| **Plugin / MCP marketplace control** | ✅ `strictKnownMarketplaces: [...]` (managed-only) | ❌ Marketplace does not yet exist | ⚠️ MCP registry (GA), via Copilot Policies page | ✅ MCP allowlist in `requirements.toml` | ⚠️ Extension allowlist via GPO ADMX |
| **Audit logging** | ✅ Compliance API (Enterprise) + local hooks | ✅ Audit dashboards, Admin API, AI Code Tracking API | ✅ GitHub Audit Log + Enterprise analytics | ✅ ChatGPT Enterprise audit log + Codex-specific events | ✅ Admin Portal analytics + OTel export |
| **Sandbox / isolation** | ✅ macOS/Linux/WSL sandbox with managed fail-closed option | ⚠️ Agent Sandbox Mode (enforce org-wide Enterprise) | ⚠️ Codespaces sandbox | ✅ Sandbox modes enforced via requirements.toml | ⚠️ VM sandbox via Cascade |
| **Telemetry enforce on/off** | ✅ `env.CLAUDE_CODE_ENABLE_TELEMETRY` managed-enforce, OTel endpoint control | ✅ Privacy Mode org-wide | ✅ Codespaces telemetry | ✅ OTel via managed config | ✅ Admin Portal OTel config |
| **On-premise / self-hosted** | ⚠️ Bedrock/Vertex/Foundry proxy for model, but CLI still talks to cloud | ❌ Cloud only | ⚠️ GitHub AE (enterprise cloud, no on-prem for agent) | ❌ Cloud only | ✅ Enterprise Hybrid (CPU/storage tenant) + Enterprise Self-hosted (GPU-enabled) tiers |
| **FedRAMP / highest compliance** | ✅ SOC 2 T2 + ISO 27001 + ISO 42001 + HIPAA BAA | ✅ SOC 2 T2 | ✅ SOC 2 T2 + FedRAMP Moderate + IP indemnity | ✅ SOC 2 T2 (ChatGPT Enterprise) | ✅ **FedRAMP High** (unique) + SOC 2 T2 |
| **Per-group / per-team policy** | ❌ Server-managed uniform per org (limitation v2.1.x) | ✅ Scope per team / SCIM group | ✅ Per org + per enterprise | ✅ Per group (requirements.toml admin-set per group) | ✅ Per team through Admin Portal |

---

## Per-product details

### Cursor Business / Enterprise

**Key mechanism:**
- **Admin Dashboard**: cloud-hosted, SAML SSO, SCIM
- **Team Rules (Enforceable + Optional)**: authored in the web dashboard, pushed to the Cursor client via service
- **MDM policies**: enforce allowed team IDs and extensions on user devices (**not content rules**)
- **Hooks for Logging/Auditing**: MDM Distribution (Enterprise) vs. **MDM & Server-side distribution** (Enterprise)
- **Privacy Mode**: Enterprise can enforce org-wide (zero retention on Cursor servers)
- **Repository Blocklist** (Enterprise only)
- **Agent Sandbox Mode**: enforce org-wide (Enterprise)
- **SCIM distribution & access gating** (Enterprise)

**Where it is weaker than Claude Code:**
- No fine-grained tool deny (Bash/Read/Write/WebFetch) at the client level as a hard gate
- Policy is distributed via cloud admin dashboard + MDM. No offline file-based machinery.

**Where it is stronger:**
- Per-team scope out of the box (SCIM group mapping)
- AI Code Tracking API for analytics
- Repository blocklist (Claude Code does not have this)

**Price:** $40/user/month Business, Enterprise custom pricing.

### GitHub Copilot Business / Enterprise

**Key mechanism:**
- **GitHub web admin** → Enterprise / Organization → Copilot → Policies page
- **Policies** controlled in the GitHub UI, applied to Copilot license holders
- **Content Exclusion**: file-level deny (via `.github/copilot-exclude.yml` or org-level settings)
- **IP indemnity** (Business + Enterprise): GitHub assumes legal liability
- **Audit Log**: via GitHub Enterprise Audit Log + Copilot-specific events
- **SAML SSO, SCIM** (Business+)
- **Knowledge Bases** (Enterprise only): index repos for codebase-aware chat
- **Model policy**: org-admin allowlist of models

**Where it is weaker than Claude Code:**
- **No device-level policy file**: everything is routed through GitHub web (admin-only)
- **No fine-grained tool deny** like Claude Code has. Copilot exclusions are file-level only.
- **Policies are web-only**: you cannot version-control the policy in git the way you can `managed-settings.json`

**Where it is stronger:**
- **IP indemnity**: Claude Code does not offer this (legal protection for AI-generated code)
- **Deep GitHub integration**: Copilot sees repo context, Actions, PRs, and Issues natively
- Cheaper ($19/seat/month Business, $39 Enterprise vs. Cursor $40, Claude Code Enterprise custom)

**Price:** $19 Business, $39 Enterprise.

### OpenAI Codex Enterprise

**Key mechanism (very close to Claude Code):**
- **`requirements.toml`**: admin-enforced constraints (approval policy, sandbox mode, web search, MCP allowlist)
- **`managed_config.toml`**: managed defaults, merges on top of the user's `config.toml`, takes precedence over CLI `--config`
- **Cloud-managed** (ChatGPT Business/Enterprise): `requirements.toml` is deployed through the admin Policies page
- **macOS MDM** (managed preferences): highest precedence for `requirements.toml` + `managed_config.toml`
- **Per-group policies**: different requirements for different user groups ✅ (**better than Claude Code's uniform server-managed model**)
- **RBAC**: Codex Admin group, Codex Users group through ChatGPT Enterprise RBAC
- **Smart approvals**: AI proposes a `prefix_rule` for allowed commands

**`requirements.toml` example:**
```toml
enforce_residency = "us"
allowed_approval_policies = ["on-request"]
allowed_sandbox_modes = ["workspace-write", "read-only"]
mcp_servers = ["github", "company-vault"]
```

**Where it is weaker than Claude Code:**
- No public MDM templates for Jamf/Kandji/Intune (only generic macOS managed preferences)
- No `managed-settings.d/` drop-in directory (modularity)
- No Managed CLAUDE.md equivalent (AGENTS.md is repo-level, not a device-level enforceable behavioral layer)

**Where it is stronger:**
- **Per-group policies in cloud-managed** mode (Codex has this; Claude Code's server-managed mode does not)
- **Explicit "rules" concept** (`.rules` files): a separate formal mechanism for sandbox escape rules
- TOML format: more readable for IT admins than JSON

### Windsurf (Codeium) Enterprise

**Key mechanism (more IDE-centric, less AI-centric):**
- **ADMX/ADML templates** for Windows Group Policy (Windsurf reads `HKLM\Software\Policies\Windsurf\{ProductName}`)
- **Configuration Profiles** on macOS (via MDM)
- **JSON policy files** on Linux
- **Admin Portal**: feature toggles (Web Search, MCP, Deploys), analytics, RBAC
- **Team-wide command allowlist/denylist**: applies to all team members (via Admin Portal)
- **Admin-Controlled Maximum Auto-Execution Level** (Teams/Enterprise): admin sets the max, users can downgrade but not exceed
- **Enterprise Hybrid** tier: CPU-and-storage-only tenant (Anthropic comparable only to Bedrock/Vertex proxy)
- **Enterprise Self-hosted** tier: GPU-enabled tenant managed by the customer
- **FedRAMP High** accreditation (unique)

**Where it is weaker than Claude Code:**
- Policy primarily controls **IDE (VS Code fork) settings**, not AI behavior rules
- No native fine-grained tool deny for the AI agent (equivalent to `permissions.deny`)
- `.windsurfrules` / `.windsurf/rules/` is a repo-level convention, not centrally enforceable

**Where it is stronger:**
- **FedRAMP High** (the only one in this group)
- **Enterprise Self-hosted** tier (full control)
- **Admin-controlled max auto-exec level**: granular

---

## Summary framework

**What to look for in any AI IDE when evaluating for enterprise:**

### Tier 1 — must have
1. **SSO / SCIM**: mandatory minimum for corporate auth
2. **Block BYOK**: force-login to an org account
3. **Audit logging**: something for compliance (API / dashboard)
4. **SOC 2 Type II**: baseline compliance certification

### Tier 2 — important
5. **Hard permission deny**: fine-grained tool/command block at the client level (not just privacy mode on/off)
6. **Managed policy file**: version-control policies in git, rollback through CI/CD
7. **MDM delivery**: Jamf / Kandji / Intune / Group Policy native
8. **Model allowlist**: restrict which models are available

### Tier 3 — nice to have
9. **MCP/plugin allowlist**: for the extensions ecosystem
10. **Per-group policies**: different groups → different rules
11. **Hooks for custom audit**: integration with Splunk/Datadog
12. **Sandbox fail-closed**: hard gate for prod environments

### Tier 4 — edge cases
13. **IP indemnity**: for heavily regulated verticals (legal protection)
14. **On-prem / self-hosted**: for air-gapped / FedRAMP High
15. **FedRAMP certification**: federal contracts
16. **Per-team RBAC**: large orgs (1000+ devs)

---

## Ranking for a typical fintech team (60 devs)

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

**Where Claude Code wins for fintech:**
- The most detailed tool-level enforcement (`permissions.deny` hard gate)
- Managed CLAUDE.md + managed-settings.json two-layer architecture
- Drop-in `managed-settings.d/` modular policy composition
- Official MDM templates for all major platforms

**Where Claude Code loses:**
- Per-group policies (all or nothing)
- No IP indemnity
- No FedRAMP High / Self-hosted tier

**Recommendation for fintech / regulated orgs:** Claude Code is best-in-class for most fintech use cases through managed-settings.json + MDM, but if the org requires FedRAMP High or regulated air-gap, consider Windsurf Enterprise Self-hosted as an alternative or complement.
