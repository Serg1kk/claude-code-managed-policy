# Managed Policy CLAUDE.md: Research

> **Language:** English · [Русский](README.ru.md)

**Research date:** 2026-04-21
**Sources:** Anthropic official docs (docs.claude.com, code.claude.com, docs.anthropic.com), Claude Code GitHub repo (issues + examples/mdm templates), managed-settings.net (community reference builder), 10+ community deployment guides (Addigy, systemprompt.io, ClaudeLog, ClaudeFa.st, Truefoundry, claude-ai.chat, etc.)

---

## Section TL;DRs

### Section 1: Paths (VERIFIED, with corrections)

| OS | managed-settings.json (**policy enforcement**) | Managed CLAUDE.md (**behavioral memory**) | MDM endpoint |
|---|---|---|---|
| **macOS** | `/Library/Application Support/ClaudeCode/managed-settings.json` | `/Library/Application Support/ClaudeCode/CLAUDE.md` | `com.anthropic.claudecode` managed preferences domain |
| **Linux / WSL** | `/etc/claude-code/managed-settings.json` | `/etc/claude-code/CLAUDE.md` | N/A (file-based or config management: Ansible/Puppet/Chef) |
| **Windows** | `C:\Program Files\ClaudeCode\managed-settings.json` ⚠️ **not ProgramData** | `C:\Program Files\ClaudeCode\CLAUDE.md` ⚠️ | `HKLM\SOFTWARE\Policies\ClaudeCode` reg key |

**Notes on common documentation discrepancies seen in community guides:**
1. **Windows path**: some community mirrors (`claude.yourdocs.dev`, various mintlify forks) still list `C:\ProgramData\ClaudeCode\CLAUDE.md` as canonical. With Claude Code v2.1.75 (late 2025), `C:\ProgramData\ClaudeCode\` was **deprecated**; canonical is now `C:\Program Files\ClaudeCode\`.
2. Many community write-ups reduce managed policy to just **Managed CLAUDE.md** (behavioral file) and skip **managed-settings.json**, the primary enterprise control surface covering permissions, MCP, models, plugins, hooks, and sandbox. Managed CLAUDE.md is *instructions* (soft, the model can ignore it); managed-settings.json is *hard enforcement* on the client.
3. The canonical Linux path is `/etc/claude-code/` (with hyphen). The `/etc/claude/` variant seen in some sources is outdated/incorrect.
4. An env var named `CLAUDE_MANAGED_SETTINGS_PATH` appears in some community-built references (mintlify fork), **but is NOT confirmed in official Anthropic docs v2.1.x**. Treat with caution; it is likely a legacy feature or community folklore.

### Section 2: MDM distribution (three channels)

Enterprise has **three delivery mechanisms** (per docs.claude.com/settings):

1. **Server-managed settings** (NEW, launched ~Q4 2025): Claude.ai admin console, pushed to the client automatically on login. **Do you need MDM? No.** You need a Claude for Teams (v2.1.38+) or Enterprise (v2.1.30+) plan.
2. **MDM / OS-level policies:**
   - macOS: plist in the `com.anthropic.claudecode` domain → Jamf / Kandji / Intune configuration profile
   - Windows: REG_SZ JSON in `HKLM\SOFTWARE\Policies\ClaudeCode\Settings` → Group Policy / Intune
3. **File-based:** `managed-settings.json` + `managed-settings.d/*.json` drop-in (systemd-style merge order) → Ansible / Puppet / Chef / shell script

**Precedence inside the managed tier:** server-managed > MDM/OS-level > file-based > HKCU registry (Windows per-user fallback). Only **one** managed source is active at a time: they are **not merged** together.

**Official templates:** https://github.com/anthropics/claude-code/tree/main/examples/mdm. Starters for Jamf, Kandji (Iru), Intune, and Group Policy.

### Section 3: What is enforceable (managed-only keys)

Full list of managed-only keys (they do not work outside managed-settings.json):
- `allowManagedMcpServersOnly`: only admin-approved MCP
- `allowManagedPermissionRulesOnly`: only admin permission rules apply
- `allowManagedHooksOnly`: admin hooks + SDK only
- `blockedMarketplaces` / `strictKnownMarketplaces`: block/allowlist plugin marketplaces
- `channelsEnabled` / `allowedChannelPlugins`: channel control (Team/Enterprise)
- `forceLoginMethod` / `forceLoginOrgUUID`: corp accounts only, block personal
- `forceRemoteSettingsRefresh`: fail-closed when server-managed is unavailable
- `pluginTrustMessage`: custom warning on plugin install
- `sandbox.filesystem.allowManagedReadPathsOnly`: admin paths only in sandbox
- `network.allowManagedDomainsOnly`: admin domains only for WebFetch
- `skipWebFetchPreflight`: skip preflight check (strict network env)

**Settings that work everywhere but get highest precedence when managed:**
- `permissions.allow/ask/deny` (including `Bash()`, `Read()`, `Write()`, `WebFetch()`, `Agent()`, `MCP()`)
- `availableModels`: model allowlist (blocks the ANTHROPIC_MODEL env var, `/model` commands, `--model` flag, Config tool)
- `disableBypassPermissionsMode: "disable"`: blocks `--dangerously-skip-permissions`
- `disableAutoMode: "disable"`: blocks auto mode
- `disableSkillShellExecution: true`: blocks `!` shell in skills
- `sandbox.enabled: true` + `sandbox.failIfUnavailable: true`: hard-gate sandboxing
- `hooks`: PreToolUse / PostToolUse / SessionStart audit hooks
- `env`: environment variables per session
- `companyAnnouncements`: welcome messages
- `minimumVersion`: pin CLI version
- `includeCoAuthoredBy` / `attribution`: control commit messages
- `allowedMcpServers` / `deniedMcpServers`: MCP allowlist/blocklist

### Section 4: Override / bypass

**What cannot be bypassed:**
- ✅ `managed-settings.json` permissions.deny: a **hard gate on the client**. The `--dangerously-skip-permissions` CLI flag can be disabled via `disableBypassPermissionsMode`.
- ✅ Managed CLAUDE.md: **cannot be excluded** via `claudeMdExcludes` (the restriction only applies to user/project/local CLAUDE.md, not managed). Confirmed in the official docs at docs.anthropic.com/memory.
- ✅ `--setting-source local` does **NOT** disable managed policies (confirmed by GitHub issue #11872; reply from @ashwin-ant at Anthropic: "Enterprise policies are not intended to be overridable").
- ✅ `forceRemoteSettingsRefresh: true`: blocks CLI startup when server-managed settings are unavailable (fail-closed).

**Known architectural gap (as of 2026-04-21, open):**
- ⚠️ **`claudeMdRequires` is not yet implemented** (GitHub issue #34349, filed 2026-03-14). Until it ships, a user can use `claudeMdExcludes` in their own user settings to exclude **project-level** security-*.md rule files, losing the modular rules. The only workaround today is to **put everything mandatory into the managed CLAUDE.md monolith**, which cannot be excluded. Enterprise admins complain that this forces a monolith instead of a modular `.claude/rules/` structure.

**What the user can still do (and this is OK by design):**
- Add a user-level CLAUDE.md with additional instructions (they merge with managed)
- Write their own project CLAUDE.md with team-shared instructions
- Use `--setting-source user` / `--setting-source local` to **narrow** the loaded settings (but managed is always present)
- Locally ignore managed CLAUDE.md **at the OS level**: if the file has no read permission, it won't load (graceful degradation; the CLI continues to work with user/project settings)

### Section 5: Real-world use cases (with concrete snippets)

1. **Block BYOK** (personal API keys):
   ```json
   {
     "forceLoginMethod": "claudeai",
     "forceLoginOrgUUID": ["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]
   }
   ```
   Login via corporate account only. A personal Claude subscription fails to log in.

2. **Coding standards enforcement (HIPAA/SOC2 examples):**
   ```json
   {
     "permissions": {
       "deny": ["Bash(curl *)", "Read(./.env)", "Read(./.env.*)",
                 "Read(./secrets/**)", "Read(~/.aws/credentials)",
                 "Read(~/.ssh/**)", "Bash(git push:*)"],
       "ask": ["Write(**)"]
     },
     "allowManagedPermissionRulesOnly": true
   }
   ```

3. **Block MCP in production:**
   ```json
   {
     "allowedMcpServers": [{"serverName": "github"}, {"serverName": "company-vault"}],
     "deniedMcpServers": [{"serverName": "filesystem"}],
     "allowManagedMcpServersOnly": true,
     "strictKnownMarketplaces": [{"source": "github", "repo": "acme-corp/plugins"}]
   }
   ```

4. **Audit via hooks (ship to Splunk/Datadog/ELK):**
   ```json
   {
     "hooks": {
       "PostToolUse": [{
         "matcher": "Edit|Write|Bash",
         "hooks": [{"type": "command", "command": "/usr/local/bin/audit-to-splunk.sh"}]
       }]
     },
     "allowManagedHooksOnly": true
   }
   ```

5. **Pin CLI version + block downgrade:**
   ```json
   {
     "minimumVersion": "2.1.100",
     "availableModels": ["sonnet", "haiku"]
   }
   ```

6. **Sandbox hard-gate:**
   ```json
   {
     "sandbox": {
       "enabled": true,
       "failIfUnavailable": true,
       "allowUnsandboxedCommands": false,
       "filesystem": {
         "denyRead": ["~/.aws/**", "~/.ssh/**", "/Users/**/keys/**"],
         "allowManagedReadPathsOnly": true
       },
       "network": {
         "allowedDomains": ["*.company.com", "github.com", "*.anthropic.com"],
         "allowManagedDomainsOnly": true
       }
     }
   }
   ```

### Section 6: Competitor comparison (summary table)

| Capability | **Claude Code** | **Cursor** | **Copilot Business/Enterprise** | **OpenAI Codex Enterprise** | **Windsurf (Codeium)** |
|---|---|---|---|---|---|
| **Managed policy file?** | ✅ `managed-settings.json` + Managed CLAUDE.md (local) | ❌ No local enforcement file; via Team Rules in the Admin dashboard + MDM at the team-IDs level | ❌ Policy only via GitHub org settings UI (not a file) | ✅ `requirements.toml` + `managed_config.toml` (cloud- or device-delivered) | ⚠️ Has policy via Windows Registry + plist + Linux JSON files, but **ADMX templates** manage the VSCode-fork, not behavioral rules |
| **MDM distribution?** | ✅ Jamf / Kandji / Intune / Group Policy (official templates) | ⚠️ MDM controls team IDs + extensions, not rules | ❌ None (web-only policy UI) | ⚠️ macOS MDM + Windows policy for `requirements.toml` | ✅ Group Policy ADMX/ADML + MDM |
| **Hard permission deny?** | ✅ Hard deny on client (Bash/Read/Write/WebFetch/MCP) | ⚠️ Privacy Mode enforce + repository blocklist, but not fine-grained tool deny | ⚠️ Content exclusions + audit; deny at the feature level, not per command | ✅ `requirements.toml` blocks commands + MCP allowlist | ⚠️ Command allow/denylist at the team level (not OS-level enforcement) |
| **Bypass impossible?** | ✅ `--dangerously-skip-permissions` blocked via `disableBypassPermissionsMode`; `--setting-source local` does not disable managed | ⚠️ Privacy Mode cannot be turned off, but project rules are not a hard gate | ❓ Not documented in detail | ✅ `requirements.toml` blocking cannot be overridden via CLI | ⚠️ Admin-set max auto-exec level blocks local upgrades |
| **Block BYOK / personal API keys?** | ✅ `forceLoginMethod` + `forceLoginOrgUUID` (hard) | ✅ Business+: enforce team org, SCIM | ✅ GitHub org policy | ✅ RBAC + ChatGPT Business/Enterprise login | ✅ SCIM + SSO + zero-data retention |
| **Model allowlist?** | ✅ `availableModels: ["sonnet", "haiku"]` | ⚠️ UI-level, not hard-enforced | ✅ Model allowlist policy | ✅ Model access per role | ⚠️ UI toggle only |
| **Plugin/marketplace allowlist?** | ✅ `strictKnownMarketplaces` (managed-only) | ❌ No marketplace yet | ⚠️ Via MCP registry (GA) | ✅ MCP allowlist in `requirements.toml` | ⚠️ Extension allowlist via Group Policy ADMX |
| **Audit logging?** | ✅ Compliance API (Enterprise) + hooks for self-hosted | ✅ Audit dashboards + Admin API + AI Code Tracking API | ✅ GitHub Audit Log + Enterprise analytics | ✅ ChatGPT Enterprise audit logs | ✅ Admin Portal analytics + OTel |
| **On-prem / self-hosted?** | ⚠️ Bedrock/Vertex/Foundry proxy, but the CLI itself still talks to the model | ❌ Cloud only | ⚠️ GitHub AE enterprise | ❌ Cloud only (via ChatGPT Enterprise) | ✅ Enterprise Hybrid + Self-hosted tiers (FedRAMP High) |
| **SOC 2 Type II / ISO 27001?** | ✅ SOC2 T2 + ISO 27001 + ISO 42001 + HIPAA BAA | ✅ SOC 2 T2 | ✅ SOC 2 T2 + IP indemnity | ✅ SOC 2 T2 | ✅ SOC 2 T2 + FedRAMP High |

**In short: where Claude Code is stronger.**
- The most detailed fine-grained hard enforcement via file-based policy (permissions, MCP, plugins, hooks, sandbox)
- Managed CLAUDE.md + managed-settings.json = a **two-layer** model (behavioral + enforcement)
- Drop-in `managed-settings.d/` for modular policy composition (unique)
- Full MDM parity with Jamf / Intune / Group Policy (official templates)

**Where it is weaker:**
- No self-hosted tier (like Windsurf Enterprise Self-hosted)
- No IP indemnity (like Copilot Business/Enterprise)
- `claudeMdRequires` not yet implemented (monolith vs modular)
- Server-managed settings are uniform per org only (no per-group policies, unlike Codex Enterprise)

### Section 7: Enterprise readiness

A baseline `managed-settings.json` template for a 60-developer fintech team lives in `section-7-template/managed-settings.json`.
The IT team checklist lives in `section-7-template/it-checklist.md`.

---

## Key findings for the final

1. **The main mistake in many current community guides** is reducing managed policy to just CLAUDE.md. In reality, these are **two different files with different jobs:** `managed-settings.json` handles hard policy enforcement (permissions, MCP, plugins, models, bypass); `CLAUDE.md` holds behavioral memory/instructions. Enterprise uses **both**. For 95% of compliance tasks you need `managed-settings.json`, not CLAUDE.md.

2. **Server-managed settings** (Claude.ai admin console pushes to the client) are a new mechanism missing from a number of public references. They do not require MDM. Available only on Claude for Teams / Enterprise plans. This is the **only** option for admins with unmanaged devices.

3. **Managed CLAUDE.md really cannot be excluded** via `claudeMdExcludes`. Confirmed in docs.anthropic.com/memory: *"Managed policy CLAUDE.md files cannot be excluded. This ensures [that] organizational policy is always applied"*.

4. **The Windows canonical path changed in v2.1.75**: `C:\ProgramData\ClaudeCode\` is **deprecated** in favor of `C:\Program Files\ClaudeCode\`. Older documentation mirrors still show ProgramData. Current community guides already use the correct `C:\Program Files\` path. ✅

5. **`--setting-source local` does NOT disable managed** (bug report #11872, confirmed by-design by Anthropic). For enterprise users this means: **once IT rolls it out, you cannot bypass it in the client**, only by deleting the file at the OS level (which requires admin rights and generates an audit event).

6. **Drop-in directory `managed-settings.d/`** is an important but under-covered feature. It lets different teams (security, DevOps, compliance) push **independent policy fragments** into a single JSON without coordinating. Systemd-style merge order (10-, 20-, 30- prefixes). Added in v2.1.83.

7. **MDM platform coverage:** Anthropic officially publishes starter templates for **four** platforms: Jamf Pro, Kandji (internal code name "Iru"), Microsoft Intune, and Group Policy (ADMX/ADML). For Linux Ansible/Puppet/Chef there are **no** official templates, but community guides (systemprompt.io, Addigy) cover that gap.

8. **Architectural limitation:** until `claudeMdRequires` ships (issue #34349 open since March 2026), enterprises are forced to put everything mandatory into a single monolithic managed CLAUDE.md. Modular `.claude/rules/*.md` files can be excluded at the user level via `claudeMdExcludes`, which means **that path is unreliable for truly mandatory rules**.

---

## Gaps / what did not pan out

- **`CLAUDE_MANAGED_SETTINGS_PATH` env var**: mentioned in mintlify forks of community docs, but **not in official Anthropic docs v2.1.x**. Needs validation via direct testing before it shows up in a presentation.
- **Audit log events for bypass attempts** against managed policy: the docs mention the Compliance API (Enterprise only), but **the specifics of what is logged when a denied command is attempted** are not public. For the M2 answer it is enough to say: "deny is blocked on the client; audit is written via the Compliance API (Enterprise plan), or via hooks if you set them up yourself."

---

## Useful links (canonical)

**Official Anthropic docs:**
- https://docs.claude.com/en/docs/claude-code/settings: full settings reference
- https://code.claude.com/docs/en/server-managed-settings.md: server-managed (new mechanism)
- https://code.claude.com/docs/en/permissions.md: managed-only permissions keys
- https://docs.anthropic.com/en/docs/claude-code/memory: CLAUDE.md hierarchy + managed policy
- https://docs.claude.com/en/docs/claude-code/third-party-integrations: enterprise deployment overview
- https://claude.com/product/claude-code/enterprise: Claude Code for Enterprise product page
- https://github.com/anthropics/claude-code/tree/main/examples/mdm: starter templates (Jamf/Kandji/Intune/GPO)

**Community references:**
- https://managed-settings.net/: community-built settings builder (Omri Yaakov). An excellent UI reference for all managed-only keys.
- https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-settings.md: complete config guide v2.1.114
- https://systemprompt.io/guides/enterprise-claude-code-managed-settings: enterprise rollout playbook
- https://systemprompt.io/guides/claude-code-organisation-rollout: 50+ developer rollout playbook
- https://addigy.com/blog/manage-claude-code-policies-addigy/: Addigy MDM deployment walkthrough
- https://claudefa.st/blog/guide/settings-reference: settings reference (2026-04-18)
- https://claude-ai.chat/blog/claude-code-in-enterprise-environments/: SOC2 / HIPAA practical guide
- https://amitkoth.com/claude-code-soc2-compliance-auditor-guide/: audit perspective

**GitHub issues (context):**
- https://github.com/anthropics/claude-code/issues/34349: claudeMdRequires feature request (open)
- https://github.com/anthropics/claude-code/issues/20880: parent CLAUDE.md exclusion (open)
- https://github.com/anthropics/claude-code/issues/11872: `--setting-source local` with managed policy (won't fix, by design)

**Competitor comparison:**
- https://cursor.com/docs/account/teams/enterprise-settings: Cursor Enterprise overview
- https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies: Copilot Enterprise policies
- https://developers.openai.com/codex/enterprise/managed-configuration/: Codex requirements.toml
- https://docs.windsurf.com/windsurf/enterprise-policies: Windsurf Enterprise Policies
- https://codeium.com/security: Windsurf deployment tiers

---
