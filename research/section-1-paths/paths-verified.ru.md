# Section 1 — Точные пути Managed Policy на трёх OS (VERIFIED)

> **Язык:** [English](paths-verified.md) · Русский

**Источник-пруф:** https://docs.claude.com/en/docs/claude-code/settings (раздел "Settings files"), https://docs.anthropic.com/en/docs/claude-code/memory (раздел "Managed policy")

---

## Ключевое различие между managed-settings.json и Managed CLAUDE.md

В enterprise есть **ДВА разных файла** с разным назначением:

| Файл | Назначение | Обход? |
|---|---|---|
| **`managed-settings.json`** | Hard policy enforcement — permissions, MCP, models, plugins, hooks, sandbox | Нельзя обойти, cannot be overridden |
| **Managed CLAUDE.md** | Behavioral instructions — как Клод себя ведёт, company context | Нельзя исключить через `claudeMdExcludes`, НО Claude может их "прочитать и проигнорировать" — это **инструкции**, не hard enforcement |

**Цитата из docs.anthropic.com/memory:**
> A managed CLAUDE.md and managed settings serve different purposes. Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer.

**Для enterprise в 95% случаев primary механизм — `managed-settings.json`.** Managed CLAUDE.md добавляется сверху для "soft" company standards и awareness.

---

## macOS

### managed-settings.json (primary policy)
```
/Library/Application Support/ClaudeCode/managed-settings.json
/Library/Application Support/ClaudeCode/managed-settings.d/*.json   # drop-in fragments (v2.1.83+)
/Library/Application Support/ClaudeCode/managed-mcp.json            # отдельный файл для MCP config
```

### Managed CLAUDE.md (behavioral)
```
/Library/Application Support/ClaudeCode/CLAUDE.md
```

### MDM preference domain
```
com.anthropic.claudecode
```
Deploy через configuration profiles: Jamf Pro, Kandji (Iru), Intune for Mac, ProfileCreator, iMazing Profile Editor. Top-level keys plist'а 1:1 mapping к `managed-settings.json`, nested = plist dict, arrays = plist arrays.

### Требования к permissions
- Директория `/Library/Application Support/ClaudeCode/` требует **admin privileges** для записи → developer не может modify
- Даже если файл отсутствует, CLI работает (graceful degradation)
- Если файл есть но нет read-permission — CLI стартует без managed policies (это фича, не баг)

---

## Linux / WSL

### managed-settings.json
```
/etc/claude-code/managed-settings.json                # canonical (с дефисом!)
/etc/claude-code/managed-settings.d/*.json            # drop-in
/etc/claude-code/managed-mcp.json
```

### Managed CLAUDE.md
```
/etc/claude-code/CLAUDE.md
```

### NO MDM domain для Linux
Anthropic не предоставляет нативного MDM integration для Linux — deployment идёт через **config-management tools:**
- Ansible playbooks
- Puppet / Chef
- Shell scripts + SSH rollout
- systemd-style config management

Community playbooks (systemprompt.io guide):
```yaml
# Ansible example
- name: Deploy Claude Code managed settings
  hosts: developers
  become: yes
  tasks:
    - name: Ensure managed directory exists
      file:
        path: /etc/claude-code
        state: directory
        mode: '0755'
        owner: root
        group: root
    - name: Deploy managed-settings.json
      template:
        src: managed-settings.json.j2
        dest: /etc/claude-code/managed-settings.json
        mode: '0644'
        owner: root
        group: root
      notify: verify settings
```

### Variant paths — которые **не работают**
- ❌ `/etc/claude/CLAUDE.md` — **неверный путь**, встречается в некоторых community-постах; canonical — `/etc/claude-code/` (с дефисом)
- ❌ `/usr/local/etc/claude-code/` — не поддерживается
- ❌ `~/.claude/managed-settings.json` — это **user-level**, не managed policy (не имеет managed-only-keys)

---

## Windows

### managed-settings.json (ИЗМЕНИЛОСЬ в v2.1.75)
```
C:\Program Files\ClaudeCode\managed-settings.json                    # ✅ canonical (v2.1.75+)
C:\Program Files\ClaudeCode\managed-settings.d\*.json                # drop-in
C:\Program Files\ClaudeCode\managed-mcp.json
```

**⚠️ DEPRECATED (не работает с v2.1.75):**
```
C:\ProgramData\ClaudeCode\managed-settings.json                      # ❌ устарело
```

Прямая цитата из docs.claude.com/settings:
> The legacy Windows path `C:\ProgramData\ClaudeCode\managed-settings.json` is no longer supported as of v2.1.75. Administrators who deployed settings to that location must migrate files to `C:\Program Files\ClaudeCode\managed-settings.json`.

Некоторые **community-форки** документации (`claude.yourdocs.dev`, mintlify-форки) всё ещё показывают `ProgramData` — это старое зеркало, ignore.

### Managed CLAUDE.md
```
C:\Program Files\ClaudeCode\CLAUDE.md
```

### Registry keys (MDM / GPO)
```
# Machine-level (admin, highest priority within registry tier)
HKLM\SOFTWARE\Policies\ClaudeCode
  Settings  (REG_SZ или REG_EXPAND_SZ)  — JSON string со всеми настройками

# User-level (lowest registry priority — используется только если admin source отсутствует)
HKCU\SOFTWARE\Policies\ClaudeCode
  Settings  (REG_SZ или REG_EXPAND_SZ)
```

Deploy через:
- Group Policy (GPO Preferences → Registry)
- Microsoft Intune (Configuration Profile → Custom template → OMA-URI, или Win32 app + PowerShell script)
- PowerShell deployment:
```powershell
New-Item -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" `
  -Name "Settings" -Value '{"permissions":{"deny":["Bash(curl *)"]}}' -Type String
```

---

## Precedence внутри managed tier (highest → lowest)

1. **Server-managed settings** (Claude.ai admin console) — available только Teams/Enterprise plans
2. **MDM / OS-level policies:**
   - macOS plist (`com.anthropic.claudecode`)
   - Windows HKLM registry (`HKLM\SOFTWARE\Policies\ClaudeCode\Settings`)
3. **File-based:** `managed-settings.json` + `managed-settings.d/*.json` в system directory
4. **HKCU registry** (Windows only, lowest managed priority)

**Критично:** только **один** managed source активен. Если server-managed задан — MDM/file-based игнорируются. Источники **НЕ мёрджатся** между собой (в отличие от drop-in фрагментов внутри одного source).

---

## Drop-in directory `managed-settings.d/` (systemd-style)

Добавлено в **v2.1.83**. Позволяет разным командам пушить **независимые фрагменты**:

```
/Library/Application Support/ClaudeCode/
├── managed-settings.json           # base — lowest priority
└── managed-settings.d/
    ├── 10-security.json            # merged first
    ├── 20-model-policy.json        # merged on top (wins over 10-)
    ├── 30-telemetry.json
    └── 50-mcp-allowlist.json
```

**Merge rules:**
- Scalar (string, number, boolean) — later file wins
- Arrays — concatenated + de-duplicated
- Objects — deep-merged

Hidden files (starting with `.`) ignored. Use numeric prefixes для управления порядком.

**Usecase:** security team владеет `10-security.json`, DevOps владеет `20-devops.json`, Compliance — `30-audit.json`. Каждая команда деплоит свою через свой CI/MDM pipeline, без координации.

---

## Env variable для path override?

**Официально — нет.** В документации Anthropic v2.1.x **нет упоминания** `CLAUDE_MANAGED_SETTINGS_PATH` или аналога.

⚠️ В некоторых community-источниках (mintlify forks, старые guides) упоминается:
```bash
export CLAUDE_MANAGED_SETTINGS_PATH="/custom/path/managed-settings.json"
```
— но это **не подтверждено** официальной документацией. Возможно это старая фича до v2.1.x или community-миф. **Не полагаться** пока не валидировано прямым тестом на текущей версии CLI.

---

## Как проверить что applied

Внутри Claude Code запустить:
```
/status
```
Вывод покажет **source settings** и какой managed layer активен:
```
Settings sources: Local, Enterprise managed policies, Command line arguments
```
Если "Enterprise managed policies" в списке → managed-settings.json / server-managed / MDM активен.

Также:
```
/permissions
```
— покажет все permission rules и из какого settings.json источника они пришли (managed / user / project / local).

---

## Anti-patterns — чего НЕ делать

- ❌ Не класть `managed-settings.json` в `~/.claude/` — это user-level, managed-only-keys не работают
- ❌ Не использовать старый Windows path `C:\ProgramData\ClaudeCode\` — deprecated с v2.1.75
- ❌ Не полагаться на `CLAUDE_MANAGED_SETTINGS_PATH` env var — не документирован официально
- ❌ Не пушить `managed-settings.json` через project-level `.claude/settings.json` — не поднимется в managed tier
- ❌ Не комбинировать несколько managed sources думая что они сольются — только один активен
- ❌ Не забыть про `managed-settings.d/` если нужна модульность (избегать монолита)

---
