# Claude Code Managed Policy: ресёрч по enterprise-деплою

> **Язык:** [English](README.md) · Русский

Практический справочник для тех, кто внедряет [Claude Code](https://claude.com/claude-code) на уровне организации: IT-администраторы, InfoSec, DevOps, платформенные команды, техлиды в регулируемых индустриях.

В фокусе — какие enterprise-контроли даёт Anthropic CLI, как их раскатывать, что именно hard-enforceable (а что нет), и как это сравнивается с другими AI-инструментами для кода.

Материал собран **2026-04-21** из официальной документации Anthropic, репозитория `anthropics/claude-code` на GitHub (issue tracker + examples), и community-гайдов по деплою. Каждое утверждение сопровождается ссылкой на источник. Community-утверждения, которые не подтверждаются официальной документацией, отдельно помечены как unverified.

---

## Что в репозитории

```
research/
├── README.md                      ← обзор + TL;DR + ключевые находки + все ссылки
├── search-log.md                  ← reproducibility: какие запросы и источники прогонялись
│
├── section-1-paths/
│   └── paths-verified.md          ← пути managed-settings.json и CLAUDE.md на macOS / Linux / WSL / Windows
│
├── section-2-mdm-distribution/
│   └── distribution.md            ← три механизма доставки + паттерны Jamf / Kandji / Intune / Group Policy / Ansible
│
├── section-3-enforceable/
│   └── enforceable-keys.md        ← все managed-only ключи настроек (что контролируют, примеры)
│
├── section-4-bypass/
│   └── bypass-attempts.md         ← что пользователи могут и не могут обойти; известные архитектурные щели; аудит
│
├── section-5-use-cases/
│   └── real-world-use-cases.md    ← блокировка BYOK, HIPAA/SOC 2 coding standards, prod vs dev профили, audit trails
│
├── section-6-comparison/
│   └── competitors.md             ← Claude Code против Cursor / GitHub Copilot / OpenAI Codex / Windsurf
│
└── section-7-template/
    ├── managed-settings.json      ← готовая fintech-baseline (команда ~60 разработчиков)
    ├── CLAUDE.md                  ← готовая behavioral-policy (memory слой)
    └── it-checklist.md            ← playbook для IT / платформенной команды на раскатывание
```

---

## Ключевые выводы

1. **Managed policy — это два файла, не один.** `managed-settings.json` — основная поверхность hard enforcement: permissions, MCP allow/deny, model allowlist, plugins, hooks, sandbox, login method. Managed `CLAUDE.md` — вторичный behavioral слой: инструкции, инжектируемые в user-context, которые модель *может* проигнорировать. Для 95% compliance-задач нужен именно `managed-settings.json`. Community-гайды, которые сводят «managed policy» только к `CLAUDE.md`, пропускают большую часть контролей.

2. **Три механизма доставки (взаимоисключающие на managed-уровне):**
   - **Server-managed settings** — пушатся из Claude.ai admin console при логине. MDM не требуется. Teams v2.1.38+ / Enterprise v2.1.30+. Подходит для BYOD и unmanaged устройств.
   - **MDM / OS-level policies** — Jamf / Kandji / Intune / Group Policy пишут managed preferences domain; файлы на диск доставляются через Configuration Profile.
   - **File-based drop-in** — `managed-settings.json` + `managed-settings.d/*.json` (systemd-style merge по префиксу имени файла); деплой через Ansible / Puppet / Chef / shell-скрипт.

3. **`C:\ProgramData\ClaudeCode\` deprecated.** Канонический путь на Windows — `C:\Program Files\ClaudeCode\` начиная с v2.1.75. Некоторые зеркала документации всё ещё показывают старый путь.

4. **`--setting-source local` НЕ переопределяет managed policy.** Подтверждено Anthropic в [issue #11872](https://github.com/anthropics/claude-code/issues/11872): *«Enterprise policies are not intended to be overridable».*

5. **`--dangerously-skip-permissions` можно отключить** через managed `disableBypassPermissionsMode: "disable"`. В комбинации с `permissions.deny` это основной путь hard enforcement.

6. **Managed `CLAUDE.md` неподвержен `claudeMdExcludes`**, но project-level файлы `.claude/rules/*.md` — подвержены, пользователь может их исключить. Feature request [`claudeMdRequires`](https://github.com/anthropics/claude-code/issues/34349) открыт, но пока не реализован. Workaround: класть обязательные правила в монолитный managed `CLAUDE.md`.

7. **Anthropic публикует официальные starter-шаблоны** для четырёх MDM-платформ: [Jamf Pro, Kandji, Intune, Group Policy](https://github.com/anthropics/claude-code/tree/main/examples/mdm). По Linux — только community-гайды (Addigy, systemprompt.io, разные блоги).

8. **Community-утверждения, не подтверждённые в ресёрче:**
   - Переменная окружения `CLAUDE_MANAGED_SETTINGS_PATH` — встречается в mintlify-подобных форках, но отсутствует в официальных docs v2.1.x.
   - Переменная окружения `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` — упоминается в community best-practices репозитории, не верифицирована.

   Проверяйте в своей среде, прежде чем на них полагаться.

---

## Первоисточники (Anthropic)

- Settings reference — https://docs.claude.com/en/docs/claude-code/settings
- Memory / CLAUDE.md hierarchy — https://docs.anthropic.com/en/docs/claude-code/memory
- Server-managed settings — https://code.claude.com/docs/en/server-managed-settings.md
- Permissions (managed-only keys) — https://code.claude.com/docs/en/permissions.md
- Third-party integrations (overview по enterprise-деплою) — https://docs.claude.com/en/docs/claude-code/third-party-integrations
- Claude Code for Enterprise (продуктовая страница) — https://claude.com/product/claude-code/enterprise
- Официальные MDM starter-шаблоны — https://github.com/anthropics/claude-code/tree/main/examples/mdm

## Community-источники

- `managed-settings.net` — community-конструктор настроек: https://managed-settings.net/
- Playbook по enterprise-раскатыванию: https://systemprompt.io/guides/enterprise-claude-code-managed-settings
- Playbook на 50+ разработчиков: https://systemprompt.io/guides/claude-code-organisation-rollout
- Walkthrough по Addigy MDM: https://addigy.com/blog/manage-claude-code-policies-addigy/
- Settings reference (2026-04-18): https://claudefa.st/blog/guide/settings-reference
- Практический гайд SOC 2 / HIPAA: https://claude-ai.chat/blog/claude-code-in-enterprise-environments/
- SOC 2 с точки зрения аудитора: https://amitkoth.com/claude-code-soc2-compliance-auditor-guide/
- Полный config-гайд v2.1.114: https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-settings.md

## Связанные GitHub issues

- [`claudeMdRequires`](https://github.com/anthropics/claude-code/issues/34349) — feature request, open
- [Parent CLAUDE.md exclusion](https://github.com/anthropics/claude-code/issues/20880) — open
- [`--setting-source local` с managed policy](https://github.com/anthropics/claude-code/issues/11872) — closed, by-design

## Enterprise-документация конкурентов (для сравнительной секции)

- Cursor Enterprise — https://cursor.com/docs/account/teams/enterprise-settings
- GitHub Copilot enterprise policies — https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies
- OpenAI Codex Enterprise (`requirements.toml`) — https://developers.openai.com/codex/enterprise/managed-configuration/
- Windsurf enterprise policies — https://docs.windsurf.com/windsurf/enterprise-policies
- Windsurf deployment tiers (включая self-hosted / FedRAMP High) — https://codeium.com/security

---

## Как пользоваться репозиторием

- **Оцениваете стоит ли подключаться** — начните с `research/README.md`. Это самодостаточный обзор: TL;DR-матрица, ключевые выводы, gaps, все ссылки на источники.
- **Пишете policy-файл сейчас** — идите в `research/section-7-template/`. Адаптируйте `managed-settings.json` и `CLAUDE.md`. Проходите по `it-checklist.md`.
- **Выбираете между AI-инструментами** — начните с `research/section-6-comparison/competitors.md`.
- **Пользователи пытаются обойти уже раскатанную policy** — `research/section-4-bypass/bypass-attempts.md` описывает что работает, что нет, и как аудитить.
- **Воспроизводите или расширяете ресёрч** — `research/search-log.md` документирует какие запросы и источники прогонялись.

---

## Оговорки

- Отражает состояние документации и публичных community-отчётов на **2026-04-21**. Claude Code обновляется быстро — перед раскатыванием сверяйтесь с актуальной документацией.
- fintech-шаблон в `section-7-template/` — это **baseline для адаптации**, не compliance-сертификация. Адаптируйте под конкретный регуляторный режим (HIPAA, SOC 2, PCI-DSS, FedRAMP) совместно с юристами и security-командой.
- Часть managed-only ключей, задокументированных Anthropic, пока не имеет публичных примеров использования — они упомянуты in-line с пометкой, а не даются как утверждение.
- Где этот ресёрч расходится с популярными community-гайдами, расхождение явно помечено и сопровождается ссылкой на авторитетный источник Anthropic.

---

## Лицензия

Содержимое репозитория распространяется по лицензии MIT — см. [`LICENSE`](LICENSE).

Claude, Claude Code и Anthropic — торговые марки Anthropic PBC. Этот репозиторий — независимый research-справочник, не аффилирован с Anthropic и не одобрен ей.
