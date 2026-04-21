# Managed Policy CLAUDE.md: Research

**Дата ресёрча:** 2026-04-21
**Источники:** Anthropic official docs (docs.claude.com, code.claude.com, docs.anthropic.com), Claude Code GitHub repo (issues + examples/mdm templates), managed-settings.net (community reference builder), 10+ community deployment guides (Addigy, systemprompt.io, ClaudeLog, ClaudeFa.st, Truefoundry, claude-ai.chat, etc.)

---

## TL;DR по секциям

### Section 1 — Paths (VERIFIED, с исправлениями)

| OS | managed-settings.json (**policy enforcement**) | Managed CLAUDE.md (**behavioral memory**) | MDM endpoint |
|---|---|---|---|
| **macOS** | `/Library/Application Support/ClaudeCode/managed-settings.json` | `/Library/Application Support/ClaudeCode/CLAUDE.md` | `com.anthropic.claudecode` managed preferences domain |
| **Linux / WSL** | `/etc/claude-code/managed-settings.json` | `/etc/claude-code/CLAUDE.md` | N/A (file-based или конфиг-менеджмент: Ansible/Puppet/Chef) |
| **Windows** | `C:\Program Files\ClaudeCode\managed-settings.json` ⚠️ **не ProgramData** | `C:\Program Files\ClaudeCode\CLAUDE.md` ⚠️ | `HKLM\SOFTWARE\Policies\ClaudeCode` reg key |

**Notes on common documentation discrepancies seen in community guides:**
1. **Windows path**: some community mirrors (`claude.yourdocs.dev`, various mintlify forks) still list `C:\ProgramData\ClaudeCode\CLAUDE.md` as canonical. With Claude Code v2.1.75 (late 2025) `C:\ProgramData\ClaudeCode\` was **deprecated**; canonical is now `C:\Program Files\ClaudeCode\`.
2. Many community write-ups reduce managed policy to just **Managed CLAUDE.md** (behavioral file) and skip **managed-settings.json** — the primary enterprise control surface covering permissions, MCP, models, plugins, hooks, sandbox. Managed CLAUDE.md is *instructions* (soft, model can ignore); managed-settings.json is *hard enforcement* on the client.
3. Linux canonical path is `/etc/claude-code/` (with hyphen). The variant `/etc/claude/` seen in some sources is outdated/incorrect.
4. An env var named `CLAUDE_MANAGED_SETTINGS_PATH` appears in some community-built references (mintlify fork), **but is NOT confirmed in official Anthropic docs v2.1.x** — treat with caution, likely a legacy feature or community folklore.

### Section 2 — MDM distribution (три канала)

Enterprise имеет **три механизма доставки** (точно по docs.claude.com/settings):

1. **Server-managed settings** (NEW, launched ~Q4 2025) — Claude.ai admin console → автоматом в клиент при логине. **Нужно MDM? Нет.** Нужно Claude for Teams (v2.1.38+) или Enterprise (v2.1.30+) plan.
2. **MDM / OS-level policies:**
   - macOS: plist в `com.anthropic.claudecode` домене → Jamf / Kandji / Intune configuration profile
   - Windows: REG_SZ JSON в `HKLM\SOFTWARE\Policies\ClaudeCode\Settings` → Group Policy / Intune
3. **File-based:** `managed-settings.json` + `managed-settings.d/*.json` drop-in (systemd-style merge order) → Ansible / Puppet / Chef / shell script

**Precedence внутри managed tier:** server-managed > MDM/OS-level > file-based > HKCU registry (Windows per-user fallback). Только **один** managed-source активен — они **не мёрджатся** между собой.

**Official templates:** https://github.com/anthropics/claude-code/tree/main/examples/mdm — стартеры для Jamf, Kandji (Iru), Intune, Group Policy.

### Section 3 — Что enforceable (managed-only keys)

Полный список managed-only ключей (не работают вне managed-settings.json):
- `allowManagedMcpServersOnly` — only admin-approved MCP
- `allowManagedPermissionRulesOnly` — only admin permission rules применяются
- `allowManagedHooksOnly` — только admin-hooks + SDK
- `blockedMarketplaces` / `strictKnownMarketplaces` — блок/allowlist plugin marketplaces
- `channelsEnabled` / `allowedChannelPlugins` — контроль channels (Team/Enterprise)
- `forceLoginMethod` / `forceLoginOrgUUID` — только корп. аккаунты, запрет personal
- `forceRemoteSettingsRefresh` — fail-closed при недоступности server-managed
- `pluginTrustMessage` — custom warning при установке плагина
- `sandbox.filesystem.allowManagedReadPathsOnly` — только admin-paths в sandbox
- `network.allowManagedDomainsOnly` — только admin-domains для WebFetch
- `skipWebFetchPreflight` — skip preflight check (strict network env)

**Settings которые работают везде, но в managed получают highest precedence:**
- `permissions.allow/ask/deny` (включая `Bash()`, `Read()`, `Write()`, `WebFetch()`, `Agent()`, `MCP()`)
- `availableModels` — allowlist моделей (блок для ANTHROPIC_MODEL env var, /model команды, --model flag, Config tool)
- `disableBypassPermissionsMode: "disable"` — запрет `--dangerously-skip-permissions`
- `disableAutoMode: "disable"` — запрет auto mode
- `disableSkillShellExecution: true` — блок `!` shell в skills
- `sandbox.enabled: true` + `sandbox.failIfUnavailable: true` — hard-gate sandboxing
- `hooks` — PreToolUse / PostToolUse / SessionStart audit hooks
- `env` — environment variables на каждую сессию
- `companyAnnouncements` — приветственные сообщения
- `minimumVersion` — пин версии CLI
- `includeCoAuthoredBy` / `attribution` — контроль commit messages
- `allowedMcpServers` / `deniedMcpServers` — white/blacklist MCP

### Section 4 — Override / bypass

**Что невозможно обойти:**
- ✅ `managed-settings.json` permissions.deny — **hard gate на уровне клиента**, CLI-flag `--dangerously-skip-permissions` может быть задизейблен через `disableBypassPermissionsMode`
- ✅ Managed CLAUDE.md — **нельзя исключить** через `claudeMdExcludes` (ограничение применяется только к user/project/local CLAUDE.md, не к managed). Подтверждено в официальных docs docs.anthropic.com/memory.
- ✅ `--setting-source local` **НЕ** отключает managed policies (подтверждено GitHub issue #11872, ответ @ashwin-ant from Anthropic: "Enterprise policies are not intended to be overridable").
- ✅ `forceRemoteSettingsRefresh: true` — блокирует CLI startup если server-managed settings не доступны (fail-closed).

**Известная архитектурная дыра (на 2026-04-21, open):**
- ⚠️ **`claudeMdRequires` ещё не реализован** (GitHub issue #34349, filed 2026-03-14). Пока его нет, user может через `claudeMdExcludes` у себя в user-settings исключить **project-level** security-*.md rule files → теряются модульные rules. Единственный workaround сейчас — **положить всё mandatory в managed CLAUDE.md монолит**, его исключить нельзя. Enterprise-админы жалуются: форсит монолит вместо модульной `.claude/rules/` структуры.

**Что всё ещё может сделать пользователь (и это ОК by design):**
- Добавить user-level CLAUDE.md с дополнительными инструкциями (они мёрджатся с managed)
- Писать свой project CLAUDE.md с team-shared инструкциями
- Использовать `--setting-source user` / `--setting-source local` чтобы **сузить** загружаемые настройки (но managed остаётся всегда)
- Локально игнорировать managed CLAUDE.md **на уровне ОС** — если у него нет read permission, файл не загрузится (graceful degradation, CLI продолжит работать с user/project settings)

### Section 5 — Real-world use-cases (с конкретными сниппетами)

1. **Запрет BYOK** (персональные API keys):
   ```json
   {
     "forceLoginMethod": "claudeai",
     "forceLoginOrgUUID": ["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]
   }
   ```
   → логин только через корп.аккаунт. Personal Claude subscription → login fails.

2. **Coding standards enforcement (HIPAA/SOC2 примеры):**
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

3. **Запрет MCP на проде:**
   ```json
   {
     "allowedMcpServers": [{"serverName": "github"}, {"serverName": "company-vault"}],
     "deniedMcpServers": [{"serverName": "filesystem"}],
     "allowManagedMcpServersOnly": true,
     "strictKnownMarketplaces": [{"source": "github", "repo": "acme-corp/plugins"}]
   }
   ```

4. **Аудит через hooks (отправка в Splunk/Datadog/ELK):**
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

5. **Pin CLI version + запрет downgrade:**
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

### Section 6 — Сравнение с конкурентами (summary table)

| Capability | **Claude Code** | **Cursor** | **Copilot Business/Enterprise** | **OpenAI Codex Enterprise** | **Windsurf (Codeium)** |
|---|---|---|---|---|---|
| **Managed policy file?** | ✅ `managed-settings.json` + Managed CLAUDE.md (local) | ❌ Нет локального enforcement-файла; через Team Rules в Admin dashboard + MDM на уровне team IDs | ❌ Policy только через GitHub org settings UI (не файл) | ✅ `requirements.toml` + `managed_config.toml` (cloud- или device-delivered) | ⚠️ Есть policy через Windows Registry + plist + Linux JSON files, но **ADMX templates** — managed VSCode-fork, не behavioral rules |
| **MDM distribution?** | ✅ Jamf / Kandji / Intune / Group Policy (official templates) | ⚠️ MDM controls team IDs + extensions, не rules | ❌ Нет (web-only policy UI) | ⚠️ macOS MDM + Windows policy для `requirements.toml` | ✅ Group Policy ADMX/ADML + MDM |
| **Hard permission deny?** | ✅ Hard deny on client (Bash/Read/Write/WebFetch/MCP) | ⚠️ Privacy Mode enforce + repository blocklist, но не fine-grained tool deny | ⚠️ Content exclusions + audit; deny на уровне features, не команд | ✅ `requirements.toml` blocks commands + MCP allowlist | ⚠️ Command allow/denylist на уровне team (не os-level enforce) |
| **Bypass невозможен?** | ✅ `--dangerously-skip-permissions` блокируется через `disableBypassPermissionsMode`; `--setting-source local` не отключает managed | ⚠️ Privacy Mode нельзя выключить, но project rules — нет hard gate | ❓ Не документировано детально | ✅ `requirements.toml` blocking нельзя override через CLI | ⚠️ Admin-set max auto-exec level блокирует локальные upgrades |
| **Block BYOK / personal API keys?** | ✅ `forceLoginMethod` + `forceLoginOrgUUID` (hard) | ✅ Business+: enforce team org, SCIM | ✅ GitHub org policy | ✅ RBAC + ChatGPT Business/Enterprise login | ✅ SCIM + SSO + zero-data retention |
| **Model allowlist?** | ✅ `availableModels: ["sonnet", "haiku"]` | ⚠️ UI-level, не hard-enforced | ✅ Model allowlist policy | ✅ Model access per role | ⚠️ UI toggle только |
| **Plugin/marketplace allowlist?** | ✅ `strictKnownMarketplaces` (managed-only) | ❌ Нет marketplace пока | ⚠️ Через MCP registry (GA) | ✅ MCP allowlist в `requirements.toml` | ⚠️ Extension allowlist через Group Policy ADMX |
| **Audit logging?** | ✅ Compliance API (Enterprise) + hooks для self-hosted | ✅ Audit dashboards + Admin API + AI Code Tracking API | ✅ GitHub Audit Log + Enterprise analytics | ✅ ChatGPT Enterprise audit logs | ✅ Admin Portal analytics + OTel |
| **On-prem / self-hosted?** | ⚠️ Bedrock/Vertex/Foundry proxy, но сам CLI всё равно talks к модели | ❌ Cloud only | ⚠️ GitHub AE enterprise | ❌ Cloud only (через ChatGPT Enterprise) | ✅ Enterprise Hybrid + Self-hosted tiers (FedRAMP High) |
| **SOC 2 Type II / ISO 27001?** | ✅ SOC2 T2 + ISO 27001 + ISO 42001 + HIPAA BAA | ✅ SOC 2 T2 | ✅ SOC 2 T2 + IP indemnity | ✅ SOC 2 T2 | ✅ SOC 2 T2 + FedRAMP High |

**Кратко — где Claude Code сильнее:**
- Самая детальная fine-grained hard enforcement через файловый policy (permissions, MCP, plugins, hooks, sandbox)
- Managed CLAUDE.md + managed-settings.json = **two-layer** (behavioral + enforcement)
- Drop-in `managed-settings.d/` для modular policy composition (уникально)
- Full MDM parity с Jamf / Intune / Group Policy (official templates)

**Где слабее:**
- Нет self-hosted tier (как у Windsurf Enterprise Self-hosted)
- Нет IP indemnity (как у Copilot Business/Enterprise)
- `claudeMdRequires` пока не реализован (монолит vs modular)
- Server-managed settings only uniform per org (нет per-group policies, как в Codex Enterprise)

### Section 7 — Enterprise readiness

Template baseline `managed-settings.json` для fintech-команды из 60 разработчиков — в `section-7-template/managed-settings.json`.
Чеклист IT-отдела — в `section-7-template/it-checklist.md`.

---

## Key findings для финала

1. **Главная ошибка текущих many community guides** — сводить managed policy только к CLAUDE.md. На самом деле это **два разных файла с разными задачами:** `managed-settings.json` — hard policy enforcement (permissions, MCP, plugins, models, bypass), `CLAUDE.md` — behavioral memory/instructions. Enterprise использует **оба**. Для 95% compliance-задач нужен `managed-settings.json`, не CLAUDE.md.

2. **Server-managed settings** (Claude.ai admin console → push в клиент) — это новый механизм, который пропущен в ряде public references. Не требует MDM. Доступен только Claude for Teams / Enterprise plans. Это **единственный** способ для админов с unmanaged devices.

3. **Managed CLAUDE.md действительно нельзя исключить** через `claudeMdExcludes` — подтверждено в docs.anthropic.com/memory: *"Managed policy CLAUDE.md files cannot be excluded. This ensures [that] organizational policy is always applied"*.

4. **Windows canonical path изменился с v2.1.75** — `C:\ProgramData\ClaudeCode\` **deprecated** в пользу `C:\Program Files\ClaudeCode\`. Старые зеркала документации ещё показывают ProgramData. Текущие many community guides уже используют корректный путь `C:\Program Files\`. ✅

5. **`--setting-source local` НЕ отключает managed** (bug-report #11872, confirmed by-design by Anthropic). Для enterprise users — это значит: **однажды IT раскатал — обойти в клиенте невозможно**, только через удаление файла на уровне OS (требует admin, audit-event).

6. **Drop-in directory `managed-settings.d/`** — важная, но пропущенная в материалах фича. Позволяет разным командам (security, DevOps, compliance) пушить **независимые policy fragments** без координации в одном JSON. Systemd-style merge order (10-, 20-, 30- prefixes). Добавлено в v2.1.83.

7. **MDM platform coverage:** Anthropic официально публикует starter templates для **четырёх** платформ — Jamf Pro, Kandji (внутренний кодовый "Iru"), Microsoft Intune, Group Policy (ADMX/ADML). Для Linux Ansible/Puppet/Chef официальных шаблонов **нет**, но community guides (systemprompt.io, Addigy) покрывают.

8. **Architectural limitation:** пока нет `claudeMdRequires` (issue #34349 open since March 2026), enterprise forced put всё mandatory в один monolithic managed CLAUDE.md. Модульные `.claude/rules/*.md` можно исключить у user-уровня через `claudeMdExcludes` — значит **для реально-мандатных правил этот путь ненадёжен**.

---

## Gaps / что не подтвердилось

- **`CLAUDE_MANAGED_SETTINGS_PATH` env var** — упоминается в mintlify-форках community docs, но **не в официальных Anthropic docs v2.1.x**. Нужно валидировать через прямое тестирование, прежде чем писать в презентацию.
- **Audit log events для попыток обойти** managed policy — документация упоминает Compliance API (Enterprise only) но **детали того, что именно логируется при попытке запустить denied command**, не публичны. Для M2 ответа достаточно сказать: "deny блокируется на клиенте, audit пишется через Compliance API (Enterprise plan), или через hooks если вы их сами настроили".

---

## Полезные ссылки (canonical)

**Official Anthropic docs:**
- https://docs.claude.com/en/docs/claude-code/settings — полная reference settings
- https://code.claude.com/docs/en/server-managed-settings.md — server-managed (new mechanism)
- https://code.claude.com/docs/en/permissions.md — managed-only permissions keys
- https://docs.anthropic.com/en/docs/claude-code/memory — CLAUDE.md hierarchy + managed policy
- https://docs.claude.com/en/docs/claude-code/third-party-integrations — enterprise deployment overview
- https://claude.com/product/claude-code/enterprise — Claude Code for Enterprise product page
- https://github.com/anthropics/claude-code/tree/main/examples/mdm — starter templates (Jamf/Kandji/Intune/GPO)

**Community references:**
- https://managed-settings.net/ — community-built settings builder (Omri Yaakov) — отличный UI-reference для всех managed-only keys
- https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-settings.md — complete config guide v2.1.114
- https://systemprompt.io/guides/enterprise-claude-code-managed-settings — enterprise rollout playbook
- https://systemprompt.io/guides/claude-code-organisation-rollout — 50+ developer rollout playbook
- https://addigy.com/blog/manage-claude-code-policies-addigy/ — Addigy MDM deployment walkthrough
- https://claudefa.st/blog/guide/settings-reference — settings reference (2026-04-18)
- https://claude-ai.chat/blog/claude-code-in-enterprise-environments/ — SOC2 / HIPAA practical guide
- https://amitkoth.com/claude-code-soc2-compliance-auditor-guide/ — audit perspective

**GitHub issues (context):**
- https://github.com/anthropics/claude-code/issues/34349 — claudeMdRequires feature request (open)
- https://github.com/anthropics/claude-code/issues/20880 — parent CLAUDE.md exclusion (open)
- https://github.com/anthropics/claude-code/issues/11872 — `--setting-source local` с managed policy (won't fix — by design)

**Сравнение с конкурентами:**
- https://cursor.com/docs/account/teams/enterprise-settings — Cursor Enterprise overview
- https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies — Copilot Enterprise policies
- https://developers.openai.com/codex/enterprise/managed-configuration/ — Codex requirements.toml
- https://docs.windsurf.com/windsurf/enterprise-policies — Windsurf Enterprise Policies
- https://codeium.com/security — Windsurf deployment tiers

---
