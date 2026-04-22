# Section 3 — What is enforceable through Managed Policy

> **Language:** English · [Русский](enforceable-keys.ru.md)

**Sources:**
- https://docs.claude.com/en/docs/claude-code/settings (full reference)
- https://code.claude.com/docs/en/permissions.md (managed-only keys)
- https://managed-settings.net/ (community reference builder: the best UI)

---

## Managed-only keys (do not work outside managed-settings.json)

These keys are **read only from managed-settings.json / server-managed / MDM**. If placed in user or project settings, they are silently ignored.

| Key | Description | Example value |
|---|---|---|
| `allowManagedMcpServersOnly` | Only MCP servers from the managed `allowedMcpServers` list work. A user can add their own entries to `.mcp.json`, but they will not load | `true` |
| `allowManagedPermissionRulesOnly` | Only permission rules (allow/ask/deny) from managed settings apply. User/project rules are ignored | `true` |
| `allowManagedHooksOnly` | Only hooks from managed + SDK + force-enabled plugins run. User/project/plugin hooks are blocked | `true` |
| `allowedChannelPlugins` | Allowlist of channel plugins (Team/Enterprise "channels" feature) | `[{"marketplace": "claude-plugins-official", "plugin": "telegram"}]` |
| `blockedMarketplaces` | Blocklist of plugin marketplace sources. **Not downloaded** | `[{"source": "github", "repo": "untrusted/plugins"}]` |
| `strictKnownMarketplaces` | Allowlist of marketplaces. An empty array = lockdown (no marketplaces at all) | `[{"source": "github", "repo": "acme-corp/plugins"}]` |
| `channelsEnabled` | Enable channels (Teams/Enterprise). Unset or false = blocked | `true` |
| `forceLoginMethod` | `"claudeai"` or `"console"`: restrict the account type | `"claudeai"` |
| `forceLoginOrgUUID` | UUID of the corporate org (single value or array). User cannot sign in to any org other than this | `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"` |
| `forceRemoteSettingsRefresh` | CLI startup is blocked until fresh server-managed settings are received. Fail-closed | `true` |
| `pluginTrustMessage` | Custom message shown in the warning when a plugin is installed | `"All plugins from our marketplace are approved by IT"` |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Only managed paths are honored in `filesystem.allowRead`; user paths are ignored. `denyRead` still merges across all levels | `true` |
| `network.allowManagedDomainsOnly` | Only managed `allowedDomains` + WebFetch allow rules apply. User domains are ignored. Denied entries still merge | `true` |
| `skipWebFetchPreflight` | Skip the WebFetch blocklist preflight (for restrictive network environments) | `true` |
| `sshConfigs` (managed mode) | SSH connections for the Desktop environment. When set from managed, read-only for the user | `[{"id": "dev-vm", "name": "Dev VM", "sshHost": "user@dev.example.com"}]` |

---

## Settings that work in any scope, but in managed get highest precedence

These keys are read from any scope, but managed-settings.json cannot be overridden. In other words, IT can **force-set** any of them.

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
- `Tool`: match all invocations of the tool
- `Tool(specifier)`: narrower match
- `Bash(npm run *)`: any `npm run ...` command
- `Read(./.env)`: a specific file
- `Read(./secrets/**)`: a folder, recursively
- `WebFetch(domain:example.com)`: requests to a domain
- `Agent(name-pattern)`: subagent invocations
- `MCP(serverName)`: MCP tool calls

**Precedence:** deny > ask > allow (first match wins). A managed deny **cannot be bypassed** by a lower-scope allow.

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

- `disableBypassPermissionsMode: "disable"`: blocks the `--dangerously-skip-permissions` CLI flag and `defaultMode: "bypassPermissions"`
- `disableAutoMode: "disable"`: removes `auto` from the `Shift+Tab` cycle; `--permission-mode auto` is rejected at startup
- `disableSkillShellExecution: true`: blocks `!` shell execution in skills and custom commands (user/project/plugin/additional-directory sources). **Bundled and managed skills are not affected.**
- `disableAllHooks`: fully blocks hooks and the custom statusline

### Model restrictions

```json
{
  "availableModels": ["sonnet", "haiku"],
  "model": "sonnet"
}
```

- `availableModels`: allowlist of models. The user cannot `/model`-switch to anything not listed. Also applies to the CLI `--model` flag, the `ANTHROPIC_MODEL` env var, and the Config tool.
- `model`: initial default for the session (not enforcement: user may switch within `availableModels`)
- Merge: arrays are concatenated and deduplicated across all scopes. **For a strict allowlist**, set `availableModels` in managed **only**.
- The **Default model picker option** is not affected by `availableModels`: it is always available.

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

- `failIfUnavailable: true` + `enabled: true`: a **hard gate**. The CLI exits if the sandbox is not available (missing dependencies, unsupported platform). Without `failIfUnavailable` it is merely a warning plus unsandboxed execution.
- `allowUnsandboxedCommands: false`: the `dangerouslyDisableSandbox` escape hatch is **fully disabled**
- `allowManagedReadPathsOnly: true`: user paths in `filesystem.allowRead` are ignored
- `allowManagedDomainsOnly: true`: user domains in `allowedDomains` and `WebFetch(domain:...)` are ignored

### Hooks (for audit logging)

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
- `PreToolUse`: before the permission check (can approve or deny, bypassing the permission system)
- `PostToolUse`: after successful execution
- `SessionStart`: when the CLI starts
- `SessionEnd`
- `UserPromptSubmit`

Hooks execute shell commands on the user's machine, so a security review matters. Managed `hooks` display a security-approval dialog to the user on first run (which may increment runtime permission).

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

`managed-mcp.json` (a separate file) is for configuring the MCP servers themselves (not just allow/deny, but the full config with command/args/env). Example:
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

## Merge vs override: what happens when the same key appears in different scopes

| Field type | Merge behavior |
|---|---|
| `permissions.allow` / `ask` / `deny` | **Concat + dedup across scopes**; managed deny has highest precedence (overrides a user/project allow on a match) |
| `permissions.additionalDirectories` | Concat + dedup |
| `availableModels` | Concat + dedup. For a strict allowlist, set ONLY in managed |
| `hooks` | Concat (each hook runs independently): this is the lever for `allowManagedHooksOnly` |
| `env` | Later scope wins per key, but managed is the highest |
| `sandbox.filesystem.allowRead/allowWrite/denyRead/denyWrite` | Concat + dedup across ALL scopes |
| `sandbox.network.allowedDomains/deniedDomains` | Concat + dedup |
| `allowedMcpServers` / `deniedMcpServers` | Concat + dedup |
| `companyAnnouncements` | Concat |
| `strictKnownMarketplaces` / `blockedMarketplaces` | Concat + dedup |
| Scalar values (`model`, `defaultMode`, `minimumVersion`, ...) | Highest scope wins (managed > CLI args > local > project > user) |
| `forceLoginMethod` / `forceLoginOrgUUID` | Managed-only; other scopes are ignored |

**Bottom line:** for **restrictive** policies, use `deny` together with `allowManagedPermissionRulesOnly: true` / `allowManagedMcpServersOnly: true` / `allowManagedHooksOnly: true` / `allowManagedReadPathsOnly: true` / `allowManagedDomainsOnly: true`. That way lower-scope settings will not leak through.

---

## What is NOT enforceable through managed policy (important gaps)

- **`claudeMdExcludes` content**: a user can exclude **project-level** `.claude/rules/*.md` via user-scope `claudeMdExcludes` (feature request #34349 is open, `claudeMdRequires` is not implemented)
- **Actual AI outputs**: managed policy controls **what Claude is allowed to run**, but not **what it generates as text** (that is a property of the model plus CLAUDE.md guidance)
- **User's prompt content**: a managed CLAUDE.md instructs Claude, but the user can still write any prompt
- **Hardware-level enforcement**: if the user removes the disk and mounts it on an unmanaged machine, managed policy is gone
- **Offline audit trail**: the Compliance API only works while the client is talking to api.anthropic.com; offline usage is not logged centrally (you need local hooks plus a later push)
- **Specific model behavior** (e.g., forbid Claude from "suggesting changes to the prod DB"): this goes through CLAUDE.md / rules, not hard enforcement
- **Token usage / cost limit per developer**: there is no direct `maxTokensPerSession` setting. Monitor via the Compliance API or OTel metrics.

---

## Per-scope limits (important edge cases)

- `autoMemoryDirectory`: **not accepted** in project settings (a security measure, so a shared repo cannot redirect memory writes to a sensitive location). Works in managed/local/user.
- `skipDangerousModePermissionPrompt`: **not accepted** in project settings (so an untrusted repo cannot auto-bypass).
- `autoMode.allow/soft_deny`: **not accepted** in shared project settings.

---

## /status and /permissions for debugging

```
/status
```
Shows:
- Which settings sources are active (managed, user, project, local, command-line)
- How many hooks are loaded from each
- Trust settings for the current project

```
/permissions
```
Shows the full list of permission rules with the source (managed/user/project). Useful for debugging "why is this command blocked".
