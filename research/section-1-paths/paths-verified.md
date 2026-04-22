# Section 1 — Managed Policy paths on all three OS (VERIFIED)

> **Language:** English · [Русский](paths-verified.ru.md)

**Source proofs:** https://docs.claude.com/en/docs/claude-code/settings (section "Settings files"), https://docs.anthropic.com/en/docs/claude-code/memory (section "Managed policy")

---

## Key distinction between managed-settings.json and Managed CLAUDE.md

In enterprise there are **TWO different files** with different purposes:

| File | Purpose | Bypassable? |
|---|---|---|
| **`managed-settings.json`** | Hard policy enforcement: permissions, MCP, models, plugins, hooks, sandbox | Cannot be bypassed, cannot be overridden |
| **Managed CLAUDE.md** | Behavioral instructions: how Claude behaves, company context | Cannot be excluded via `claudeMdExcludes`, BUT Claude may "read and ignore" them. These are **instructions**, not hard enforcement |

**Quote from docs.anthropic.com/memory:**
> A managed CLAUDE.md and managed settings serve different purposes. Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer.

**For enterprise, the primary mechanism in 95% of cases is `managed-settings.json`.** Managed CLAUDE.md is layered on top for "soft" company standards and awareness.

---

## macOS

### managed-settings.json (primary policy)
```
/Library/Application Support/ClaudeCode/managed-settings.json
/Library/Application Support/ClaudeCode/managed-settings.d/*.json   # drop-in fragments (v2.1.83+)
/Library/Application Support/ClaudeCode/managed-mcp.json            # separate file for MCP config
```

### Managed CLAUDE.md (behavioral)
```
/Library/Application Support/ClaudeCode/CLAUDE.md
```

### MDM preference domain
```
com.anthropic.claudecode
```
Deploy via configuration profiles: Jamf Pro, Kandji (Iru), Intune for Mac, ProfileCreator, iMazing Profile Editor. Top-level plist keys map 1:1 to `managed-settings.json`, nested objects become plist dicts, and arrays become plist arrays.

### Permission requirements
- The `/Library/Application Support/ClaudeCode/` directory requires **admin privileges** to write to → developers cannot modify it
- Even if the file is absent, the CLI still works (graceful degradation)
- If the file exists but read permission is missing, the CLI starts without managed policies (this is a feature, not a bug)

---

## Linux / WSL

### managed-settings.json
```
/etc/claude-code/managed-settings.json                # canonical (with a hyphen!)
/etc/claude-code/managed-settings.d/*.json            # drop-in
/etc/claude-code/managed-mcp.json
```

### Managed CLAUDE.md
```
/etc/claude-code/CLAUDE.md
```

### NO MDM domain for Linux
Anthropic does not provide native MDM integration for Linux. Deployment happens through **config-management tools:**
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

### Variant paths that **do not work**
- ❌ `/etc/claude/CLAUDE.md`: **wrong path**, shows up in some community posts. The canonical path is `/etc/claude-code/` (with a hyphen)
- ❌ `/usr/local/etc/claude-code/`: not supported
- ❌ `~/.claude/managed-settings.json`: this is **user-level**, not managed policy (it has no managed-only keys)

---

## Windows

### managed-settings.json (CHANGED in v2.1.75)
```
C:\Program Files\ClaudeCode\managed-settings.json                    # ✅ canonical (v2.1.75+)
C:\Program Files\ClaudeCode\managed-settings.d\*.json                # drop-in
C:\Program Files\ClaudeCode\managed-mcp.json
```

**⚠️ DEPRECATED (does not work as of v2.1.75):**
```
C:\ProgramData\ClaudeCode\managed-settings.json                      # ❌ obsolete
```

Direct quote from docs.claude.com/settings:
> The legacy Windows path `C:\ProgramData\ClaudeCode\managed-settings.json` is no longer supported as of v2.1.75. Administrators who deployed settings to that location must migrate files to `C:\Program Files\ClaudeCode\managed-settings.json`.

Some **community forks** of the documentation (`claude.yourdocs.dev`, mintlify forks) still show `ProgramData`. That is an old mirror, ignore it.

### Managed CLAUDE.md
```
C:\Program Files\ClaudeCode\CLAUDE.md
```

### Registry keys (MDM / GPO)
```
# Machine-level (admin, highest priority within the registry tier)
HKLM\SOFTWARE\Policies\ClaudeCode
  Settings  (REG_SZ or REG_EXPAND_SZ)  — JSON string with all settings

# User-level (lowest registry priority — used only if no admin source is present)
HKCU\SOFTWARE\Policies\ClaudeCode
  Settings  (REG_SZ or REG_EXPAND_SZ)
```

Deploy via:
- Group Policy (GPO Preferences → Registry)
- Microsoft Intune (Configuration Profile → Custom template → OMA-URI, or Win32 app + PowerShell script)
- PowerShell deployment:
```powershell
New-Item -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" `
  -Name "Settings" -Value '{"permissions":{"deny":["Bash(curl *)"]}}' -Type String
```

---

## Precedence inside the managed tier (highest → lowest)

1. **Server-managed settings** (Claude.ai admin console): available only on Teams/Enterprise plans
2. **MDM / OS-level policies:**
   - macOS plist (`com.anthropic.claudecode`)
   - Windows HKLM registry (`HKLM\SOFTWARE\Policies\ClaudeCode\Settings`)
3. **File-based:** `managed-settings.json` + `managed-settings.d/*.json` in the system directory
4. **HKCU registry** (Windows only, lowest managed priority)

**Critical:** only **one** managed source is active. If server-managed is set, MDM/file-based are ignored. Sources are **NOT merged** with each other (in contrast to drop-in fragments inside a single source).

---

## Drop-in directory `managed-settings.d/` (systemd-style)

Added in **v2.1.83**. Lets different teams push **independent fragments**:

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
- Scalars (string, number, boolean): the later file wins
- Arrays: concatenated + de-duplicated
- Objects: deep-merged

Hidden files (starting with `.`) are ignored. Use numeric prefixes to control ordering.

**Use case:** the security team owns `10-security.json`, DevOps owns `20-devops.json`, Compliance owns `30-audit.json`. Each team deploys its own file through its own CI/MDM pipeline, with no cross-team coordination required.

---

## Env variable for path override?

**Officially, no.** The Anthropic v2.1.x documentation contains **no mention** of `CLAUDE_MANAGED_SETTINGS_PATH` or any equivalent.

⚠️ Some community sources (mintlify forks, older guides) mention:
```bash
export CLAUDE_MANAGED_SETTINGS_PATH="/custom/path/managed-settings.json"
```
But this is **not confirmed** by official documentation. It may be an old feature that predates v2.1.x or a community myth. **Do not rely on it** unless validated by a direct test against the current CLI version.

---

## How to verify that it was applied

Inside Claude Code, run:
```
/status
```
The output shows **settings sources** and which managed layer is active:
```
Settings sources: Local, Enterprise managed policies, Command line arguments
```
If "Enterprise managed policies" appears in the list → managed-settings.json / server-managed / MDM is active.

Also:
```
/permissions
```
This shows every permission rule and the settings.json source it came from (managed / user / project / local).

---

## Anti-patterns: what NOT to do

- ❌ Do not put `managed-settings.json` in `~/.claude/`. That is user-level, managed-only keys will not work
- ❌ Do not use the old Windows path `C:\ProgramData\ClaudeCode\`. Deprecated as of v2.1.75
- ❌ Do not rely on the `CLAUDE_MANAGED_SETTINGS_PATH` env var. It is not officially documented
- ❌ Do not ship `managed-settings.json` via a project-level `.claude/settings.json`. It will not be promoted into the managed tier
- ❌ Do not combine multiple managed sources expecting them to merge. Only one is active
- ❌ Do not forget about `managed-settings.d/` when you need modularity (avoid a monolith)

---
