# Section 4 — Override / bypass attempts

> **Language:** English · [Русский](bypass-attempts.ru.md)

**Sources:**
- https://docs.anthropic.com/en/docs/claude-code/memory (managed CLAUDE.md cannot be excluded)
- https://docs.claude.com/en/docs/claude-code/settings (precedence hierarchy)
- https://github.com/anthropics/claude-code/issues/11872 (`--setting-source local` does not disable managed policies: by design)
- https://github.com/anthropics/claude-code/issues/34349 (`claudeMdRequires` feature request: not yet implemented)
- https://github.com/anthropics/claude-code/issues/20880 (parent CLAUDE.md exclusion)

---

## TL;DR

- **Managed CLAUDE.md cannot be excluded** via `claudeMdExcludes` (confirmed by official docs)
- **`--setting-source local` does NOT disable** managed policies (confirmed by Anthropic in bug #11872: "Enterprise policies are not intended to be overridable")
- **`--dangerously-skip-permissions` can be disabled** via managed `disableBypassPermissionsMode: "disable"`
- **`forceRemoteSettingsRefresh: true`** makes the CLI fail-closed on offline startup
- **But `claudeMdRequires` is not yet implemented** (issue #34349): a user can, via `claudeMdExcludes` in user scope, exclude **project-level** `.claude/rules/*.md` files. The only workaround is a monolithic managed CLAUDE.md.

---

## 1. `claudeMdExcludes` and managed CLAUDE.md

### Actual mechanics (from official docs.anthropic.com/memory)

> The `claudeMdExcludes` setting lets you skip specific files by path or glob pattern. [...]
> You can configure `claudeMdExcludes` at any settings layer: user, project, local, or managed policy. Arrays merge across layers.
> **Managed policy CLAUDE.md files cannot be excluded. This ensures** [that] organizational policy is always applied.

This means:
- Managed CLAUDE.md (`/Library/Application Support/ClaudeCode/CLAUDE.md` and equivalents) is **immune to `claudeMdExcludes`**
- Other CLAUDE.md tiers (user `~/.claude/CLAUDE.md`, project `./CLAUDE.md`, local `./CLAUDE.local.md`, subdirectory-level) **can be excluded**
- `.claude/rules/*.md` files (modular rules, including project-level `security-policy.md`, `compliance-*.md`) **can also be excluded** via `claudeMdExcludes` at user scope

### Practical gap (issue #34349)

**Scenario:**
1. Enterprise deploys a project-level `.claude/rules/security-policy.md` with mandatory security rules
2. A developer adds the following to their `~/.claude/settings.json`:
   ```json
   {"claudeMdExcludes": ["**/security-policy.md"]}
   ```
3. **The file is excluded**: `claudeMdExcludes` merges across layers, and this is permitted at user scope
4. Managed settings have no anti-exclusion mechanism

**Current workaround (Anthropic's official position):**
- Place all mandatory instructions in a **monolithic managed CLAUDE.md**: it cannot be excluded, but you lose modularity
- Use `.claude/rules/*.md` only for **guidance**, not for mandatory enforcement
- Enforce mandatory rules via `permissions.deny` in managed-settings.json (hard rules), not via CLAUDE.md guidance

**Proposal in the issue (not implemented as of 2026-04-21):**
```json
{
  "claudeMdRequires": [
    "**/security-policy.md",
    "**/compliance-*.md",
    ".claude/rules/audit-trail.md"
  ]
}
```
→ files listed in `claudeMdRequires` (managed-settings.json only) cannot be excluded at any scope.

---

## 2. CLI flag bypasses

### `--dangerously-skip-permissions`
```bash
claude --dangerously-skip-permissions "do something"
```
**Default:** bypasses all permission checks.

**Enterprise block:**
```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```
→ flag rejected at startup. `defaultMode: "bypassPermissions"` in user settings is also blocked. **Hard enforcement, managed can disable this.**

### `--permission-mode auto`
```bash
claude --permission-mode auto "do something"
```
Auto mode: an LLM classifier decides what to pre-approve.

**Enterprise block:**
```json
{
  "disableAutoMode": "disable"
}
```
→ Removes `auto` from the `Shift+Tab` cycle and rejects the flag.

### `--setting-source local`
```bash
claude --setting-source local "do something"
```
**User intent:** ignore enterprise policies.
**Reality (bug #11872, response by Anthropic @ashwin-ant):**
> This is expected behavior. Enterprise policies are not intended to be overridable.

`/status` still shows:
```
Setting sources: Local, Enterprise managed policies, Command line arguments
```

→ **Does NOT disable managed policies.** The user can only disable the user/project/local slots; the managed tier remains. **Hard enforcement by design.**

### `--model <name>`
```bash
claude --model claude-opus-4-6 "..."
```

**Enterprise block:**
```json
{
  "availableModels": ["sonnet", "haiku"]
}
```
→ If `--model opus` is passed but `opus` is not in `availableModels` (managed), it is rejected.

**Obscure bypass:** the `ANTHROPIC_MODEL` env var is also blocked by `availableModels`.

### `--settings <custom-path>`
```bash
claude --settings ./custom-settings.json "..."
```
Custom settings merge in as local project settings but do **NOT** replace managed settings. Managed remains active.

---

## 3. Environment variable bypasses

### Bypass attempts via env
```bash
export ANTHROPIC_MODEL=claude-opus-4-7        # blocked by availableModels
export CLAUDE_CODE_SKIP_PROMPT_HISTORY=1       # allowed (only changes history)
export CLAUDE_CODE_DISABLE_CLAUDE_MDS=1        # ⚠️ may disable ALL CLAUDE.md files, including managed (!)
export CLAUDE_MANAGED_SETTINGS_PATH=/tmp/lax.json  # ⚠️ unofficial, may or may not work
```

**Critical: `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1`:**
Mentioned in community references (`shanraisshan/claude-code-best-practice`). If it actually works, it **bypasses managed CLAUDE.md** (but NOT managed-settings.json enforcement). Needs verification against the current CLI version.

**Mitigation:**
Managed policy **cannot** block an env var at the client level (env is an OS-level construct). However, you can **deny env manipulation via managed policy**:
```json
{
  "permissions": {
    "deny": ["Bash(export *CLAUDE*)"]
  }
}
```
This is weak protection: if the user launches the CLI via a wrapper script with the env already set, a Bash deny rule will not help.

**Better mitigation at the OS level:**
- Managed shell profile (via MDM profile) plus a bash `PROMPT_COMMAND` env check
- Anti-tamper EDR solutions (CrowdStrike, SentinelOne) monitor env changes

---

## 4. File-level bypass

### User deletes managed CLAUDE.md / managed-settings.json
Requires **admin/root**: an ordinary developer cannot do this.

If deletion does happen (escalated privileges or unmanaged machine):
- CLI starts without managed policy → only user/project/local tiers apply
- Audit event: MDM (Jamf/Intune) can detect file drift via a compliance check and auto-redeploy within minutes

### User runs `chmod 000` on managed files
- If the user cannot read the managed policy file, the CLI **falls back**: it starts without the managed tier and shows a warning in `/status`
- This is graceful degradation, but **effectively the user has bypassed policy on that machine**

**Mitigation:**
- MDM compliance policies: if file permissions deviate from expected, auto-fix / auto-restore / alert
- File integrity monitoring (Jamf: Compliance Reporter; Intune: Compliance Policy with custom detection)
- Managed CLAUDE.md check via a `/status` hook:
  ```json
  {
    "hooks": {
      "SessionStart": [{
        "hooks": [{"type": "command", "command": "/usr/local/bin/verify-policy.sh"}]
      }]
    }
  }
  ```

### User mounts a read-only filesystem over `/etc/claude-code/`
Exotic bypass: mount a tmpfs with empty files over the managed policy directory. Requires root. EDR detects it.

---

## 5. Unmanaged device (BYOD)

**The main gap:** if a user runs Claude Code on a personal laptop that is not enrolled in MDM and not signed into a corporate account:
- No managed-settings.json → runs without enforcement
- No managed CLAUDE.md → no company context
- A user with a personal Claude subscription can do whatever they want

**Mitigation:**
1. **`forceLoginMethod: "claudeai"` + `forceLoginOrgUUID`** (server-managed): requires an org login for CLI startup, personal accounts are rejected. But this only works if the managed policy **is present** on the machine.
2. **Server-managed settings:** delivered via api.anthropic.com at org login. If the user is signed into an org account, they are pulled in even on a personal machine. But server-managed **requires** a Claude for Teams/Enterprise plan.
3. **Content filtering / audit via the Compliance API:** Enterprise can see all activity through the org account, regardless of device.
4. **Zero Trust approach:** block direct internet access from dev machines, force all traffic through a corporate proxy → the CLI's calls go through the proxy → you see everything.
5. **Endpoint posture:** treat MDM enrollment as a conditional access requirement. No enrollment → no corporate login → no managed settings.

---

## 6. Audit logging of bypass attempts

### What is logged officially
- **Compliance API (Enterprise plan only):** `prompts, conversations, model used, tool calls`. Available to the Primary Owner. Not a realtime per-event alert feed, but a full export.
- **Compliance API (audit events):** recently added: authentication, org admin changes, policy changes.
- **Hooks (self-configured):** if IT sets up `PreToolUse` / `PostToolUse` hooks, they log **every denied attempt** (the hook runs before the permission check and sees both allows and denies).

### What is NOT logged centrally (gap)
- Local attempts to pass `--dangerously-skip-permissions` (if managed policy is present, this is rejected at startup and is not an event)
- Attempts to edit managed-settings.json (this is an OS-level event, not a Claude-level one: you need OSQuery / EDR)
- Offline sessions on unmanaged devices (if the user is not signed into an org account)

### Recommended audit stack for enterprise
```
Claude Code CLI
    ↓ managed hooks
    PostToolUse hook → JSON event to stdout
    ↓ wrapped by
/usr/local/bin/audit-forward.sh → curl POST → Splunk HEC / Datadog Logs / ELK
    ↓ in parallel
Compliance API (pull scheduled) → archive → Snowflake / S3 for long-term retention
    ↓ file integrity
MDM compliance check every 4h → alert on drift
```

---

## 7. Read-access edge cases

### What if the user has no read access to /etc/claude-code/CLAUDE.md?

**Current behavior (current CLI):**
- CLI starts without the managed layer
- `/status` shows "Managed policies: not loaded (permission denied)" or similar
- User/project/local settings are active as usual
- **Effectively a bypass:** the user sees a "warning" but the CLI still works

This is **graceful degradation, not fail-closed.** For strict enterprise, combine with `forceRemoteSettingsRefresh: true` (although that is about server-managed, not file-based, settings).

**Mitigation:**
- Correct file permissions at deploy time (`chmod 644`: world-readable)
- MDM compliance check: if permissions drift, alert
- Periodic verification via a hook:
  ```bash
  #!/bin/bash
  if [[ ! -r "/Library/Application Support/ClaudeCode/managed-settings.json" ]]; then
    logger -t claude-code "ALERT: managed policy not readable!"
    exit 1
  fi
  ```

---

## 8. Questions to raise with Anthropic (for future updates)

1. Will `claudeMdRequires` be implemented? (issue #34349 open since Mar 2026)
2. Are per-group policies planned for server-managed settings?
3. Can a **fail-closed** mode be added for file-based managed policies (if the file exists but is not readable, fail CLI startup)?
4. Add an official managed event to the Compliance API when a user tries to run a denied command (currently only available via custom hooks).
5. Does the env var `CLAUDE_CODE_DISABLE_CLAUDE_MDS` exist, and does it work for managed policies?
