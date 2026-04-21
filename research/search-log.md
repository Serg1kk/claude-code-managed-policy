# Search log —  research

**Date:** 2026-04-21
**Mode:** course
**Sources run:** 1 (KB grep), 3 (raw transcripts), 6 (Exa web), existing course research baseline
**Sources skipped:** 2 (raw videos list — no matches), 4 (no additional inputs), 5 (NotebookLM — off, Anthropic docs are primary source-of-truth)

---

## Searches executed

### Source 1 — Knowledge Base (internal)

```
Grep: "managed.policy|Managed Policy|MANAGED_POLICY" in AI stuff/knowledge-base/
  → 0 files

Grep: "/etc/claude-code|Library/Application Support/ClaudeCode" in AI stuff/
  → 0 files

Grep: "enterprise|managedSettings|claudeMdExcludes|Jamf|Intune|MDM" in AI stuff/
  → 30+ files (generic mentions, no concrete managed-policy content)

Grep: "managed.policy|Managed Policy|enterprise.policy" in AI stuff/youtube/processed/
  → 0 files
```

**Conclusion:** Community YouTube analyses and KB content do not materially cover managed policy. Research relies 100% on external sources (Anthropic docs + community guides).

### Source 3 — Raw YouTube transcripts

```
Grep: "CLAUDE\.md|managed|enterprise" in 2026-04-12_xCd9ykretlg (Keith Rabois podcast)
  → 8 hits, но это generic (не про managed policy)

Спот-чек других raw transcripts — Nicolas Puru, Simon Scrapes, Cole Medin
(ранее processed video-analyses) — никто не покрывает managed policy детально.
```

**Вывод:** community YouTube не покрывает managed policy — это enterprise/IT-admin тема, а YouTubers ориентированы на indie developers.

### Source 6 — Exa web searches

1. **Query:** `"Anthropic Claude Code enterprise managed policy settings documentation docs.anthropic.com"`
   **Top hits:** docs.claude.com/en/docs/claude-code/settings, server-managed-settings, permissions, third-party-integrations, claude.com/product/claude-code/enterprise
   **Value:** ⭐⭐⭐ канонические official docs

2. **Query:** `"Claude Code managed settings path macOS Library Application Support ClaudeCode /etc/claude-code Windows ProgramData"`
   **Top hits:** claudelog.com, claudefa.st (settings reference v2.1.79 / v2.1.114), managed-settings.net
   **Value:** ⭐⭐⭐ verified paths + deprecation note (C:\ProgramData\ → C:\Program Files\)

3. **Query:** `"Claude Code MDM Jamf Intune Configuration Profile deploy enterprise policy PowerShell"`
   **Top hits:** Addigy blog (Apple-first), systemprompt.io rollout playbooks, Anthropic French/Chinese help-center (Claude Desktop enterprise config)
   **Value:** ⭐⭐⭐ practical deployment scripts + phased rollout

4. **Query:** `"Claude Code managed policy CLAUDE.md enterprise IT admin cannot be excluded BYOK"`
   **Top hits:** GitHub issue #34349 (claudeMdRequires feature request), truefoundry.com blog, GitHub issue #11872 (--setting-source local bypass attempt)
   **Value:** ⭐⭐⭐ critical gaps: claudeMdRequires not implemented, --setting-source не отключает managed

5. **Query:** `"Cursor enterprise team admin policies managed rules vs GitHub Copilot Business Enterprise organization policies"`
   **Top hits:** docs.github.com Copilot enterprise policies, cursor.com enterprise docs, aicodereview.cc comparison
   **Value:** ⭐⭐⭐ competitor comparison table

6. **Query:** `"Windsurf Codeium enterprise admin team policy deployment MDM"`
   **Top hits:** docs.windsurf.com/enterprise-policies, codeium.com/security (deployment tiers), docs.codeium.com guide-for-admins
   **Value:** ⭐⭐ Windsurf Enterprise policies details (Group Policy ADMX + FedRAMP High + Enterprise Hybrid/Self-hosted)

7. **Query:** `"OpenAI Codex enterprise org admin controls policy Codex Enterprise edition"`
   **Top hits:** developers.openai.com/codex/enterprise, developers.openai.com/codex/enterprise/managed-configuration, /codex/exec-policy (rules), help.openai.com codex admin guide
   **Value:** ⭐⭐⭐ Codex requirements.toml mechanism — очень близко к Claude managed-settings.json

8. **Query:** `"Claude Code claudeMdExcludes cannot exclude managed policy CLAUDE.md system directory enterprise"`
   **Top hits:** docs.anthropic.com/memory (official wording), GitHub issues #20880 + #34349 + #11872, mintlify settings reference
   **Value:** ⭐⭐⭐ verified official position on what cannot be excluded

9. **Query:** `"Anthropic Claude Code Ansible Puppet Chef playbook managed-settings.json deployment Linux"`
   **Top hits:** kodekloud.com (Claude Code + Ansible demo), systemprompt.io, Learn Linux TV YouTube
   **Value:** ⭐⭐ practical Ansible/Puppet/Chef templates

10. **Query:** `"Claude Code managed policy HIPAA SOC2 compliance audit logging fintech enterprise deployment"`
    **Top hits:** amitkoth.com SOC 2 guide, claude-ai.chat enterprise environments guide, Anthropic Compliance API help-center article
    **Value:** ⭐⭐⭐ compliance-specific use-cases + auditor perspective

### Source 6 — Exa web fetches

Fetched full content:
- https://docs.claude.com/en/docs/claude-code/settings — complete settings reference (all keys, precedence, merge behavior)
- https://code.claude.com/docs/en/server-managed-settings.md — server-managed mechanism
- https://code.claude.com/docs/en/permissions.md — managed-only permissions keys
- https://managed-settings.net/ — community-built reference + precedence diagram

---

## Key discoveries

1. **Two files, two purposes** — `managed-settings.json` (policy enforcement, primary) vs. Managed `CLAUDE.md` (behavioral memory, secondary). Current course materials conflate the two — this is the main correction.

2. **Server-managed settings (Q4 2025+)** — third delivery mechanism beyond file-based and MDM. Not in course materials at all.

3. **Drop-in directory `managed-settings.d/`** (v2.1.83) — modular policy composition, systemd-style merge. Not in course materials.

4. **Windows path deprecation v2.1.75** — `C:\ProgramData\ClaudeCode\` removed in favor of `C:\Program Files\ClaudeCode\`. Course materials already use correct path ✅.

5. **claudeMdRequires** feature request still open — `claudeMdExcludes` can exclude project-level rule files, enterprise forced into monolithic managed CLAUDE.md.

6. **--setting-source local does NOT disable managed policies** — Anthropic official position (bug #11872 closed as "by design").

7. **Five distinct managed-only keys tiers:** `allowManagedXxxOnly` family (`PermissionRulesOnly`, `McpServersOnly`, `HooksOnly`, `DomainsOnly`, `ReadPathsOnly`) — combine for strict lockdown.

8. **MDM templates** officially published for 4 platforms: Jamf, Kandji (Iru), Intune, Group Policy — at github.com/anthropics/claude-code/tree/main/examples/mdm.

---

## Sources not fully explored (future research hooks)

- Anthropic trust.anthropic.com portal — SOC 2 certificate / report (need login)
- Compliance API PDF (linked from help center) — detailed event types
- Specific Jamf ProfileCreator templates from Anthropic examples/mdm — not fetched content, only acknowledged presence
- Live tests of:
  - `CLAUDE_MANAGED_SETTINGS_PATH` env var (unconfirmed officially)
  - `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var (unconfirmed, community-mentioned)
  - Behavior when managed policy file exists but not readable (graceful degradation confirmed theoretically, not tested)

---

## Time spent

- Phase 0 (mode detection, source triage): 3 min
- Phase 1 (reading promise README, existing baseline): 5 min
- Phase 2 (Exa searches + Grep + fetches): 15 min
- Phase 6 (authoring README + 7 section files): 45 min

Total: ~70 min
