# Section 2 — MDM distribution механика

**Источники:**
- https://docs.claude.com/en/docs/claude-code/settings#managed-settings
- https://code.claude.com/docs/en/server-managed-settings.md
- https://github.com/anthropics/claude-code/tree/main/examples/mdm (official starter templates)
- https://addigy.com/blog/manage-claude-code-policies-addigy/ (Apple-first MDM)
- https://systemprompt.io/guides/enterprise-claude-code-managed-settings
- https://systemprompt.io/guides/claude-code-organisation-rollout

---

## Три канала доставки (по official docs)

### 1. Server-managed settings (NEW, Q4 2025)

**Что это:** админ в Claude.ai → Admin Settings → Claude Code → Managed settings → пишет JSON, клиент автоматом подтягивает при логине или hourly polling.

**Требования:**
- Claude for Teams plan (CLI v2.1.38+) или Claude for Enterprise plan (v2.1.30+)
- Клиент имеет network access к `api.anthropic.com`
- **НЕ требует** MDM, device management infrastructure, admin access к ОС

**Плюсы:**
- Работает на BYOD / unmanaged devices (где IT не ставит Jamf)
- Changes deploy мгновенно (при restart / hourly poll)
- Нет задержек на MDM push

**Минусы / limitations:**
- Только uniform per org (нет per-group policies — это всё ещё limitation v2.1.x)
- Requires Claude for Teams / Enterprise plan (not available on Pro individual subscriptions)
- Settings полностью **offline** недоступны — нужен live connection к api.anthropic.com при fetch
- Если полагаться + `forceRemoteSettingsRefresh: true` — CLI **fail-closed** при offline startup
- **Нельзя использовать** при работе через сторонних провайдеров (Bedrock / Vertex / Foundry) — только direct Anthropic API

**Configure flow:**
```
1. Organization Owner / Primary Owner → claude.ai → Admin Settings → Claude Code → Managed settings
2. Paste JSON (same schema as managed-settings.json)
3. Save → rollout ~1 hour (next user startup или polling cycle)
```

Пример (from official docs):
```json
{
  "permissions": {
    "deny": ["Bash(curl *)", "Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"]
  },
  "disableBypassPermissionsMode": "disable",
  "allowManagedPermissionRulesOnly": true
}
```

### 2. MDM / OS-level policies

#### macOS через Jamf Pro / Kandji / Intune for Mac / Addigy

**Механика:** Configuration Profile деплоится на managed preferences domain `com.anthropic.claudecode`. plist top-level keys 1:1 mapping к managed-settings.json.

**Jamf Pro пошагово:**
```
Jamf Pro Console → Computers → Configuration Profiles → + New
  → Application & Custom Settings payload
  → Preference Domain: com.anthropic.claudecode
  → Upload plist file (или использовать Jamf Pro profile editor)
  → Scope: Developer Smart Group (не all-fleet)
  → Deploy
```

Либо через **Files and Processes** payload — drops `managed-settings.json` прямо в `/Library/Application Support/ClaudeCode/`.

**Kandji пошагово:**
```
Kandji → Library → Custom Profile → Upload .mobileconfig
  или
Library → Custom Script (Audit & Enforce):
  - Script checks file hash
  - Replaces if drift detected
```

**Addigy пошагово** (из https://addigy.com/blog/manage-claude-code-policies-addigy/):
```bash
# Installation Script
#!/bin/bash
mkdir -p "/Library/Application Support/ClaudeCode"
cat > "/Library/Application Support/ClaudeCode/managed-settings.json" << 'EOF'
{ ... policy JSON ... }
EOF
chown root:wheel "/Library/Application Support/ClaudeCode/managed-settings.json"
chmod 644 "/Library/Application Support/ClaudeCode/managed-settings.json"

# Condition (check if redeploy needed)
#!/bin/bash
POLICY_PATH="/Library/Application Support/ClaudeCode/managed-settings.json"
EXPECTED_HASH="sha256-of-current-policy"
[[ $(shasum -a 256 "$POLICY_PATH" 2>/dev/null | awk '{print $1}') == "$EXPECTED_HASH" ]] && exit 0 || exit 1
```

**Intune for Mac:** Configuration profile → Custom Template → upload `.mobileconfig` targeting `com.anthropic.claudecode` preference domain.

#### Windows через Group Policy / Microsoft Intune

**Механика:** REG_SZ JSON в `HKLM\SOFTWARE\Policies\ClaudeCode\Settings` (machine-level) или `HKCU\SOFTWARE\Policies\ClaudeCode\Settings` (user-level fallback).

**Group Policy:**
```
Group Policy Management Editor →
  Computer Configuration → Preferences → Windows Settings → Registry → New:
    Hive: HKEY_LOCAL_MACHINE
    Key Path: SOFTWARE\Policies\ClaudeCode
    Value Name: Settings
    Value Type: REG_SZ
    Value Data: {"permissions":{"deny":["Bash(curl *)"]}}
  → Link GPO to developer OU
```

**Intune через Configuration Profile:**
```
Intune → Devices → Configuration Profiles → + Create → Windows 10+
  → Templates → Administrative Templates → нет (нет Claude Code ADMX)
  → Settings catalog → Registry-based custom:
       OMA-URI: ./Device/Vendor/MSFT/Registry/HKLM/SOFTWARE/Policies/ClaudeCode/Settings
       Data type: String
       Value: <JSON>
```

Или через **Win32 app** packaging `managed-settings.json` + PowerShell install script.

**PowerShell deployment (бейс):**
```powershell
# На target machine под admin
$settings = @{
    permissions = @{
        deny = @("Bash(curl *)", "Read(./.env)", "Read(./.env.*)")
    }
    disableBypassPermissionsMode = "disable"
    allowManagedPermissionRulesOnly = $true
} | ConvertTo-Json -Depth 10 -Compress

New-Item -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" `
    -Name "Settings" -Value $settings -Type String
```

### 3. File-based deployment (config-management tools)

#### Ansible (Linux / macOS)

```yaml
# roles/claude-code-policy/tasks/main.yml
- name: Ensure Claude Code managed directory exists
  file:
    path: "{{ claude_managed_dir }}"
    state: directory
    mode: '0755'
    owner: root
    group: "{{ 'wheel' if ansible_os_family == 'Darwin' else 'root' }}"
  vars:
    claude_managed_dir: "{{ '/Library/Application Support/ClaudeCode' if ansible_os_family == 'Darwin' else '/etc/claude-code' }}"

- name: Deploy managed-settings.json
  template:
    src: managed-settings.json.j2
    dest: "{{ claude_managed_dir }}/managed-settings.json"
    mode: '0644'
    owner: root
    backup: yes
  notify: validate claude code settings

- name: Deploy managed CLAUDE.md (behavioral layer)
  template:
    src: CLAUDE.md.j2
    dest: "{{ claude_managed_dir }}/CLAUDE.md"
    mode: '0644'
    owner: root

- name: Deploy security policy fragment (modular)
  template:
    src: "policy-fragments/10-security.json.j2"
    dest: "{{ claude_managed_dir }}/managed-settings.d/10-security.json"
    mode: '0644'
    owner: root

# handlers
- name: validate claude code settings
  shell: "jq empty {{ claude_managed_dir }}/managed-settings.json"
  register: jq_result
  failed_when: jq_result.rc != 0
```

#### Puppet

```puppet
# modules/claude_code/manifests/policy.pp
class claude_code::policy (
  String $policy_json,
  Optional[String] $claude_md = undef,
) {
  $managed_dir = $facts['os']['family'] ? {
    'Darwin'  => '/Library/Application Support/ClaudeCode',
    'Debian'  => '/etc/claude-code',
    'RedHat'  => '/etc/claude-code',
    default   => fail("Unsupported OS ${facts['os']['family']}"),
  }

  file { $managed_dir:
    ensure => directory,
    owner  => 'root',
    group  => $facts['os']['family'] ? { 'Darwin' => 'wheel', default => 'root' },
    mode   => '0755',
  }

  file { "${managed_dir}/managed-settings.json":
    ensure  => file,
    content => $policy_json,
    owner   => 'root',
    mode    => '0644',
    require => File[$managed_dir],
    notify  => Exec['validate_claude_settings'],
  }

  exec { 'validate_claude_settings':
    command     => "/usr/bin/jq empty ${managed_dir}/managed-settings.json",
    refreshonly => true,
    path        => ['/usr/bin', '/usr/local/bin', '/bin'],
  }

  if $claude_md {
    file { "${managed_dir}/CLAUDE.md":
      ensure  => file,
      content => $claude_md,
      owner   => 'root',
      mode    => '0644',
    }
  }
}
```

#### Chef

```ruby
# cookbooks/claude_code/recipes/policy.rb
managed_dir = case node['platform_family']
  when 'mac_os_x' then '/Library/Application Support/ClaudeCode'
  when 'debian', 'rhel' then '/etc/claude-code'
end

directory managed_dir do
  owner 'root'
  group platform_family?('mac_os_x') ? 'wheel' : 'root'
  mode '0755'
  recursive true
end

template "#{managed_dir}/managed-settings.json" do
  source 'managed-settings.json.erb'
  owner 'root'
  mode '0644'
  variables(
    deny_rules: node['claude_code']['policy']['deny'],
    allow_rules: node['claude_code']['policy']['allow'],
    allowed_models: node['claude_code']['allowed_models']
  )
  notifies :run, 'execute[validate_settings]', :immediately
end

execute 'validate_settings' do
  command "jq empty #{managed_dir}/managed-settings.json"
  action :nothing
end
```

#### Shell script (простейший вариант)

```bash
#!/bin/bash
# deploy-claude-policy.sh — для small teams без configuration management
set -euo pipefail

if [[ "$OSTYPE" == "darwin"* ]]; then
  MANAGED_DIR="/Library/Application Support/ClaudeCode"
elif [[ "$OSTYPE" == "linux-gnu"* ]]; then
  MANAGED_DIR="/etc/claude-code"
else
  echo "Unsupported OS: $OSTYPE" >&2
  exit 1
fi

sudo mkdir -p "$MANAGED_DIR/managed-settings.d"
sudo cp managed-settings.json "$MANAGED_DIR/"
sudo cp managed-settings.d/*.json "$MANAGED_DIR/managed-settings.d/"
sudo cp CLAUDE.md "$MANAGED_DIR/"
sudo chmod 644 "$MANAGED_DIR"/*.json "$MANAGED_DIR"/*.md

# Validate
jq empty "$MANAGED_DIR/managed-settings.json" && echo "✅ valid"
for f in "$MANAGED_DIR/managed-settings.d/"*.json; do
  jq empty "$f" && echo "✅ $f valid"
done
```

---

## Official Anthropic templates

**Repo:** https://github.com/anthropics/claude-code/tree/main/examples/mdm

Включает стартерные шаблоны для:
- **Jamf Pro** — `.mobileconfig` profile
- **Kandji (Iru)** — Custom Profile format
- **Microsoft Intune** — Configuration Profile JSON
- **Group Policy (GPO)** — ADMX/ADML templates

Для Linux официальных stater-ов **нет** — используйте Ansible/Puppet/Chef по примерам выше.

---

## Phased rollout playbook (60 разработчиков fintech)

**Из https://systemprompt.io/guides/claude-code-organisation-rollout:**

| Неделя | Что делать | Аудитория |
|---|---|---|
| **Week 1** | Написать draft policy, версионировать в git, code-review от security team | 0 devs (admin prep) |
| **Week 2** | Pilot group: 5-10 senior devs. Deploy через Jamf/Intune pilot scope. Сбор feedback, tweaks policy | 5-10 |
| **Week 3** | Expand pilot до 20 devs + mandatory onboarding session. Добавить Managed CLAUDE.md для company context | 20 |
| **Week 4** | Full rollout на all developer scope. Enable `forceRemoteSettingsRefresh` если нужен fail-closed | 60 |
| **Week 5+** | Ongoing: policy changes через PR в git-репо, rollout через MDM, audit log review weekly | all |

**Rollback strategy:**
1. Держать prev policy versioned в git (например, `managed-settings-v1.2.json`)
2. При проблеме — push prev version через MDM (Jamf/Intune revert)
3. Для server-managed — revert через Claude.ai admin console (хранит history)
4. Emergency kill-switch — deploy пустой `{}` — managed policy effective minimum

---

## Distinguishing server-managed vs endpoint-managed — когда какой использовать

| Сценарий | Рекомендация |
|---|---|
| BYOD / unmanaged laptops | Server-managed (не требует MDM) |
| Enterprise Mac fleet с Jamf | Endpoint-managed через Jamf (tamper-resistant на OS-level) |
| Windows enterprise с SCCM/Intune | Endpoint-managed через Intune/GPO |
| Mixed Mac + Windows + Linux | Server-managed (unified, но требует Enterprise plan) |
| Air-gapped / offline environments | Endpoint-managed (server-managed требует api.anthropic.com) |
| Startup, нет IT-инфраструктуры | Server-managed (если есть Teams plan) или file-based через script |
| Contractors / vendors | Server-managed + `forceLoginOrgUUID` |

**Security-tradeoff:** endpoint-managed **tamper-resistant на OS-level** — пользователь не может отредактировать файл без admin. Server-managed operates как client-side control — теоретически клиент можно подменить на unmanaged device.

---

## Platform-specific gotchas

### macOS
- Если директория `/Library/Application Support/ClaudeCode/` не существует — создать вручную (mkdir -p) перед первым deploy
- permissions должны быть `root:wheel 0755` (directory) + `root:wheel 0644` (files) — иначе user может не иметь read
- SIP (System Integrity Protection) не защищает эту папку — users с admin могут её менять (но обычные developers — нет)
- При деплое через Files and Processes (Jamf) — Jamf использует `sudo`, права правильные по умолчанию

### Windows
- REG_SZ имеет лимит 1 MB на value — если policy большая, дробить в drop-in fragments
- `HKLM\SOFTWARE\Policies\` лочится Group Policy — local admin не может modify без GPO
- WSL работает с Linux path (`/etc/claude-code/`), НЕ с Windows registry — для WSL нужен отдельный deploy
- **⚠️ Не использовать** `C:\ProgramData\ClaudeCode\` — deprecated с v2.1.75

### Linux
- Пакетных менеджеров для Claude Code **нет** (npm install -g, curl bash) — distribute через configuration management
- Контейнеры / CI/CD: mount `/etc/claude-code/` read-only из host, или bake в image
- NFS mounts — проверить что locking работает (некоторые NFS variants ломают CLI restart polling)

---

## Validation после deploy

Каждый deploy должен завершаться verify-шагом:

```bash
# 1. Validate JSON
jq empty /etc/claude-code/managed-settings.json

# 2. Check Claude Code picks it up
claude /status | grep -i "enterprise managed"
# Expected output contains: "Setting sources: ..., Enterprise managed policies, ..."

# 3. Test deny rule
claude -p "run curl google.com"
# Expected: blocked by policy (Bash(curl *) deny)

# 4. Test permission enforce
claude /permissions | grep "deny"
# Expected: deny rules listed with "managed" source
```

---

## Maintenance / monitoring

**Что мониторить:**
- File hash drift (Kandji / Addigy conditions)
- CLI version minimum (managed-settings.json `minimumVersion`) — чтобы user не мог downgrade до версии без managed support
- Compliance API logs (Enterprise plan) — видны попытки denied команд
- Hooks audit output (если настроены PostToolUse hooks в Splunk/Datadog)

**Red flags:**
- Users отчитываются что `/status` не показывает "Enterprise managed policies"
- `managed-settings.json` permissions изменились без MDM push (tamper)
- CLI fails to start после policy update (`forceRemoteSettingsRefresh: true` + api.anthropic.com unreachable)
- Developers обходят через `--setting-source local` (bug #11872 — это НЕ работает для managed, они всё равно применяются; но если на user/project level стоят conflicting rules — могут быть surprises)
