# Search log — research reproducibility

**Date:** 2026-04-21
**Mode:** full external + baseline
**Sources run:** existing research baseline, Exa web searches (10 queries), targeted Exa fetches
**Sources skipped:** NotebookLM (Anthropic docs are primary source of truth)

---

## Searches executed

### Internal baseline — quick triage

A private research baseline was grepped first to establish whether community transcripts and prior analyses materially covered Claude Code managed policy. Queries run against a processed YouTube transcript archive and a curated knowledge-base of AI-tooling notes:

- `managed.policy|Managed Policy|MANAGED_POLICY` → 0 hits
- `/etc/claude-code|Library/Application Support/ClaudeCode` → 0 hits
- `enterprise|managedSettings|claudeMdExcludes|Jamf|Intune|MDM` → 30+ files with generic mentions, none with concrete managed-policy content
- `managed.policy|Managed Policy|enterprise.policy` across processed YouTube analyses → 0 hits

**Conclusion:** community YouTube analyses and general AI-tooling write-ups do not materially cover Claude Code managed policy. This is an enterprise/IT-admin topic that indie-developer-oriented creators haven't covered in depth yet. The research below relies 100% on external sources (Anthropic official docs + community deployment guides).

### Exa web searches — 10 queries

1. **Query:** `"Anthropic Claude Code enterprise managed policy settings documentation docs.anthropic.com"`
   **Top hits:** docs.claude.com/en/docs/claude-code/settings, server-managed-settings, permissions, third-party-integrations, claude.com/product/claude-code/enterprise
   **Value:** ⭐⭐⭐ canonical official docs

2. **Query:** `"Claude Code managed settings path macOS Library Application Support ClaudeCode /etc/claude-code Windows ProgramData"`
   **Top hits:** claudelog.com, claudefa.st (settings reference v2.1.79 / v2.1.114), managed-settings.net
   **Value:** ⭐⭐⭐ verified paths + deprecation note (C:\ProgramData\ → C:\Program Files\)

3. **Query:** `"Claude Code MDM Jamf Intune Configuration Profile deploy enterprise policy PowerShell"`
   **Top hits:** Addigy blog (Apple-first), systemprompt.io rollout playbooks, Anthropic French/Chinese help-center (Claude Desktop enterprise config)
   **Value:** ⭐⭐⭐ practical deployment scripts + phased rollout

4. **Query:** `"Claude Code managed policy CLAUDE.md enterprise IT admin cannot be excluded BYOK"`
   **Top hits:** GitHub issue #34349 (claudeMdRequires feature request), truefoundry.com blog, GitHub issue #11872 (--setting-source local bypass attempt)
   **Value:** ⭐⭐⭐ critical gaps — claudeMdRequires not implemented, --setting-source does not disable managed

5. **Query:** `"Cursor enterprise team admin policies managed rules vs GitHub Copilot Business Enterprise organization policies"`
   **Top hits:** docs.github.com Copilot enterprise policies, cursor.com enterprise docs, aicodereview.cc comparison
   **Value:** ⭐⭐⭐ competitor comparison table

6. **Query:** `"Windsurf Codeium enterprise admin team policy deployment MDM"`
   **Top hits:** docs.windsurf.com/enterprise-policies, codeium.com/security (deployment tiers), docs.codeium.com guide-for-admins
   **Value:** ⭐⭐ Windsurf Enterprise policies details (Group Policy ADMX + FedRAMP High + Enterprise Hybrid/Self-hosted)

7. **Query:** `"OpenAI Codex enterprise org admin controls policy Codex Enterprise edition"`
   **Top hits:** developers.openai.com/codex/enterprise, developers.openai.com/codex/enterprise/managed-configuration, /codex/exec-policy (rules), help.openai.com codex admin guide
   **Value:** ⭐⭐⭐ Codex requirements.toml mechanism — very close to Claude managed-settings.json

8. **Query:** `"Claude Code claudeMdExcludes cannot exclude managed policy CLAUDE.md system directory enterprise"`
   **Top hits:** docs.anthropic.com/memory (official wording), GitHub issues #20880 + #34349 + #11872, mintlify settings reference
   **Value:** ⭐⭐⭐ verified official position on what cannot be excluded

9. **Query:** `"Anthropic Claude Code Ansible Puppet Chef playbook managed-settings.json deployment Linux"`
   **Top hits:** kodekloud.com (Claude Code + Ansible demo), systemprompt.io, Learn Linux TV YouTube
   **Value:** ⭐⭐ practical Ansible/Puppet/Chef templates

10. **Query:** `"Claude Code managed policy HIPAA SOC2 compliance audit logging fintech enterprise deployment"`
    **Top hits:** amitkoth.com SOC 2 guide, claude-ai.chat enterprise environments guide, Anthropic Compliance API help-center article
    **Value:** ⭐⭐⭐ compliance-specific use-cases + auditor perspective

### Exa web fetches — full content pulled

- https://docs.claude.com/en/docs/claude-code/settings — complete settings reference (all keys, precedence, merge behavior)
- https://code.claude.com/docs/en/server-managed-settings.md — server-managed mechanism
- https://code.claude.com/docs/en/permissions.md — managed-only permissions keys
- https://managed-settings.net/ — community-built reference + precedence diagram

---

## Key discoveries

1. **Two files, two purposes** — `managed-settings.json` (policy enforcement, primary) vs. Managed `CLAUDE.md` (behavioral memory, secondary). Many community write-ups conflate the two — this is the main correction surfaced by the research.

2. **Server-managed settings (Q4 2025+)** — third delivery mechanism beyond file-based and MDM. Rarely mentioned in community write-ups.

3. **Drop-in directory `managed-settings.d/`** (v2.1.83) — modular policy composition, systemd-style merge. Rarely documented outside official docs.

4. **Windows path deprecation v2.1.75** — `C:\ProgramData\ClaudeCode\` removed in favor of `C:\Program Files\ClaudeCode\`. Some mirrors still show the old path.

5. **claudeMdRequires** feature request still open — `claudeMdExcludes` can exclude project-level rule files, enterprise currently forced into monolithic managed CLAUDE.md.

6. **`--setting-source local` does NOT disable managed policies** — Anthropic official position (issue #11872 closed as "by design").

7. **Five distinct managed-only keys tiers:** `allowManagedXxxOnly` family (`PermissionRulesOnly`, `McpServersOnly`, `HooksOnly`, `DomainsOnly`, `ReadPathsOnly`) — combine for strict lockdown.

8. **MDM templates** officially published for 4 platforms: Jamf, Kandji (Iru), Intune, Group Policy — at github.com/anthropics/claude-code/tree/main/examples/mdm.

---

## Sources not fully explored (future research hooks)

- Anthropic trust.anthropic.com portal — SOC 2 certificate / report (requires login)
- Compliance API PDF (linked from help center) — detailed event types
- Specific Jamf ProfileCreator templates from Anthropic `examples/mdm` — acknowledged presence, not fetched content
- Live tests of:
  - `CLAUDE_MANAGED_SETTINGS_PATH` env var (unconfirmed officially)
  - `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var (unconfirmed, community-mentioned)
  - Behavior when managed policy file exists but not readable (graceful degradation confirmed theoretically, not tested)

---

## Time spent

Roughly 70 minutes end-to-end — source triage, 10 Exa queries, targeted fetches, and write-up across the 7 sections.
