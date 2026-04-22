# Claude Code Managed Policy — Enterprise Deployment Research

> **Language:** English · [Русский](README.ru.md)

Practical reference for anyone rolling out [Claude Code](https://claude.com/claude-code) at the organizational level: IT admins, InfoSec, DevOps, platform teams, tech leads in regulated industries.

Focus: the enterprise controls the Anthropic CLI exposes, how to distribute them, what is (and isn't) hard-enforceable, and how it compares to other AI coding tools.

Compiled **2026-04-21** from Anthropic official documentation, the `anthropics/claude-code` GitHub repository (issue tracker + examples), and community deployment guides. Every claim is linked to its source. Community claims not backed by official docs are explicitly flagged as unverified.

---

## What's in this repo

```
research/
├── README.md                      ← overview + TL;DR + key findings + all source links
├── search-log.md                  ← reproducibility: what queries / sources were run
│
├── section-1-paths/
│   └── paths-verified.md          ← managed-settings.json & CLAUDE.md paths on macOS / Linux / WSL / Windows
│
├── section-2-mdm-distribution/
│   └── distribution.md            ← three delivery mechanisms + Jamf / Kandji / Intune / Group Policy / Ansible patterns
│
├── section-3-enforceable/
│   └── enforceable-keys.md        ← every managed-only settings key (what it controls, example values)
│
├── section-4-bypass/
│   └── bypass-attempts.md         ← what users can and cannot bypass; known gaps; audit considerations
│
├── section-5-use-cases/
│   └── real-world-use-cases.md    ← BYOK block, HIPAA/SOC 2 coding standards, prod-vs-dev profiles, audit trails
│
├── section-6-comparison/
│   └── competitors.md             ← Claude Code vs Cursor / GitHub Copilot / OpenAI Codex / Windsurf
│
└── section-7-template/
    ├── managed-settings.json      ← ready-to-adapt fintech baseline (60-developer team scale)
    ├── CLAUDE.md                  ← ready-to-adapt behavioral policy (memory layer)
    └── it-checklist.md            ← IT / platform rollout playbook
```

---

## Key takeaways

1. **Managed policy = two files, not one.** `managed-settings.json` is the primary hard-enforcement surface: permissions, MCP allow/deny, model allowlist, plugins, hooks, sandbox, login method. Managed `CLAUDE.md` is a secondary behavioral layer — instructions injected as user-context that the model *can* ignore. For 95% of compliance asks, `managed-settings.json` is what you need; community write-ups that reduce "managed policy" to just `CLAUDE.md` miss the larger control surface.

2. **Three delivery mechanisms (mutually exclusive at the managed tier):**
   - **Server-managed settings** — pushed from Claude.ai admin console at login. No MDM required. Teams v2.1.38+ / Enterprise v2.1.30+. Suits BYOD / unmanaged devices.
   - **MDM / OS-level policies** — Jamf / Kandji / Intune / Group Policy write the managed preferences domain; files on disk delivered via Configuration Profile.
   - **File-based drop-in** — `managed-settings.json` + `managed-settings.d/*.json` (systemd-style merge by filename prefix); delivered via Ansible / Puppet / Chef / shell script.

3. **`C:\ProgramData\ClaudeCode\` is deprecated.** Canonical Windows path is `C:\Program Files\ClaudeCode\` as of v2.1.75. Several docs mirrors still show the old path.

4. **`--setting-source local` does NOT override managed policy.** Confirmed by Anthropic in [issue #11872](https://github.com/anthropics/claude-code/issues/11872): *"Enterprise policies are not intended to be overridable."*

5. **`--dangerously-skip-permissions` can be killed** via managed `disableBypassPermissionsMode: "disable"`. Combined with `permissions.deny`, this is the hard enforcement path.

6. **Managed `CLAUDE.md` is immune to `claudeMdExcludes`**, but project-level `.claude/rules/*.md` files are not — a user can still exclude them. Feature request [`claudeMdRequires`](https://github.com/anthropics/claude-code/issues/34349) is open but unimplemented. Workaround: put mandatory rules in the monolithic managed `CLAUDE.md` until that ships.

7. **Anthropic publishes official starter templates** for four MDM platforms: [Jamf Pro, Kandji, Intune, Group Policy](https://github.com/anthropics/claude-code/tree/main/examples/mdm). Linux guidance is community-only (Addigy, systemprompt.io, various blogs).

8. **Unverified community claims flagged inside the research:**
   - `CLAUDE_MANAGED_SETTINGS_PATH` environment variable — appears in mintlify-style forks, not in official v2.1.x docs.
   - `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` environment variable — mentioned in a community best-practices repo, unverified.

   Test in your own environment before depending on them.

---

## Primary sources (Anthropic)

- Settings reference — https://docs.claude.com/en/docs/claude-code/settings
- Memory / CLAUDE.md hierarchy — https://docs.anthropic.com/en/docs/claude-code/memory
- Server-managed settings — https://code.claude.com/docs/en/server-managed-settings.md
- Permissions (managed-only keys) — https://code.claude.com/docs/en/permissions.md
- Third-party integrations (enterprise deployment overview) — https://docs.claude.com/en/docs/claude-code/third-party-integrations
- Claude Code for Enterprise product page — https://claude.com/product/claude-code/enterprise
- Official MDM starter templates — https://github.com/anthropics/claude-code/tree/main/examples/mdm

## Community references

- `managed-settings.net` — community settings builder: https://managed-settings.net/
- Enterprise rollout playbook: https://systemprompt.io/guides/enterprise-claude-code-managed-settings
- 50+ developer rollout playbook: https://systemprompt.io/guides/claude-code-organisation-rollout
- Addigy MDM walkthrough: https://addigy.com/blog/manage-claude-code-policies-addigy/
- Settings reference (2026-04-18): https://claudefa.st/blog/guide/settings-reference
- SOC 2 / HIPAA practical guide: https://claude-ai.chat/blog/claude-code-in-enterprise-environments/
- SOC 2 auditor perspective: https://amitkoth.com/claude-code-soc2-compliance-auditor-guide/
- Complete config guide v2.1.114: https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-settings.md

## Related GitHub issues

- [`claudeMdRequires`](https://github.com/anthropics/claude-code/issues/34349) — feature request, open
- [Parent CLAUDE.md exclusion](https://github.com/anthropics/claude-code/issues/20880) — open
- [`--setting-source local` with managed policy](https://github.com/anthropics/claude-code/issues/11872) — closed as by-design

## Competitor enterprise docs (for the comparison section)

- Cursor Enterprise — https://cursor.com/docs/account/teams/enterprise-settings
- GitHub Copilot enterprise policies — https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies
- OpenAI Codex Enterprise (`requirements.toml`) — https://developers.openai.com/codex/enterprise/managed-configuration/
- Windsurf enterprise policies — https://docs.windsurf.com/windsurf/enterprise-policies
- Windsurf deployment tiers (including self-hosted / FedRAMP High) — https://codeium.com/security

---

## How to use this repo

- **Scoping enterprise adoption** — start with `research/README.md`. Self-contained overview, TL;DR matrix, key findings, gaps, every source link.
- **Writing a policy file now** — jump to `research/section-7-template/`. Adapt `managed-settings.json` and `CLAUDE.md`. Work through `it-checklist.md`.
- **Deciding between AI coding tools** — start with `research/section-6-comparison/competitors.md`.
- **Users trying to bypass an existing policy** — `research/section-4-bypass/bypass-attempts.md` covers what works, what doesn't, and how to audit.
- **Reproducing or extending the research** — `research/search-log.md` documents which queries and sources were run.

---

## Caveats

- Reflects documentation and public community reporting as of **2026-04-21**. Claude Code ships fast; re-verify against current docs before rolling out.
- The fintech template in `section-7-template/` is a **baseline for adaptation**, not a compliance certification. Adapt to your specific regulatory regime (HIPAA, SOC 2, PCI-DSS, FedRAMP) with your legal and security teams.
- A handful of managed-only settings keys documented by Anthropic have no public examples yet; they are noted in-line rather than asserted.
- Where this research contradicts a widely-circulated community guide, the contradiction is explicit and linked to the authoritative Anthropic source.

---

## License

Content in this repository is released under the MIT License — see [`LICENSE`](LICENSE).

Claude, Claude Code, and Anthropic are trademarks of Anthropic PBC. This repository is an independent research reference and is not affiliated with or endorsed by Anthropic.
