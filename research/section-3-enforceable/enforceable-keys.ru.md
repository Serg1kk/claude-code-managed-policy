# Section 3 — Что enforceable через Managed Policy

> **Язык:** [English](enforceable-keys.md) · Русский

**Источники:**
- https://docs.claude.com/en/docs/claude-code/settings (полная reference)
- https://code.claude.com/docs/en/permissions.md (managed-only keys)
- https://managed-settings.net/ (community reference builder — лучший UI)

---

## Managed-only keys (не работают вне managed-settings.json)

Эти ключи **читаются только из managed-settings.json / server-managed / MDM**. Если положить их в user/project settings — silent ignore.

| Ключ | Описание | Пример value |
|---|---|---|
| `allowManagedMcpServersOnly` | Только MCP servers из managed `allowedMcpServers` работают. User может добавить свои в `.mcp.json`, но они не загрузятся | `true` |
| `allowManagedPermissionRulesOnly` | Только permission rules (allow/ask/deny) из managed settings применяются. User/project rules игнорируются | `true` |
| `allowManagedHooksOnly` | Только hooks из managed + SDK + force-enabled plugins. User/project/plugin hooks блокируются | `true` |
| `allowedChannelPlugins` | Allowlist channel plugins (Team/Enterprise "channels" feature) | `[{"marketplace": "claude-plugins-official", "plugin": "telegram"}]` |
| `blockedMarketplaces` | Blocklist plugin marketplace sources — **не скачиваются** | `[{"source": "github", "repo": "untrusted/plugins"}]` |
| `strictKnownMarketplaces` | Allowlist marketplaces. Empty array = lockdown (никаких marketplaces) | `[{"source": "github", "repo": "acme-corp/plugins"}]` |
| `channelsEnabled` | Разрешить channels (Teams/Enterprise). Unset/false = блок | `true` |
| `forceLoginMethod` | `"claudeai"` или `"console"` — ограничить тип аккаунта | `"claudeai"` |
| `forceLoginOrgUUID` | UUID корп.орги (single или array). User не может логиниться вне этой орги | `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"` |
| `forceRemoteSettingsRefresh` | CLI startup блокируется пока не получит свежие server-managed settings. Fail-closed | `true` |
| `pluginTrustMessage` | Custom message в warning при установке плагина | `"All plugins from our marketplace are approved by IT"` |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Только managed paths в `filesystem.allowRead`, user paths ignored. `denyRead` всё ещё мёрджится со всех уровней | `true` |
| `network.allowManagedDomainsOnly` | Только managed `allowedDomains` + WebFetch allow rules. User domains ignored. Denied мёрджится | `true` |
| `skipWebFetchPreflight` | Skip WebFetch blocklist preflight (для restrictive network envs) | `true` |
| `sshConfigs` (managed mode) | SSH connections для Desktop env — когда из managed, read-only для user | `[{"id": "dev-vm", "name": "Dev VM", "sshHost": "user@dev.example.com"}]` |

---

## Настройки которые работают везде, но в managed получают highest precedence

Эти ключи читаются из любого scope, но managed-settings.json не может быть overridden — т.е. IT может **force-set** любое из них.

### Permissions (core enforcement)

```json
{
  "permissions": {
    "allow": ["Bash(npm run *)", "Bash(git status)", "Read(./src/**)", "Edit(./src/**)"],
    "ask":   ["Bash(git push *)", "Write(./README.md)", "WebFetch(domain:stackoverflow.com)"],
    "deny":  ["Bash(curl *)", "Bash(wget *)", "Bash(rm -rf *)", "Read(./.env)", "Read(./.env.*)",
              "Read(./secrets/**)", "Read(~/.aws/credentials)", "Read(~/.ssh/**)",
              "WebFetch", "Agent(third-party-*)", "MCP(filesystem)"],
    "defaultMode": "acceptEdits",
    "disableBypassPermissionsMode": "disable",
    "additionalDirectories": []
  }
}
```

**Rule syntax:**
- `Tool` — match all invocations of tool
- `Tool(specifier)` — narrower match
- `Bash(npm run *)` — любая `npm run ...` команда
- `Read(./.env)` — конкретный file
- `Read(./secrets/**)` — рекурсивно папка
- `WebFetch(domain:example.com)` — requests к домену
- `Agent(name-pattern)` — subagent invocations
- `MCP(serverName)` — MCP tool calls

**Precedence:** deny > ask > allow (first match wins). Managed deny — **нельзя обойти** lower-scope allow.

### Bypass controls

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  },
  "disableAutoMode": "disable",
  "disableSkillShellExecution": true,
  "disableAllHooks": false
}
```

- `disableBypassPermissionsMode: "disable"` — блокирует `--dangerously-skip-permissions` CLI flag + `defaultMode: "bypassPermissions"`
- `disableAutoMode: "disable"` — убирает `auto` из `Shift+Tab` cycle, `--permission-mode auto` rejected at startup
- `disableSkillShellExecution: true` — блок `!` shell execution в skills и custom commands (user/project/plugin/additional-directory sources). **Bundled + managed skills не затронуты.**
- `disableAllHooks` — полный блок hooks + custom statusline

### Model restrictions

```json
{
  "availableModels": ["sonnet", "haiku"],
  "model": "sonnet"
}
```

- `availableModels` — allowlist моделей. User не может `/model`-switch на не-listed. Работает и для CLI `--model` flag, и для `ANTHROPIC_MODEL` env var, и для Config tool.
- `model` — initial default для сессии (не enforcement — user может переключать в пределах `availableModels`)
- Merge: arrays concat'ятся + dedup через все scopes. **Для strict allowlist** — set в managed + `availableModels` **только там**.
- **Default model picker option** не затронут `availableModels` — он всегда доступен.

### Sandbox controls

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["docker *"],
    "filesystem": {
      "allowWrite": ["/tmp/build", "~/.kube"],
      "denyWrite": ["/etc", "/usr/local/bin"],
      "denyRead": ["~/.aws/**", "~/.ssh/**", "/Users/**/keys/**"],
      "allowRead": ["."],
      "allowManagedReadPathsOnly": true
    },
    "network": {
      "allowedDomains": ["*.company.com", "github.com", "*.anthropic.com", "registry.npmjs.org"],
      "deniedDomains": ["sensitive.internal.company.com"],
      "allowManagedDomainsOnly": true,
      "allowLocalBinding": false,
      "allowAllUnixSockets": false
    }
  }
}
```

- `failIfUnavailable: true` + `enabled: true` — **hard gate**, CLI exits если sandbox не доступен (missing deps, unsupported platform). Без failIfUnavailable — просто warning + unsandboxed execution.
- `allowUnsandboxedCommands: false` — `dangerouslyDisableSandbox` escape hatch **полностью отключён**
- `allowManagedReadPathsOnly: true` — user paths в `filesystem.allowRead` игнорируются
- `allowManagedDomainsOnly: true` — user domains в `allowedDomains` + `WebFetch(domain:...)` игнорируются

### Hooks (для audit logging)

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Edit|Write|WebFetch",
      "hooks": [{"type": "command", "command": "/usr/local/bin/audit-pre.sh"}]
    }],
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{"type": "command", "command": "/usr/local/bin/audit-post.sh"}]
    }],
    "SessionStart": [{
      "hooks": [{"type": "command", "command": "/usr/local/bin/session-log.sh"}]
    }]
  },
  "allowManagedHooksOnly": true,
  "allowedHttpHookUrls": ["https://audit.company.com/*", "https://splunk.internal/*"],
  "httpHookAllowedEnvVars": ["AUDIT_TOKEN", "HOOK_SECRET"]
}
```

Hook events:
- `PreToolUse` — до permission check (можно approve/deny в обход permission system)
- `PostToolUse` — после успешного выполнения
- `SessionStart` — при старте CLI
- `SessionEnd`
- `UserPromptSubmit`

Hooks запускают shell commands на user machine — security review важен. Managed `hooks` показывают security-approval dialog user'у при первом запуске (он может increment runtime permission).

### MCP controls

```json
{
  "allowedMcpServers": [{"serverName": "github"}, {"serverName": "company-vault"}],
  "deniedMcpServers": [{"serverName": "filesystem"}, {"serverName": "untrusted-*"}],
  "disabledMcpjsonServers": ["filesystem"],
  "enabledMcpjsonServers": ["memory", "github"],
  "enableAllProjectMcpServers": false,
  "allowManagedMcpServersOnly": true
}
```

`managed-mcp.json` (отдельный файл) — для конфигурации MCP серверов самих (не просто allow/deny, а полный config с command/args/env). Пример:
```json
{
  "mcpServers": {
    "company-vault": {
      "command": "node",
      "args": ["/opt/mcp-servers/company-vault/index.js"],
      "env": {"VAULT_URL": "https://vault.company.com"}
    }
  }
}
```

### Plugin marketplace controls

```json
{
  "strictKnownMarketplaces": [{"source": "github", "repo": "acme-corp/plugins"}],
  "blockedMarketplaces": [{"source": "github", "repo": "untrusted/plugins"}],
  "pluginTrustMessage": "All plugins from acme-corp/plugins are approved by IT Security"
}
```

### Environment variables (enforce per-session)

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://otel.company.com:4318",
    "ANTHROPIC_LOG": "info",
    "NO_COLOR": "1"
  }
}
```

### Login / auth controls

```json
{
  "forceLoginMethod": "claudeai",
  "forceLoginOrgUUID": ["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"],
  "apiKeyHelper": "/bin/generate_temp_api_key.sh",
  "awsAuthRefresh": "aws sso login --profile myprofile",
  "awsCredentialExport": "/bin/generate_aws_grant.sh"
}
```

### UI / DX controls (company branding)

```json
{
  "companyAnnouncements": [
    "Welcome to Acme Corp! Review security guidelines at security.acme.com",
    "Reminder: Code reviews required for all AI-generated code",
    "Claude Code telemetry is enabled and monitored for compliance"
  ],
  "attribution": {
    "commit": "🤖 Generated with Claude Code (Acme Corp managed)",
    "pr": ""
  },
  "includeCoAuthoredBy": true,
  "minimumVersion": "2.1.100",
  "autoUpdatesChannel": "stable",
  "language": "english",
  "showThinkingSummaries": true,
  "cleanupPeriodDays": 60
}
```

### Git / worktree controls

```json
{
  "includeGitInstructions": true,
  "worktree": {
    "symlinkDirectories": ["node_modules", ".cache"],
    "sparsePaths": ["packages/my-app", "shared/utils"]
  }
}
```

### Telemetry / observability

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://otel.company.com:4318",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer ${OTEL_TOKEN}"
  },
  "otelHeadersHelper": "/bin/generate_otel_headers.sh",
  "feedbackSurveyRate": 0
}
```

---

## Merge vs override — что происходит когда одно и то же ключ в разных scope

| Тип поля | Merge behavior |
|---|---|
| `permissions.allow` / `ask` / `deny` | **Concat + dedup across scopes**, managed deny has highest precedence (overrides user/project allow if match) |
| `permissions.additionalDirectories` | Concat + dedup |
| `availableModels` | Concat + dedup. Для strict allowlist — set ТОЛЬКО в managed |
| `hooks` | Concat (каждый hook runs independently) — это точка для `allowManagedHooksOnly` |
| `env` | Later scope wins per-key, но managed — highest |
| `sandbox.filesystem.allowRead/allowWrite/denyRead/denyWrite` | Concat + dedup across ВСЕ scopes |
| `sandbox.network.allowedDomains/deniedDomains` | Concat + dedup |
| `allowedMcpServers` / `deniedMcpServers` | Concat + dedup |
| `companyAnnouncements` | Concat |
| `strictKnownMarketplaces` / `blockedMarketplaces` | Concat + dedup |
| Scalar values (`model`, `defaultMode`, `minimumVersion`, ...) | Highest scope wins (managed > CLI args > local > project > user) |
| `forceLoginMethod` / `forceLoginOrgUUID` | Managed-only, other scopes ignored |

**В итоге:** для **restrictive** policies — используй `deny` + `allowManagedPermissionRulesOnly: true` / `allowManagedMcpServersOnly: true` / `allowManagedHooksOnly: true` / `allowManagedReadPathsOnly: true` / `allowManagedDomainsOnly: true`. Тогда lower-scope settings не проникнут.

---

## Что НЕ enforceable через managed policy (важные gaps)

- ❌ **`claudeMdExcludes` content** — user может исключить **project-level** `.claude/rules/*.md` через user-scope `claudeMdExcludes` (feature request #34349 open, `claudeMdRequires` не реализован)
- ❌ **Actual AI outputs** — managed policy контролирует **что Клод может запустить**, но не **что он генерирует в тексте** (это свойство модели + CLAUDE.md guidance)
- ❌ **User's prompt content** — managed CLAUDE.md инструктирует Клода, но user всё равно может написать любой prompt
- ❌ **Hardware-level enforcement** — если user снимет диск и смонтирует на unmanaged machine, managed policy больше нет
- ❌ **Offline audit trail** — Compliance API работает только когда клиент talk к api.anthropic.com; offline usage не логируется централизованно (нужны local hooks + push позже)
- ❌ **Specific model behavior** (e.g., запретить Клоду "предлагать изменять prod DB") — это через CLAUDE.md / rules, не hard enforcement
- ❌ **Token usage / cost limit per developer** — нет прямого `maxTokensPerSession` setting. Monitor через Compliance API или OTel metrics.

---

## Per-scope limit (important edge cases)

- `autoMemoryDirectory` — **НЕ принимается** в project settings (security — чтобы shared repo не перенаправил memory writes в sensitive location). Работает в managed/local/user.
- `skipDangerousModePermissionPrompt` — **НЕ принимается** в project settings (чтобы untrusted repo не auto-bypass).
- `autoMode.allow/soft_deny` — **НЕ принимается** в shared project settings.

---

## /status + /permissions для debug

```
/status
```
Показывает:
- Какие settings sources активны (managed, user, project, local, command-line)
- Сколько hooks загружено из каждого
- Trust settings для current project

```
/permissions
```
Показывает full permission rules list с указанием source (managed/user/project). Полезно для debug "почему эта команда заблокирована".
