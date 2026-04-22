# Claude Code Managed Policy: ресёрч по enterprise-деплою

> **Язык:** [English](README.md) · Русский

Практический справочник для тех, кто раскатывает [Claude Code](https://claude.com/claude-code) на уровне организации: IT-администраторы, InfoSec, DevOps, платформенные команды, техлиды в регулируемых индустриях.

Фокус: какие enterprise-контроли открывает Anthropic CLI, как их доставлять, что hard-enforceable (а что нет) и как это сравнивается с другими AI-инструментами для кода.

Собрано **2026-04-21** из официальной документации Anthropic, репозитория `anthropics/claude-code` на GitHub (issue tracker + examples) и community-гайдов по деплою. Каждое утверждение ведёт к источнику. Community-утверждения без подтверждения в официальных docs помечены как unverified.

---

## Что в репозитории

```
research/
├── README.md                      ← обзор + TL;DR + ключевые находки + все ссылки на источники
├── search-log.md                  ← воспроизводимость: какие запросы и источники прогоняли
│
├── section-1-paths/
│   └── paths-verified.md          ← пути managed-settings.json и CLAUDE.md на macOS / Linux / WSL / Windows
│
├── section-2-mdm-distribution/
│   └── distribution.md            ← три механизма доставки + паттерны Jamf / Kandji / Intune / Group Policy / Ansible
│
├── section-3-enforceable/
│   └── enforceable-keys.md        ← все managed-only ключи настроек (что контролируют, примеры значений)
│
├── section-4-bypass/
│   └── bypass-attempts.md         ← что юзер может обойти, а что нет; известные щели; аудит
│
├── section-5-use-cases/
│   └── real-world-use-cases.md    ← блокировка BYOK, coding standards под HIPAA/SOC 2, prod vs dev профили, audit trails
│
├── section-6-comparison/
│   └── competitors.md             ← Claude Code против Cursor / GitHub Copilot / OpenAI Codex / Windsurf
│
└── section-7-template/
    ├── managed-settings.json      ← готовый к адаптации fintech-baseline (команда ~60 разработчиков)
    ├── CLAUDE.md                  ← готовый к адаптации behavioral-policy (memory-слой)
    └── it-checklist.md            ← playbook раскатывания для IT / платформенной команды
```

---

## Ключевые выводы

1. **Managed policy это два файла, не один.** `managed-settings.json` даёт первичный hard enforcement: permissions, MCP allow/deny, allowlist моделей, plugins, hooks, sandbox, login method. Managed `CLAUDE.md` идёт вторым слоем, поведенческим: инструкции инжектируются в user-context, модель *может* их проигнорировать. На 95% compliance-запросов нужен именно `managed-settings.json`. Community-гайды, которые сводят «managed policy» к одному `CLAUDE.md`, пропускают большую часть контролей.

2. **Три механизма доставки (взаимоисключающие на managed-уровне):**
   - **Server-managed settings**: пушатся из Claude.ai admin-консоли при логине. MDM не нужен. Tier: Teams v2.1.38+ / Enterprise v2.1.30+. Подходит для BYOD и unmanaged-устройств.
   - **MDM / OS-level policies**: Jamf / Kandji / Intune / Group Policy пишут managed preferences domain; файлы на диск доставляются через Configuration Profile.
   - **File-based drop-in**: `managed-settings.json` + `managed-settings.d/*.json` (systemd-style merge по префиксу имени файла); доставка через Ansible / Puppet / Chef / shell-скрипт.

3. **Путь `C:\ProgramData\ClaudeCode\` deprecated.** Канонический путь на Windows → `C:\Program Files\ClaudeCode\` начиная с v2.1.75. Часть зеркал документации всё ещё показывает старый путь.

4. **Флаг `--setting-source local` НЕ обходит managed policy.** Подтверждено Anthropic в [issue #11872](https://github.com/anthropics/claude-code/issues/11872): *«Enterprise policies are not intended to be overridable».*

5. **Флаг `--dangerously-skip-permissions` глушится** через managed `disableBypassPermissionsMode: "disable"`. В связке с `permissions.deny` это главный путь hard enforcement.

6. **Managed `CLAUDE.md` невосприимчив к `claudeMdExcludes`**, а вот project-level `.claude/rules/*.md` юзер всё ещё может исключить. Feature request [`claudeMdRequires`](https://github.com/anthropics/claude-code/issues/34349) открыт, но не реализован. Workaround: кладите обязательные правила в монолитный managed `CLAUDE.md`, пока фича не приедет.

7. **Anthropic публикует официальные стартовые шаблоны** для четырёх MDM-платформ: [Jamf Pro, Kandji, Intune, Group Policy](https://github.com/anthropics/claude-code/tree/main/examples/mdm). По Linux официальных шаблонов нет → только community-материалы (Addigy, systemprompt.io, разные блоги).

8. **Неподтверждённые community-утверждения, помеченные внутри ресёрча:**
   - Переменная окружения `CLAUDE_MANAGED_SETTINGS_PATH`: встречается в mintlify-style форках, в официальных docs v2.1.x отсутствует.
   - Переменная окружения `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1`: упомянута в community best-practices репозитории, не верифицирована.

   Прежде чем опираться на них, проверьте в своей среде.

---

## Первоисточники (Anthropic)

- Settings reference: https://docs.claude.com/en/docs/claude-code/settings
- Memory / CLAUDE.md hierarchy: https://docs.anthropic.com/en/docs/claude-code/memory
- Server-managed settings: https://code.claude.com/docs/en/server-managed-settings.md
- Permissions (managed-only keys): https://code.claude.com/docs/en/permissions.md
- Third-party integrations (обзор enterprise-деплоя): https://docs.claude.com/en/docs/claude-code/third-party-integrations
- Claude Code for Enterprise (продуктовая страница): https://claude.com/product/claude-code/enterprise
- Официальные MDM-шаблоны: https://github.com/anthropics/claude-code/tree/main/examples/mdm

## Community-источники

- `managed-settings.net`: community-конструктор настроек → https://managed-settings.net/
- Playbook по enterprise-раскатыванию: https://systemprompt.io/guides/enterprise-claude-code-managed-settings
- Playbook на команду 50+ разработчиков: https://systemprompt.io/guides/claude-code-organisation-rollout
- Walkthrough по Addigy MDM: https://addigy.com/blog/manage-claude-code-policies-addigy/
- Settings reference (2026-04-18): https://claudefa.st/blog/guide/settings-reference
- Практический гайд по SOC 2 / HIPAA: https://claude-ai.chat/blog/claude-code-in-enterprise-environments/
- SOC 2 глазами аудитора: https://amitkoth.com/claude-code-soc2-compliance-auditor-guide/
- Полный config-гайд v2.1.114: https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-settings.md

## Связанные GitHub issues

- [`claudeMdRequires`](https://github.com/anthropics/claude-code/issues/34349): feature request, open
- [Parent CLAUDE.md exclusion](https://github.com/anthropics/claude-code/issues/20880): open
- [`--setting-source local` с managed policy](https://github.com/anthropics/claude-code/issues/11872): closed, by-design

## Enterprise-документация конкурентов (для сравнительной секции)

- Cursor Enterprise: https://cursor.com/docs/account/teams/enterprise-settings
- GitHub Copilot enterprise policies: https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies
- OpenAI Codex Enterprise (`requirements.toml`): https://developers.openai.com/codex/enterprise/managed-configuration/
- Windsurf enterprise policies: https://docs.windsurf.com/windsurf/enterprise-policies
- Windsurf deployment tiers (включая self-hosted / FedRAMP High): https://codeium.com/security

---

## Как пользоваться репозиторием

- **Прицеливаетесь на enterprise-внедрение**: начните с `research/README.md`. Самодостаточный обзор: TL;DR-матрица, ключевые находки, gaps, все ссылки на источники.
- **Пишете policy-файл прямо сейчас**: идите в `research/section-7-template/`. Адаптируйте `managed-settings.json` и `CLAUDE.md`. Пройдите по `it-checklist.md`.
- **Выбираете между AI-инструментами**: начните с `research/section-6-comparison/competitors.md`.
- **Юзеры пытаются обойти уже раскатанную policy**: `research/section-4-bypass/bypass-attempts.md` описывает, что работает, что нет и как аудитить.
- **Воспроизводите или расширяете ресёрч**: `research/search-log.md` фиксирует, какие запросы и источники прогоняли.

---

## Оговорки

- Отражает состояние документации и публичных community-отчётов на **2026-04-21**. Claude Code обновляется быстро; перед раскатыванием сверяйтесь с актуальной документацией.
- fintech-шаблон в `section-7-template/` это **baseline для адаптации**, не compliance-сертификация. Адаптируйте под конкретный регуляторный режим (HIPAA, SOC 2, PCI-DSS, FedRAMP) вместе с юристами и security-командой.
- У части managed-only ключей, задокументированных Anthropic, пока нет публичных примеров. Они упомянуты inline с пометкой, а не выдаются за утверждение.
- Там, где этот ресёрч расходится с популярным community-гайдом, расхождение явно помечено и ведёт к авторитетному источнику Anthropic.

---

## Лицензия

Содержимое репозитория распространяется по лицензии MIT → см. [`LICENSE`](LICENSE).

Claude, Claude Code и Anthropic: торговые марки Anthropic PBC. Этот репозиторий это независимый research-справочник, он не аффилирован с Anthropic и ей не одобрен.
