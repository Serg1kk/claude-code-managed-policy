# IT Checklist — rolling out Claude Code to 60+ developers

> **Language:** English · [Русский](it-checklist.ru.md)

> Use as a "Day 0 deployment tracker" for your typical fintech / enterprise team.

---

## Phase 1 — Pre-deployment (weeks -2 to 0)

### Business / compliance

- [ ] **License procurement**: Claude for Teams (up to 150 seats) or Claude for Enterprise (150+ seats, SSO, compliance API)
- [ ] **Legal review**: MSA, DPA, BAA (if HIPAA applies), tenant restriction agreement
- [ ] **Compliance mapping**: which SOC 2 / PCI / SOX / HIPAA controls are in scope
  - [ ] SOC 2 CC6.1: logical access restrictions → `forceLoginMethod`
  - [ ] SOC 2 CC6.8: unauthorized removal / modification → managed-settings.json deployed via MDM
  - [ ] SOC 2 CC7.2: audit logging → Compliance API + hooks
  - [ ] PCI DSS 7.1.2: need-to-know basis → `permissions.deny` on PCI scope
  - [ ] PCI DSS 10.2: audit all access → hooks → SIEM
- [ ] **Budget approval**: seat cost + implementation hours
- [ ] **Sign-off from CISO / Security Officer**

### Technical prep

- [ ] **Identify MDM platform:**
  - [ ] Mac: Jamf Pro / Kandji / Addigy / Intune for Mac
  - [ ] Windows: Group Policy / Microsoft Intune
  - [ ] Linux: Ansible / Puppet / Chef / Salt
- [ ] **Pick delivery mechanism:**
  - [ ] Server-managed (Teams+/Enterprise, no MDM): the simplest option
  - [ ] MDM/OS-level policies: tamper-resistant, for a managed fleet
  - [ ] File-based via config management: for Linux / mixed environments
- [ ] **Set up a policy git repo** (internal): version control for all managed files
- [ ] **Design CI/CD for policy deploys** (e.g., GitHub Actions → Jamf API / Intune API)
- [ ] **Provision a SIEM endpoint** (Splunk HEC / Datadog Logs / ELK)
- [ ] **Create audit hook scripts** (session-start, pre-tool, post-tool, session-end)

### Org setup

- [ ] **Create the Acme Claude org** in claude.ai (Primary Owner role)
- [ ] **Enable SSO** (SAML / OAuth) via Okta / Entra / Google Workspace
- [ ] **Configure SCIM**: automated user provisioning
- [ ] **Set up RBAC roles** (Primary Owner, Owner, Admin, Member)
- [ ] **Enable the Compliance API** (Enterprise plan only): Org settings → Data and privacy → enable → create key
- [ ] **Configure tenant restrictions** (if you need no-BYOK): `forceLoginMethod: claudeai` + `forceLoginOrgUUID`
- [ ] **Record the ORG UUID**: for managed-settings.json

---

## Phase 2 — Policy authoring (week -1)

- [ ] **Draft `managed-settings.json`**: start from the template in `section-7-template/`
- [ ] **Validate JSON:** `jq empty managed-settings.json`
- [ ] **Validate schema:** add `"$schema": "https://json.schemastore.org/claude-code-settings.json"` in your editor (VS Code auto-validates)
- [ ] **Customize for your org:**
  - [ ] Replace `acme-corp.com` with your domain
  - [ ] Replace `REPLACE-WITH-ACME-ORG-UUID` with the real UUID
  - [ ] Extend `permissions.deny` with your own patterns (test env, prod secrets, etc.)
  - [ ] Configure `network.allowedDomains`: only the domains you actually need
  - [ ] Configure `allowedMcpServers`: only approved servers
  - [ ] Point the SIEM endpoint via `env`
  - [ ] Bump `minimumVersion` to the current stable release
- [ ] **Draft `managed-mcp.json`** (if you have internal MCP servers)
- [ ] **Draft a managed `CLAUDE.md`**: company context, restricted areas, escalation paths
- [ ] **Draft audit hook scripts:**
  - [ ] `/usr/local/bin/acme-claude-session-start.sh`: log session start + verify user training
  - [ ] `/usr/local/bin/acme-claude-audit-pre.sh`: forward every tool use to SIEM (pre-approval)
  - [ ] `/usr/local/bin/acme-claude-audit-post.sh`: forward tool results to SIEM
  - [ ] `/usr/local/bin/acme-claude-session-end.sh`: log session end + metrics
- [ ] **Deploy hook scripts** via MDM (placed in a system path, chmod 755, root:root)
- [ ] **Security code review for policy**: policy files go through a security-team PR
- [ ] **Rollback plan**: keep the previous policy version on hand, document the restore procedure

---

## Phase 3 — Pilot (week 1)

- [ ] **Pilot group: 5-10 senior devs** (representative mix across backend / frontend / platform / security)
- [ ] **Deploy managed policy** to the pilot group:
  - [ ] macOS: Jamf / Kandji configuration profile (scope = pilot Smart Group)
  - [ ] Windows: Intune / GPO policy (scope = pilot security group)
  - [ ] Linux: Ansible playbook with pilot inventory
- [ ] **Install the Claude Code CLI** for the pilot:
  - [ ] Official installer via the corporate software center / MDM self-service
  - [ ] Minimum version ≥ `minimumVersion` set in the managed policy
- [ ] **Onboarding session** (30-min webinar / in-person):
  - [ ] Quick overview of Claude Code + managed policy
  - [ ] Walk through the CLAUDE.md expectations
  - [ ] Demo: `/permissions`, `/status`, `/config`
  - [ ] Q&A
- [ ] **Collect feedback** (daily standup / Slack):
  - [ ] Which allow rules are missing (deny false positives)?
  - [ ] Which deny rules break workflows?
  - [ ] Performance issues? (policy parse latency)
  - [ ] Audit hooks: are they firing correctly?
- [ ] **Iterate the policy**: PR into the git repo → merge → redeploy
- [ ] **Verify the audit trail:**
  - [ ] Compliance API (if on Enterprise) exports events
  - [ ] SIEM shows tool use events
  - [ ] Hook script logs are readable

---

## Phase 4 — Full rollout (weeks 2-3)

- [ ] **Expand scope** in MDM to all developers
- [ ] **Onboarding assignments**: every dev completes the 30-min training module before being granted CLI access
- [ ] **Enable `forceRemoteSettingsRefresh: true`** (if you're using server-managed + fail-closed)
- [ ] **Monitor adoption:**
  - [ ] Compliance API analytics: how many active sessions
  - [ ] Usage analytics: which models are being used
  - [ ] Error rate: how many denied commands, and which patterns trigger them
- [ ] **Slack support channel** (`#claude-code-support`) with an on-duty SecOps engineer for the first 2 weeks
- [ ] **Office hours** (1x/week) for the first month for Q&A

---

## Phase 5 — Steady state (week 4+)

### Weekly operations

- [ ] **Weekly audit log review** (security team):
  - [ ] Denied attempts per developer
  - [ ] Unusual patterns (many denies → tool misuse or policy too strict?)
  - [ ] Model usage breakdown
  - [ ] Cost tracking
- [ ] **Weekly Compliance API export** → S3 / long-term archive (7-year retention)

### Monthly

- [ ] **Policy review meeting**: Security + Platform + Engineering representatives
  - [ ] Review false-positive denies
  - [ ] Review false negatives (what should have been blocked but wasn't)
  - [ ] New MCP requests
  - [ ] CLI version upgrade plan (`minimumVersion` bump)
- [ ] **Update `companyAnnouncements`** with new guidelines / lessons learned from incidents

### Quarterly

- [ ] **Full compliance review** (SecOps + Compliance Officer)
  - [ ] Re-verify SOC 2 / PCI / SOX control mapping
  - [ ] Check audit log retention
  - [ ] Access review: do all current devs have legitimate access?
- [ ] **Dependency upgrade**: Claude Code CLI → latest stable
- [ ] **Policy version bump** in the git repo + changelog entry

### Annually

- [ ] **Full security audit** by an external auditor
- [ ] **HIPAA BAA re-sign** (if applicable)
- [ ] **DPA renewal** with Anthropic
- [ ] **Tabletop exercise**: what if there's a managed-policy-bypass incident? Test the incident response runbook

---

## Monitoring / alerting dashboard

**Must-have metrics:**
- Number of active Claude Code users (daily / weekly / monthly)
- Denied commands per user (a spike indicates policy drift or an attack)
- Model distribution (sonnet / haiku / opus usage)
- Session duration / volume
- Tool invocation frequency (Bash / Edit / Write / WebFetch / MCP)
- Hook failure rate (< 1% acceptable)
- Policy sync latency (from git commit → MDM deploy → client apply)

**Alerts:**
- [ ] `managed-settings.json` tampering detected (file hash drift) → PagerDuty
- [ ] User attempted `--dangerously-skip-permissions` (should be blocked but logged) → Slack SecOps
- [ ] User logged in with a personal account (should fail but logged) → Slack SecOps
- [ ] Unusual deny rate (>10/day for a single user) → Slack SecOps
- [ ] Policy sync failure (>24h since last success) → Slack Platform team

---

## Incident response checklist

### "Claude Code wrote secrets to a commit" incident

1. [ ] Confirm via audit log (Compliance API / SIEM)
2. [ ] Rotate exposed credentials immediately
3. [ ] Force-push clean history (with Security sign-off)
4. [ ] Scan public repos (if pushed to public) via the GitHub Secret Scanning API
5. [ ] Notify stakeholders + customers (if the scope qualifies as a data breach)
6. [ ] Post-mortem: why didn't the policy block this? → adjust `permissions.deny`

### "Developer bypassed policy" incident

1. [ ] Determine whether the bypass succeeded (check the Compliance API)
2. [ ] If yes → verify policy deployment on the dev's machine
3. [ ] If the policy was missing → re-deploy via MDM, root-cause why the MDM sync failed
4. [ ] If the policy was tampered with → revoke access, investigate intent
5. [ ] Update MDM compliance checks to auto-detect file hash drift

### "Managed policy fail-closed breaking productivity" incident

1. [ ] If `forceRemoteSettingsRefresh: true` and api.anthropic.com is unreachable → the CLI won't start
2. [ ] Temporary workaround: set `forceRemoteSettingsRefresh: false` (via a server-managed hot update)
3. [ ] Root-cause the network issue (corp firewall, DNS, etc.)
4. [ ] Post-mortem: tighten monitoring for api.anthropic.com reachability

---

## Deliverables artifact list (what should live in the git repo)

```
acme-claude-code-policy/
├── README.md                    ← this cheat sheet, as a README
├── CHANGELOG.md                 ← policy version history
├── managed/
│   ├── macOS/
│   │   ├── com.anthropic.claudecode.mobileconfig     ← Jamf profile
│   │   └── managed-settings.json                      ← file-based fallback
│   ├── windows/
│   │   ├── claude-code-policy.admx                   ← GPO template
│   │   └── managed-settings.json                      ← file-based fallback
│   └── linux/
│       └── managed-settings.json
├── managed-CLAUDE.md            ← behavioral layer
├── managed-settings.d/
│   ├── 10-security.json
│   ├── 20-models.json
│   ├── 30-mcp.json
│   └── 40-audit.json
├── hooks/
│   ├── acme-claude-session-start.sh
│   ├── acme-claude-audit-pre.sh
│   ├── acme-claude-audit-post.sh
│   └── acme-claude-session-end.sh
├── ansible/
│   ├── playbook.yml
│   ├── roles/claude-code-policy/
│   └── inventories/{dev,pilot,prod}.ini
├── ci/
│   ├── validate-policy.sh       ← JSON schema + business rules
│   ├── deploy-jamf.sh
│   ├── deploy-intune.sh
│   └── deploy-ansible.sh
└── docs/
    ├── onboarding.md
    ├── runbook-incidents.md
    └── changelog-templates/
```

---

## Final sanity checks pre-rollout

- [ ] Can the user run Claude Code? `claude --version` returns ≥ minimumVersion
- [ ] Does `/status` show "Enterprise managed policies"?
- [ ] Does `/permissions` show managed rules with source "managed"?
- [ ] Does a deny rule actually block? `claude -p "curl google.com"` → expected to be rejected
- [ ] Does an allow rule actually pass? `claude -p "run tests"` → expected to be accepted
- [ ] Does `--dangerously-skip-permissions` fail? `claude --dangerously-skip-permissions "x"` → error
- [ ] Does `--setting-source local` still show managed rules? → yes (by design)
- [ ] Does SIEM receive audit events? Check the last 10 minutes in Splunk
- [ ] Does the Compliance API return events? Recent activity visible
- [ ] Can a user override the model? `claude --model opus` → rejected (if opus isn't in availableModels)
- [ ] Does the managed CLAUDE.md load? Check `/status` → shows the managed memory file

---

## Communication templates

### Email to devs: "Claude Code policy rollout"

```
Subject: Claude Code is now enabled company-wide (managed policy)

Hi team,

Claude Code is now officially supported at Acme Corp. IT has deployed a managed policy that enables secure AI-assisted development.

What this means:
- Claude Code is installed on your workstation (auto-deployed via Jamf/Intune)
- Company standards are applied automatically (you'll see a welcome message)
- Some commands are blocked for security (reading .env, committing secrets, etc.)
- All activity is logged per our SOC 2 requirements

What to do:
1. Watch the 20-min onboarding video: [link]
2. Try Claude Code: `claude` in terminal → review managed CLAUDE.md context
3. Ask questions in #claude-code-support

Your AI assistant is now compliance-ready. 🤖

— IT Security
```

### Slack announcement: "New policy version"

```
📣 Claude Code policy v2.1 rolled out

What changed:
- Added allowlist for `npm audit fix` (was previously blocked)
- Blocked reading `~/.terraformrc` (contains cloud creds)
- Added `company-docs-mcp` to allowed MCP servers

Expected rollout: next 24h via MDM. Restart Claude Code after MDM sync.
Questions: #claude-code-support. Full changelog: [git repo link].
```
