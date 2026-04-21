# IT Checklist — раскатывание Claude Code на 60+ разработчиков

> Используйте как "Day 0 deployment tracker" для вашей типовой fintech / enterprise команды.

---

## Phase 1 — Pre-deployment (недели -2 до 0)

### Business / compliance

- [ ] **License procurement** — Claude for Teams (до 150 seats) или Claude for Enterprise (150+ seats, SSO, compliance API)
- [ ] **Legal review** — MSA, DPA, BAA (если HIPAA), tenant restriction agreement
- [ ] **Compliance mapping** — какие SOC 2 / PCI / SOX / HIPAA controls затронуты
  - [ ] SOC 2 CC6.1 — logical access restrictions → `forceLoginMethod`
  - [ ] SOC 2 CC6.8 — unauthorized removal / modification → managed-settings.json deploy через MDM
  - [ ] SOC 2 CC7.2 — audit logging → Compliance API + hooks
  - [ ] PCI DSS 7.1.2 — need-to-know basis → `permissions.deny` на PCI scope
  - [ ] PCI DSS 10.2 — audit all access → hooks → SIEM
- [ ] **Budget approval** — seat cost + implementation hours
- [ ] **Approval from CISO / Security Officer**

### Technical prep

- [ ] **Identify MDM platform:**
  - [ ] Mac: Jamf Pro / Kandji / Addigy / Intune for Mac
  - [ ] Windows: Group Policy / Microsoft Intune
  - [ ] Linux: Ansible / Puppet / Chef / Salt
- [ ] **Pick delivery mechanism:**
  - [ ] Server-managed (Teams+/Enterprise, нет MDM) — простейший
  - [ ] MDM/OS-level policies — tamper-resistant, для managed fleet
  - [ ] File-based через config-management — для Linux / mixed env
- [ ] **Set up policy git repo** (internal) — version control всех managed files
- [ ] **Design CI/CD для policy deploy** (например, GitHub Actions → Jamf API / Intune API)
- [ ] **Provision SIEM endpoint** (Splunk HEC / Datadog Logs / ELK)
- [ ] **Create audit hook scripts** (session-start, pre-tool, post-tool, session-end)

### Org setup

- [ ] **Create Acme Claude org** в claude.ai (Primary Owner роль)
- [ ] **Enable SSO** (SAML / OAuth) через Okta / Entra / Google Workspace
- [ ] **Configure SCIM** — автоматическая provisioning пользователей
- [ ] **Set up RBAC roles** (Primary Owner, Owner, Admin, Member)
- [ ] **Enable Compliance API** (Enterprise plan only) — Org settings → Data and privacy → enable → create key
- [ ] **Configure tenant restrictions** (если нужен no-BYOK): `forceLoginMethod: claudeai` + `forceLoginOrgUUID`
- [ ] **Запомнить ORG UUID** — для managed-settings.json

---

## Phase 2 — Policy authoring (неделя -1)

- [ ] **Draft `managed-settings.json`** — начать с template из `section-7-template/`
- [ ] **Validate JSON:** `jq empty managed-settings.json`
- [ ] **Validate schema:** добавить `"$schema": "https://json.schemastore.org/claude-code-settings.json"` в editor (VS Code auto-validate)
- [ ] **Customize для вашей org:**
  - [ ] Заменить `acme-corp.com` на ваш домен
  - [ ] Заменить `REPLACE-WITH-ACME-ORG-UUID` на настоящий UUID
  - [ ] Дополнить `permissions.deny` вашими паттернами (test env, prod secrets, etc.)
  - [ ] Настроить `network.allowedDomains` — только нужные domains
  - [ ] Настроить `allowedMcpServers` — только approved servers
  - [ ] Указать SIEM endpoint в `env`
  - [ ] Изменить `minimumVersion` на current stable
- [ ] **Draft `managed-mcp.json`** (если есть internal MCP servers)
- [ ] **Draft managed `CLAUDE.md`** — company context, restricted areas, escalation
- [ ] **Draft audit hook scripts:**
  - [ ] `/usr/local/bin/acme-claude-session-start.sh` — log session start + verify user training
  - [ ] `/usr/local/bin/acme-claude-audit-pre.sh` — forward every tool use to SIEM (pre-approval)
  - [ ] `/usr/local/bin/acme-claude-audit-post.sh` — forward tool results to SIEM
  - [ ] `/usr/local/bin/acme-claude-session-end.sh` — log session end + metrics
- [ ] **Deploy hook scripts** через MDM (они лежат в system path, chmod 755, root:root)
- [ ] **Security code review policy** — policy files go through security team PR
- [ ] **Rollback plan** — hold previous policy version, restore procedure documented

---

## Phase 3 — Pilot (неделя 1)

- [ ] **Pilot group: 5-10 senior devs** (representative across backend / frontend / platform / security)
- [ ] **Deploy managed policy** to pilot group:
  - [ ] macOS: Jamf / Kandji configuration profile (scope = pilot Smart Group)
  - [ ] Windows: Intune / GPO policy (scope = pilot security group)
  - [ ] Linux: Ansible playbook с pilot inventory
- [ ] **Install Claude Code CLI** для pilot:
  - [ ] Official installer через корп software center / MDM self-service
  - [ ] Minimum version ≥ `minimumVersion` set in managed policy
- [ ] **Onboarding session** (30 min webinar / in-person):
  - [ ] Quick overview Claude Code + managed policy
  - [ ] Walk through CLAUDE.md expectations
  - [ ] Demo: `/permissions`, `/status`, `/config`
  - [ ] Q&A
- [ ] **Collect feedback** (daily standup / Slack):
  - [ ] Какие allow rules не хватает (ложное срабатывание deny)?
  - [ ] Какие deny rules ломают workflow?
  - [ ] Performance issues? (policy parse latency)
  - [ ] Audit hooks — работают ли правильно?
- [ ] **Iterate policy** — PR в git repo → merge → redeploy
- [ ] **Verify audit trail:**
  - [ ] Compliance API (если Enterprise) exports events
  - [ ] SIEM shows tool use events
  - [ ] Hook scripts logs are readable

---

## Phase 4 — Full rollout (неделя 2-3)

- [ ] **Expand scope** в MDM до всех разработчиков
- [ ] **Onboarding assignments** — каждый dev проходит 30-min training module перед получением CLI access
- [ ] **Enable `forceRemoteSettingsRefresh: true`** (если используете server-managed + fail-closed)
- [ ] **Monitor adoption:**
  - [ ] Compliance API analytics — сколько активных sessions
  - [ ] Usage analytics — какие модели используются
  - [ ] Error rate — сколько denied commands, по каким паттернам
- [ ] **Slack support channel** (`#claude-code-support`) с on-duty SecOps engineer первые 2 недели
- [ ] **Office hours** (1x/week) первый месяц для Q&A

---

## Phase 5 — Steady state (неделя 4+)

### Weekly operations

- [ ] **Weekly audit log review** (security team):
  - [ ] Denied attempts per developer
  - [ ] Unusual patterns (many denies → tool misuse or policy too strict?)
  - [ ] Model usage breakdown
  - [ ] Cost tracking
- [ ] **Weekly Compliance API export** → S3 / long-term archive (7 years retention)

### Monthly

- [ ] **Policy review meeting** — Security + Platform + Engineering представители
  - [ ] Review false-positive denies
  - [ ] Review false-negative (что должно было блокироваться, но не)
  - [ ] New MCP requests
  - [ ] CLI version upgrade plan (`minimumVersion` bump)
- [ ] **Update `companyAnnouncements`** с новыми guidelines / incidents learnings

### Quarterly

- [ ] **Full compliance review** (SecOps + Compliance Officer)
  - [ ] SOC 2 / PCI / SOX control mapping re-verify
  - [ ] Audit log retention check
  - [ ] Access review — все ли current devs have legitimate access
- [ ] **Dependency upgrade** — Claude Code CLI → latest stable
- [ ] **Policy version bump** в git repo + changelog entry

### Annually

- [ ] **Full security audit** by external auditor
- [ ] **HIPAA BAA re-sign** (если applicable)
- [ ] **DPA renewal** with Anthropic
- [ ] **Tabletop exercise** — what if managed policy bypass incident? Incident response runbook test

---

## Monitoring / alerting dashboard

**Must-have metrics:**
- Number of active Claude Code users (daily / weekly / monthly)
- Denied commands per user (spike indicates policy drift or attack)
- Model distribution (sonnet / haiku / opus usage)
- Session duration / volume
- Tool invocation frequency (Bash / Edit / Write / WebFetch / MCP)
- Hook failure rate (< 1% acceptable)
- Policy sync latency (from git commit → MDM deploy → client apply)

**Alerts:**
- [ ] `managed-settings.json` tampering detected (file hash drift) → PagerDuty
- [ ] User attempted `--dangerously-skip-permissions` (should be blocked but logged) → Slack SecOps
- [ ] User logged in with personal account (should fail but logged) → Slack SecOps
- [ ] Unusual deny rate (>10/day for single user) → Slack SecOps
- [ ] Policy sync failure (>24h since last success) → Slack Platform team

---

## Incident response checklist

### "Claude Code wrote secrets to a commit" incident

1. [ ] Confirm via audit log (Compliance API / SIEM)
2. [ ] Rotate exposed credentials immediately
3. [ ] Force-push clean history (with approval from Security)
4. [ ] Scan public repos (if pushed to public) — GitHub Secret Scanning API
5. [ ] Notify stakeholders + customers (если data breach scope)
6. [ ] Post-mortem: why policy didn't block? → adjust `permissions.deny`

### "Developer bypassed policy" incident

1. [ ] Determine: was bypass successful? (check Compliance API)
2. [ ] If yes → verify policy deployment on dev's machine
3. [ ] If policy was missing → re-deploy via MDM, root cause why MDM-sync failed
4. [ ] If policy was tampered with → terminate access, investigate intent
5. [ ] Update MDM compliance check to auto-detect file hash drift

### "Managed policy fail-closed breaking productivity" incident

1. [ ] If `forceRemoteSettingsRefresh: true` + api.anthropic.com unreachable → CLI won't start
2. [ ] Temporary workaround: set `forceRemoteSettingsRefresh: false` (via server-managed hot update)
3. [ ] Root cause network issue (corp firewall, DNS, etc.)
4. [ ] Post-mortem: adjust monitoring for api.anthropic.com reachability

---

## Deliverables artifact-list (что должно быть в git repo)

```
acme-claude-code-policy/
├── README.md                    ← эта шпаргалка в виде README
├── CHANGELOG.md                  ← policy version history
├── managed/
│   ├── macOS/
│   │   ├── com.anthropic.claudecode.mobileconfig     ← Jamf profile
│   │   └── managed-settings.json                      ← file-based fallback
│   ├── windows/
│   │   ├── claude-code-policy.admx                   ← GPO template
│   │   └── managed-settings.json                      ← file-based fallback
│   └── linux/
│       └── managed-settings.json
├── managed-CLAUDE.md             ← behavioral layer
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
│   ├── validate-policy.sh        ← JSON schema + business rules
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

- [ ] Can user run Claude Code? `claude --version` returns ≥ minimumVersion
- [ ] Does `/status` show "Enterprise managed policies"?
- [ ] Does `/permissions` show managed rules with source "managed"?
- [ ] Does deny rule actually block? `claude -p "curl google.com"` → expected rejected
- [ ] Does allow rule actually pass? `claude -p "run tests"` → expected accepted
- [ ] Does `--dangerously-skip-permissions` fail? `claude --dangerously-skip-permissions "x"` → error
- [ ] Does `--setting-source local` still show managed? → yes (by design)
- [ ] Does SIEM receive audit events? Check last 10 min of Splunk
- [ ] Does Compliance API return events? Recent activity visible
- [ ] Can user override model? `claude --model opus` → rejected (if opus not in availableModels)
- [ ] Does managed CLAUDE.md load? Check `/status` → shows managed memory file

---

## Communication templates

### Email to devs — "Claude Code policy rollout"

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

### Slack announcement — "New policy version"

```
📣 Claude Code policy v2.1 rolled out

What changed:
- Added allowlist for `npm audit fix` (was previously blocked)
- Blocked reading `~/.terraformrc` (contains cloud creds)
- Added `company-docs-mcp` to allowed MCP servers

Expected rollout: next 24h via MDM. Restart Claude Code after MDM sync.
Questions: #claude-code-support. Full changelog: [git repo link].
```
