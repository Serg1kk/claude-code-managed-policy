# Section 4 — Override / bypass попытки

**Источники:**
- https://docs.anthropic.com/en/docs/claude-code/memory (managed CLAUDE.md cannot be excluded)
- https://docs.claude.com/en/docs/claude-code/settings (precedence hierarchy)
- https://github.com/anthropics/claude-code/issues/11872 (`--setting-source local` не отключает managed — by design)
- https://github.com/anthropics/claude-code/issues/34349 (claudeMdRequires feature request — ещё не реализован)
- https://github.com/anthropics/claude-code/issues/20880 (parent CLAUDE.md exclusion)

---

## TL;DR

- **Managed CLAUDE.md нельзя исключить** через `claudeMdExcludes` ✅ (подтверждено official docs)
- **`--setting-source local` НЕ отключает** managed policies ✅ (подтверждено Anthropic в bug #11872: "Enterprise policies are not intended to be overridable")
- **`--dangerously-skip-permissions` можно задизейблить** через managed `disableBypassPermissionsMode: "disable"`
- **`forceRemoteSettingsRefresh: true`** делает CLI fail-closed при offline startup
- **Но `claudeMdRequires` пока не реализован** (issue #34349) — user может через `claudeMdExcludes` в user-scope исключить **project-level** `.claude/rules/*.md` файлы. Единственный workaround — monolithic managed CLAUDE.md.

---

## 1. `claudeMdExcludes` и managed CLAUDE.md

### Фактическая механика (из official docs.anthropic.com/memory)

> The `claudeMdExcludes` setting lets you skip specific files by path or glob pattern. [...]
> You can configure `claudeMdExcludes` at any settings layer: user, project, local, or managed policy. Arrays merge across layers.
> **Managed policy CLAUDE.md files cannot be excluded. This ensures** [that] organizational policy is always applied.

Значит:
- ✅ Managed CLAUDE.md (`/Library/Application Support/ClaudeCode/CLAUDE.md` и аналоги) — **immune к claudeMdExcludes**
- ❌ Остальные CLAUDE.md уровней (user `~/.claude/CLAUDE.md`, project `./CLAUDE.md`, local `./CLAUDE.local.md`, subdirectory-level) — **можно исключить**
- ❌ `.claude/rules/*.md` (modular rules, включая project-level "security-policy.md", "compliance-*.md") — **тоже можно исключить** через `claudeMdExcludes` у user-scope

### Практическая дыра (issue #34349)

**Сценарий:**
1. Enterprise деплоит project-level `.claude/rules/security-policy.md` с mandatory security rules
2. Developer добавляет в свой `~/.claude/settings.json`:
   ```json
   {"claudeMdExcludes": ["**/security-policy.md"]}
   ```
3. **File исключается** — управляется же `claudeMdExcludes` merge across layers, и в user-scope это разрешено
4. Managed-settings не имеет анти-exclusion mechanism

**Текущий workaround (официальная позиция Anthropic):**
- Поместить все mandatory instructions в **monolithic managed CLAUDE.md** — его нельзя исключить, но теряется модульность
- Использовать `.claude/rules/*.md` только для **guidance**, не для mandatory enforcement
- Mandatory enforcement делать через `permissions.deny` в managed-settings.json (hard rules), не через CLAUDE.md guidance

**Предложение в issue (не реализовано на 2026-04-21):**
```json
{
  "claudeMdRequires": [
    "**/security-policy.md",
    "**/compliance-*.md",
    ".claude/rules/audit-trail.md"
  ]
}
```
→ файлы в `claudeMdRequires` (managed-settings.json only) нельзя exclude ни в одном scope.

---

## 2. CLI flags bypass

### `--dangerously-skip-permissions`
```bash
claude --dangerously-skip-permissions "do something"
```
**Default:** bypass всех permission checks.

**Enterprise block:**
```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```
→ flag rejected at startup. Также блокируется `defaultMode: "bypassPermissions"` в user settings. ✅ **Hard enforce, managed может это задизейблить.**

### `--permission-mode auto`
```bash
claude --permission-mode auto "do something"
```
Auto mode — LLM classifier решает что pre-approve.

**Enterprise block:**
```json
{
  "disableAutoMode": "disable"
}
```
→ Убирает `auto` из `Shift+Tab` cycle + flag rejected. ✅

### `--setting-source local`
```bash
claude --setting-source local "do something"
```
**Intent (user):** игнорировать enterprise policies.
**Реальность (bug #11872, response by Anthropic @ashwin-ant):**
> This is expected behavior. Enterprise policies are not intended to be overridable.

`/status` всё равно показывает:
```
Setting sources: Local, Enterprise managed policies, Command line arguments
```

→ **НЕ отключает managed**. User может отключить только user/project/local slots, но managed tier остаётся. ✅ **Hard enforce by design.**

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
→ Если `--model opus` передан, но `opus` не в `availableModels` (managed), rejected. ✅

**Obscure bypass:** `ANTHROPIC_MODEL` env var — тоже блокируется `availableModels`. ✅

### `--settings <custom-path>`
```bash
claude --settings ./custom-settings.json "..."
```
Custom settings mergers как local project settings, но **НЕ** заменяет managed. Managed всё равно active. ✅

---

## 3. Environment variables обход

### Попытки обхода через env
```bash
export ANTHROPIC_MODEL=claude-opus-4-7        # блокируется availableModels
export CLAUDE_CODE_SKIP_PROMPT_HISTORY=1       # разрешено (меняет только history)
export CLAUDE_CODE_DISABLE_CLAUDE_MDS=1        # ⚠️ может disable ВСЕ CLAUDE.md, включая managed (!)
export CLAUDE_MANAGED_SETTINGS_PATH=/tmp/lax.json  # ⚠️ неофициальный, может работать или нет
```

**Критично — `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1`:**
Упомянут в community references (`shanraisshan/claude-code-best-practice`). Если он реально работает — это **обходит managed CLAUDE.md** (но НЕ managed-settings.json enforcement). Нужна проверка в current CLI version.

**Mitigation:**
Managed policy **не может** заблокировать env var на уровне клиента (env — это OS-уровень). Но **запрет env через managed**:
```json
{
  "permissions": {
    "deny": ["Bash(export *CLAUDE*)"]
  }
}
```
Слабый защита — если user запускает CLI через wrapper script с уже выставленным env, deny на Bash не поможет.

**Better mitigation — на уровне ОС:**
- Managed shell profile (через MDM profile) + bash PROMPT_COMMAND проверка env
- Anti-tamper EDR-решения (CrowdStrike, SentinelOne) monitor env changes

---

## 4. File-level bypass

### User удаляет managed CLAUDE.md / managed-settings.json
Требует **admin/root** — обычный developer не может.

Если удаление сделано (escalated privileges или unmanaged machine):
- CLI стартует без managed → user/project/local только
- Audit event: MDM (Jamf/Intune) может детектить file drift через compliance check, auto-redeploy через minutes

### User делает `chmod 000` на managed файлы
- Если user не может read managed policy file — CLI **fallback**: стартует без managed tier, warning в `/status`
- Это graceful degradation, но **effectively user обошёл policy на этой машине**

**Mitigation:**
- MDM compliance policies — если file permissions !== expected, auto-fix / auto-restore / alert
- File integrity monitoring (Jamf: Compliance Reporter; Intune: Compliance Policy with custom detection)
- Managed CLAUDE.md check через `/status` hook:
  ```json
  {
    "hooks": {
      "SessionStart": [{
        "hooks": [{"type": "command", "command": "/usr/local/bin/verify-policy.sh"}]
      }]
    }
  }
  ```

### User монтирует read-only filesystem над `/etc/claude-code/`
Экзотический bypass — mount tmpfs с пустыми файлами поверх managed policy. Требует root. EDR detects.

---

## 5. Unmanaged device (BYOD)

**Главный gap:** если user запускает Claude Code на personal laptop, не enrolled в MDM, не залогинен в корп-аккаунт:
- Нет managed-settings.json → работает без enforcement
- Нет managed CLAUDE.md → без company context
- User с personal Claude subscription — может делать что угодно

**Mitigation:**
1. **`forceLoginMethod: "claudeai"` + `forceLoginOrgUUID`** (server-managed) — requires org login для CLI startup, personal аккаунт отвергается. Но это работает только если managed policy **есть** на машине.
2. **Server-managed settings** — доставляются через api.anthropic.com при org login. Если user залогинен в org-аккаунт, они подтягиваются даже на personal машине. Но server-managed **требует** Claude for Teams/Enterprise plan.
3. **Content filtering / audit через Compliance API** — Enterprise может видеть все activity через org-аккаунт, независимо от device.
4. **Zero Trust approach** — запретить direct internet access с dev machines, force всё через корп-proxy → CLI calls goes через proxy → вижу всё.
5. **Endpoint посторонится** — MDM enrollment как conditional access требование. Нет enrollment → нет корп login → нет managed settings.

---

## 6. Audit logging при попытке обойти

### Что логируется официально
- **Compliance API (Enterprise plan only):** `prompts, conversations, model used, tool calls`. Доступен Primary Owner. Не realtime alert per-event, но full export.
- **Compliance API (audit events):** недавно добавлено — аутентификация, org admin changes, policy changes.
- **hooks (self-configured):** если IT настроил `PreToolUse` / `PostToolUse` hooks, они логируют **каждую denied попытку** (hook runs before permission check, видит и allow, и deny).

### Что НЕ логируется централизованно (gap)
- Локальные попытки `--dangerously-skip-permissions` (если managed есть, это reject at startup — не event)
- Попытки редактировать managed-settings.json (это OS-level event, не Claude-level — нужен OSQuery / EDR)
- Offline sessions на unmanaged devices (если user не залогинен в org-аккаунт)

### Recommended audit stack для enterprise
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

### Что если user не имеет read на /etc/claude-code/CLAUDE.md?

**Current behavior (current CLI):**
- CLI стартует без managed layer
- `/status` показывает "Managed policies: not loaded (permission denied)" или аналогичное
- User/project/local settings активны как обычно
- **Effectively bypass** — user видит "warning", но CLI работает

Это **graceful degradation, не fail-closed.** Для strict enterprise — комбинировать с `forceRemoteSettingsRefresh: true` (хотя это про server-managed, не file-based).

**Mitigation:**
- Правильные file permissions при deploy (`chmod 644` — world-readable)
- MDM compliance check — если permissions drift, alert
- Periodically verify через hook:
  ```bash
  #!/bin/bash
  if [[ ! -r "/Library/Application Support/ClaudeCode/managed-settings.json" ]]; then
    logger -t claude-code "ALERT: managed policy not readable!"
    exit 1
  fi
  ```

---

## 8. Что можно спросить у Anthropic (для будущих обновлений)

1. Будет ли реализован `claudeMdRequires`? (issue #34349 open c Mar 2026)
2. Планируется ли per-group policy в server-managed settings?
3. Можно ли добавить **fail-closed** mode для file-based managed (если файл есть, но не readable — fail CLI startup)?
4. Добавить official managed event в Compliance API когда user пытается run denied command (сейчас только через custom hooks)
5. Есть ли env var `CLAUDE_CODE_DISABLE_CLAUDE_MDS` — работает ли для managed?
