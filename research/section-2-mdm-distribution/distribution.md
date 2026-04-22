# Section 2 — MDM distribution mechanics

> **Language:** English · [Русский](distribution.ru.md)

**Sources:**
- https://docs.claude.com/en/docs/claude-code/settings#managed-settings
- https://code.claude.com/docs/en/server-managed-settings.md
- https://github.com/anthropics/claude-code/tree/main/examples/mdm (official starter templates)
- https://addigy.com/blog/manage-claude-code-policies-addigy/ (Apple-first MDM)
- https://systemprompt.io/guides/enterprise-claude-code-managed-settings
- https://systemprompt.io/guides/claude-code-organisation-rollout

---

## Three delivery channels (per official docs)

### 1. Server-managed settings (NEW, Q4 2025)

**What it is:** an admin goes to Claude.ai → Admin Settings → Claude Code → Managed settings, writes JSON, and the client picks it up automatically on login or via hourly polling.

**Requirements:**
- Claude for Teams plan (CLI v2.1.38+) or Claude for Enterprise plan (v2.1.30+)
- Client has network access to `api.anthropic.com`
- **Does NOT require** MDM, device management infrastructure, or admin access to the OS

**Pros:**
- Works on BYOD / unmanaged devices (where IT does not deploy Jamf)
- Changes deploy instantly (on restart / hourly poll)
- No delays waiting for an MDM push

**Cons / limitations:**
- Only uniform per org (no per-group policies: still a limitation in v2.1.x)
- Requires a Claude for Teams / Enterprise plan (not available on Pro individual subscriptions)
- Settings are fully **unavailable offline**: a live connection to api.anthropic.com is required at fetch time
- If you rely on `forceRemoteSettingsRefresh: true`, the CLI is **fail-closed** on offline startup
- **Cannot be used** when working through third-party providers (Bedrock / Vertex / Foundry): direct Anthropic API only

**Configure flow:**
```
1. Organization Owner / Primary Owner → claude.ai → Admin Settings → Claude Code → Managed settings
2. Paste JSON (same schema as managed-settings.json)
3. Save → rollout ~1 hour (next user startup or polling cycle)
```

Example (from official docs):
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

#### macOS via Jamf Pro / Kandji / Intune for Mac / Addigy

**Mechanism:** a Configuration Profile is deployed to the managed preferences domain `com.anthropic.claudecode`. The plist top-level keys map 1:1 to managed-settings.json.

**Jamf Pro step by step:**
```
Jamf Pro Console → Computers → Configuration Profiles → + New
  → Application & Custom Settings payload
  → Preference Domain: com.anthropic.claudecode
  → Upload plist file (or use the Jamf Pro profile editor)
  → Scope: Developer Smart Group (not all-fleet)
  → Deploy
```

Alternatively, via the **Files and Processes** payload: drop `managed-settings.json` directly into `/Library/Application Support/ClaudeCode/`.

**Kandji step by step:**
```
Kandji → Library → Custom Profile → Upload .mobileconfig
  or
Library → Custom Script (Audit & Enforce):
  - Script checks file hash
  - Replaces if drift detected
```

**Addigy step by step** (from https://addigy.com/blog/manage-claude-code-policies-addigy/):
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

**Intune for Mac:** Configuration profile → Custom Template → upload `.mobileconfig` targeting the `com.anthropic.claudecode` preference domain.

#### Windows via Group Policy / Microsoft Intune

**Mechanism:** REG_SZ JSON in `HKLM\SOFTWARE\Policies\ClaudeCode\Settings` (machine-level) or `HKCU\SOFTWARE\Policies\ClaudeCode\Settings` (user-level fallback).

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

**Intune via Configuration Profile:**
```
Intune → Devices → Configuration Profiles → + Create → Windows 10+
  → Templates → Administrative Templates → none (no Claude Code ADMX)
  → Settings catalog → Registry-based custom:
       OMA-URI: ./Device/Vendor/MSFT/Registry/HKLM/SOFTWARE/Policies/ClaudeCode/Settings
       Data type: String
       Value: <JSON>
```

Or via a **Win32 app** packaging `managed-settings.json` together with a PowerShell install script.

**PowerShell deployment (baseline):**
```powershell
# On target machine as admin
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

#### Shell script (simplest option)

```bash
#!/bin/bash
# deploy-claude-policy.sh — for small teams without configuration management
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

Includes starter templates for:
- **Jamf Pro**: `.mobileconfig` profile
- **Kandji (Iru)**: Custom Profile format
- **Microsoft Intune**: Configuration Profile JSON
- **Group Policy (GPO)**: ADMX/ADML templates

There are **no** official starters for Linux. Use the Ansible/Puppet/Chef examples above instead.

---

## Phased rollout playbook (60 fintech developers)

**From https://systemprompt.io/guides/claude-code-organisation-rollout:**

| Week | What to do | Audience |
|---|---|---|
| **Week 1** | Write draft policy, version it in git, get code-review from the security team | 0 devs (admin prep) |
| **Week 2** | Pilot group: 5-10 senior devs. Deploy via Jamf/Intune pilot scope. Gather feedback, tweak policy | 5-10 |
| **Week 3** | Expand pilot to 20 devs + mandatory onboarding session. Add Managed CLAUDE.md for company context | 20 |
| **Week 4** | Full rollout to the entire developer scope. Enable `forceRemoteSettingsRefresh` if you need fail-closed | 60 |
| **Week 5+** | Ongoing: policy changes via PRs in the git repo, rollout via MDM, audit log review weekly | all |

**Rollback strategy:**
1. Keep prior policy versions in git (e.g., `managed-settings-v1.2.json`)
2. On issues: push the previous version through MDM (Jamf/Intune revert)
3. For server-managed: revert via the Claude.ai admin console (history is kept)
4. Emergency kill-switch: deploy an empty `{}` as the effective minimum managed policy

---

## Distinguishing server-managed vs endpoint-managed: when to use each

| Scenario | Recommendation |
|---|---|
| BYOD / unmanaged laptops | Server-managed (no MDM required) |
| Enterprise Mac fleet with Jamf | Endpoint-managed via Jamf (tamper-resistant at the OS level) |
| Windows enterprise with SCCM/Intune | Endpoint-managed via Intune/GPO |
| Mixed Mac + Windows + Linux | Server-managed (unified, but requires Enterprise plan) |
| Air-gapped / offline environments | Endpoint-managed (server-managed requires api.anthropic.com) |
| Startup with no IT infrastructure | Server-managed (if on a Teams plan) or file-based via script |
| Contractors / vendors | Server-managed + `forceLoginOrgUUID` |

**Security tradeoff:** endpoint-managed is **tamper-resistant at the OS level**: users cannot edit the file without admin rights. Server-managed operates as a client-side control, so in theory the client could be swapped out on an unmanaged device.

---

## Platform-specific gotchas

### macOS
- If `/Library/Application Support/ClaudeCode/` does not exist, create it manually (mkdir -p) before the first deploy
- Permissions must be `root:wheel 0755` (directory) + `root:wheel 0644` (files), otherwise users may not have read access
- SIP (System Integrity Protection) does not protect this folder: users with admin can modify it (but regular developers cannot)
- When deploying through Files and Processes (Jamf), Jamf uses `sudo`, so permissions are correct by default

### Windows
- REG_SZ has a 1 MB per-value limit. If the policy is large, split it into drop-in fragments
- `HKLM\SOFTWARE\Policies\` is locked by Group Policy: local admins cannot modify it without GPO
- WSL works with Linux paths (`/etc/claude-code/`), NOT with the Windows registry. WSL needs a separate deploy
- **⚠️ Do not use** `C:\ProgramData\ClaudeCode\`: deprecated since v2.1.75

### Linux
- There are **no** package managers for Claude Code (npm install -g, curl bash). Distribute via configuration management
- Containers / CI/CD: mount `/etc/claude-code/` read-only from the host, or bake it into the image
- NFS mounts: verify that locking works (some NFS variants break CLI restart polling)

---

## Validation after deploy

Every deploy should end with a verify step:

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

**What to monitor:**
- File hash drift (Kandji / Addigy conditions)
- CLI minimum version (managed-settings.json `minimumVersion`): prevent users from downgrading to a version without managed support
- Compliance API logs (Enterprise plan): visible attempts at denied commands
- Hooks audit output (if PostToolUse hooks are wired up in Splunk/Datadog)

**Red flags:**
- Users report that `/status` does not show "Enterprise managed policies"
- `managed-settings.json` permissions changed without an MDM push (tamper)
- CLI fails to start after a policy update (`forceRemoteSettingsRefresh: true` + api.anthropic.com unreachable)
- Developers bypass via `--setting-source local` (bug #11872: this does NOT work for managed, which are still applied; but if conflicting rules exist at user/project level, surprises are possible)
