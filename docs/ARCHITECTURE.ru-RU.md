# Архитектура GSD

> Системная архитектура для контрибьюторов и продвинутых пользователей. Пользовательская документация — [Feature Reference](FEATURES.md) или [User Guide](USER-GUIDE.ru-RU.md).

---

## Оглавление

- [Обзор системы](#обзор-системы)
- [Принципы проектирования](#принципы-проектирования)
- [Архитектура компонентов](#архитектура-компонентов)
- [Модель агентов](#модель-агентов)
- [Поток данных](#поток-данных)
- [Раскладка файловой системы](#раскладка-файловой-системы)
- [Архитектура установщика](#архитектура-установщика)
- [Система хуков](#система-хуков)
- [Слой CLI-инструментов](#слой-cli-инструментов)
- [Абстракция рантайма](#абстракция-рантайма)

---

## Обзор системы

GSD — это **фреймворк мета-промптинга**, сидящий между пользователем и AI coding-агентами (Claude Code, Gemini CLI, OpenCode, Kilo, Codex, Copilot, Antigravity, Trae, Cline, Augment Code). Он предоставляет:

1. **Контекстную инженерию** — структурированные артефакты, дающие AI всё что нужно на каждую задачу
2. **Мультиагентную оркестрацию** — тонкие оркестраторы, спавнящие специализированных агентов со свежими контекстными окнами
3. **Spec-driven разработку** — пайплайн требования → research → планы → исполнение → верификация
4. **Управление состоянием** — постоянная память проекта через сессии и сбросы контекста

```
┌──────────────────────────────────────────────────────┐
│                      ПОЛЬЗОВАТЕЛЬ                    │
│            /gsd-command [args]                       │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│              СЛОЙ КОМАНД                             │
│   commands/gsd/*.md — Prompt-based командные файлы   │
│   (Кастомные команды Claude Code / скиллы Codex)     │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│              СЛОЙ WORKFLOW                           │
│   get-shit-done/workflows/*.md — Логика оркестрации  │
│   (Читает референсы, спавнит агентов, ведёт состояние) │
└──────┬──────────────┬─────────────────┬──────────────┘
       │              │                 │
┌──────▼──────┐ ┌─────▼─────┐ ┌────────▼───────┐
│  АГЕНТ      │ │  АГЕНТ    │ │  АГЕНТ         │
│  (свежий    │ │  (свежий  │ │  (свежий       │
│   контекст) │ │   контекст)│ │   контекст)    │
└──────┬──────┘ └─────┬─────┘ └────────┬───────┘
       │              │                 │
┌──────▼──────────────▼─────────────────▼──────────────┐
│              СЛОЙ CLI-ИНСТРУМЕНТОВ                   │
│   gsd-sdk query (sdk/src/query) + gsd-tools.cjs      │
│   Программный SDK-мост: GSDTools/query-runtime-bridge.ts │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│              ФАЙЛОВАЯ СИСТЕМА (.planning/)           │
│   PROJECT.md | REQUIREMENTS.md | ROADMAP.md          │
│   STATE.md | config.json | phases/ | research/       │
└──────────────────────────────────────────────────────┘
```

---

## Принципы проектирования

### 1. Свежий контекст на каждого агента

Каждый агент, заспавненный оркестратором, получает чистое контекстное окно (до 200K токенов). Это устраняет context rot — деградацию качества, происходящую по мере того как AI заполняет своё контекстное окно накопленным диалогом.

### 2. Тонкие оркестраторы

Workflow-файлы (`get-shit-done/workflows/*.md`) никогда не делают тяжёлую работу. Они:

- Грузят контекст через `gsd-sdk query init.<workflow>` (или легаси `gsd-tools.cjs init <workflow>`)
- Спавнят специализированных агентов с фокусными промптами
- Собирают результаты и роутят в следующий шаг
- Обновляют состояние между шагами

### 3. Файло-ориентированное состояние

Всё состояние живёт в `.planning/` как human-readable Markdown и JSON. Никакой БД, никакого сервера, никаких внешних зависимостей. Это значит:

- Состояние переживает сбросы контекста (`/clear`)
- Состояние инспектится и людьми, и агентами
- Состояние можно закоммитить в git для видимости команде

### 4. Absent = Enabled

Фичефлаги воркфлоу следуют паттерну **absent = enabled**. Если ключ отсутствует в `config.json` — он по умолчанию `true`. Пользователи явно отключают фичи; им не надо включать дефолты.

### 5. Глубокая защита (Defense in Depth)

Несколько слоёв предотвращают типичные режимы сбоя:

- Планы верифицируются до исполнения (агент plan-checker)
- Исполнение производит атомарные коммиты на каждую задачу
- Постфактум-верификация проверяет против целей фазы
- UAT даёт человеческую верификацию как финальный gate

---

## Архитектура компонентов

### Команды (`commands/gsd/*.md`)

Точки входа для пользователя. Каждый файл содержит YAML-frontmatter (name, description, allowed-tools) и тело-промпт, бутстрапящий воркфлоу. Команды устанавливаются как:

- **Claude Code:** Кастомные slash-команды (форма через дефис, `/gsd-command-name`)
- **OpenCode / Kilo:** Slash-команды (форма через дефис, `/gsd-command-name`)
- **Codex:** Скиллы (`$gsd-command-name`)
- **Copilot:** Slash-команды (форма через дефис, `/gsd-command-name`)
- **Gemini CLI:** Slash-команды под namespace `gsd:` (форма через двоеточие, `/gsd:command-name`) — Gemini неймспейсит все кастомные команды под id своего плагина, поэтому путь установки переписывает все ссылки в тексте на форму с двоеточием
- **Antigravity:** Скиллы

**Всего команд:** см. [`docs/INVENTORY.md`](INVENTORY.md#commands) для авторитетного счёта и полного roster'а.

#### Двухступенчатый иерархический роутинг (v1.40, [#2792](https://github.com/gsd-build/get-shit-done/issues/2792))

Чтобы держать токен-стоимость eager skill-листинга низкой, v1.40 вводит шесть namespace **мета-скилов** (`gsd-workflow`, `gsd-project`, `gsd-quality`, `gsd-context`, `gsd-manage`, `gsd-ideate` — из `commands/gsd/ns-*.md`, но вызываемое `name:` — это голая форма как показано здесь), наслоённых поверх конкретных sub-скилов. Модель видит 6 namespace-роутеров (~120 токенов) вместо плоского листинга 86 скиллов (~2150 токенов), выбирает namespace, затем роутит на конкретный sub-скилл через таблицу роутинга, встроенную в тело namespace-роутера. Namespace-скиллы **аддитивны** — каждая конкретная команда по-прежнему вызывается напрямую.

Описания роутеров используют pipe-разделённые keyword-теги (≤ 60 символов) согласно исследованию Tool Attention, показывающему что keyword-плотные теги превосходят прозу в роутинге при ~40 % токенов.

#### Взаимодействие с MCP-токен-бюджетом

Eager skill-листинг — одна из двух повторяющихся per-turn токен-затрат. Другая — MCP tool-схема, инжектируемая каждым включённым MCP-сервером в `.claude/settings.json`. Тяжёлые MCP-серверы (browser/playwright, Mac-tools, Windows-tools) могут стоить 20k+ токенов за turn каждый — часто перевешивая то, что экономит тюнинг `model_profile`. Переключатель живёт в harness'е Claude Code (`enabledMcpjsonServers` / `disabledMcpjsonServers` в `.claude/settings.json`) и **не** забота GSD. Вместе двухступенчатый слой роутинга (#2792) и дисциплинированное включение MCP — самые крупные рычаги стоимости на turn. См. [`docs/USER-GUIDE.md`](USER-GUIDE.ru-RU.md) и `references/context-budget.md` для чек-листа аудита.

### Workflow'ы (`get-shit-done/workflows/*.md`)

Логика оркестрации, на которую ссылаются команды. Содержит пошаговый процесс, включая:

- Загрузку контекста через init-handler'ы `gsd-sdk query` (или легаси `gsd-tools.cjs init`)
- Инструкции спавна агентов с резолвом моделей
- Определения gate'ов/чекпоинтов
- Паттерны обновления состояния
- Обработку ошибок и восстановление

**Всего workflow'ов:** см. [`docs/INVENTORY.md`](INVENTORY.md#workflows) для авторитетного счёта.

#### Progressive disclosure для воркфлоу

Workflow-файлы загружаются дословно в контекст Claude каждый раз, когда вызывается соответствующая команда `/gsd-*`. Чтобы держать эту стоимость ограниченной, бюджет размера воркфлоу, форсимый `tests/workflow-size-budget.test.cjs`, зеркалит бюджет агентов из #2361:

| Tier      | Лимит строк на файл |
|-----------|---------------------|
| `XL`      | 1700 — top-level оркестраторы (`execute-phase`, `plan-phase`, `new-project`) |
| `LARGE`   | 1500 — многошаговые планировщики и большие feature-workflow |
| `DEFAULT` | 1000 — фокусные single-purpose workflow (целевой tier) |

`workflows/discuss-phase.md` держится в более строгом потолке <500 строк по issue #2551. Когда workflow перерастает свой tier, выдели per-mode тела в `workflows/<workflow>/modes/<mode>.md`, шаблоны в `workflows/<workflow>/templates/`, общее знание в `get-shit-done/references/`. Родительский файл становится тонким диспатчером, который Read'ит только нужные mode- и template-файлы.

`workflows/discuss-phase/` — канонический пример этого паттерна — родитель диспатчит, modes/ держит per-flag поведение (`power.md`, `all.md`, `auto.md`, `chain.md`, `text.md`, `batch.md`, `analyze.md`, `default.md`, `advisor.md`), templates/ держит схемы CONTEXT.md, DISCUSSION-LOG.md и checkpoint.json, читаемые только когда пишется соответствующий output.

### Агенты (`agents/*.md`)

Специализированные определения агентов с frontmatter'ом:

- `name` — Идентификатор агента
- `description` — Роль и назначение
- `tools` — Разрешённый инструментальный доступ (Read, Write, Edit, Bash, Grep, Glob, WebSearch и т.д.)
- `color` — Цвет терминального вывода для визуального различия

**Всего агентов:** 33

### Референсы (`get-shit-done/references/*.md`)

Общие knowledge-документы, на которые workflow'ы и агенты ссылаются через `@-reference` (см. [`docs/INVENTORY.md`](INVENTORY.md#references-41-shipped) для авторитетного счёта):

**Core-референсы:**

- `checkpoints.md` — Определения типов чекпоинтов и паттерны взаимодействия
- `gates.md` — 4 канонических типа gate'ов (Confirm, Quality, Safety, Transition), проводных в plan-checker и verifier
- `model-profiles.md` — Per-agent назначения tier'ов моделей
- `model-profile-resolution.md` — Документация алгоритма резолва моделей
- `verification-patterns.md` — Как верифицировать разные типы артефактов
- `verification-overrides.md` — Per-artifact правила override верификации
- `planning-config.md` — Полная схема конфига и поведение
- `git-integration.md` — Паттерны git-коммитов, ветвления, истории
- `git-planning-commit.md` — Конвенции коммитов директории planning
- `questioning.md` — Философия dream extraction для инициализации проекта
- `tdd.md` — Паттерны интеграции TDD
- `ui-brand.md` — Паттерны форматирования визуального вывода
- `common-bug-patterns.md` — Общие баг-паттерны для code-review и верификации

**Workflow-референсы:**

- `agent-contracts.md` — Формальный интерфейс между оркестраторами и агентами
- `context-budget.md` — Правила аллокации бюджета контекстного окна
- `continuation-format.md` — Формат продолжения/возобновления сессии
- `domain-probes.md` — Доменные probing-вопросы для discuss-phase
- `gate-prompts.md` — Шаблоны промптов gate/checkpoint
- `revision-loop.md` — Паттерны итерации revision-планов
- `universal-anti-patterns.md` — Общие анти-паттерны для детекции и избегания
- `artifact-types.md` — Определения типов планировочных артефактов
- `phase-argument-parsing.md` — Конвенции парсинга аргументов фаз
- `decimal-phase-calculation.md` — Правила нумерации десятичных sub-фаз
- `workstream-flag.md` — Конвенции workstream-active указателя
- `user-profiling.md` — Методология поведенческого профайлинга пользователя
- `thinking-partner.md` — Условная активация thinking partner'а в точках решений

**Референсы thinking-моделей:**

Референсы для интеграции thinking-class моделей (o3, o4-mini, Gemini 2.5 Pro) в воркфлоу GSD:

- `thinking-models-debug.md` — Паттерны thinking-моделей для debug-воркфлоу
- `thinking-models-execution.md` — Паттерны thinking-моделей для execution-агентов
- `thinking-models-planning.md` — Паттерны thinking-моделей для planning-агентов
- `thinking-models-research.md` — Паттерны thinking-моделей для research-агентов
- `thinking-models-verification.md` — Паттерны thinking-моделей для verification-агентов

**Модульная декомпозиция планировщика:**

Агент планировщика (`agents/gsd-planner.md`) был декомпозирован из одного монолитного файла в core-агента плюс reference-модули, чтобы оставаться под лимитом 50K символов, наложенным некоторыми рантаймами:

- `planner-gap-closure.md` — Поведение gap-closure режима (читает VERIFICATION.md, таргетный replanning)
- `planner-reviews.md` — Интеграция cross-AI ревью (читает REVIEWS.md от `/gsd-review`)
- `planner-revision.md` — Паттерны ревизии планов для итеративной доработки

### Шаблоны (`get-shit-done/templates/`)

Markdown-шаблоны для всех планировочных артефактов. Используются `gsd-sdk query template.fill` / `phase.scaffold` (и легаси `gsd-tools.cjs template fill` / top-level `scaffold`) для создания пред-структурированных файлов:
- `project.md`, `requirements.md`, `roadmap.md`, `state.md` — Core project-файлы
- `phase-prompt.md` — Шаблон промпта исполнения фазы
- `summary.md` (+ `summary-minimal.md`, `summary-standard.md`, `summary-complex.md`) — Granularity-aware шаблоны summary
- `DEBUG.md` — Шаблон трекинга debug-сессии
- `UI-SPEC.md`, `UAT.md`, `VALIDATION.md` — Специализированные шаблоны верификации
- `discussion-log.md` — Шаблон discussion audit-trail
- `codebase/` — Brownfield mapping шаблоны (stack, architecture, conventions, concerns, structure, testing, integrations)
- `research-project/` — Шаблоны research-output (SUMMARY, STACK, FEATURES, ARCHITECTURE, PITFALLS)

### Хуки (`hooks/`)

Runtime-хуки, интегрирующиеся с host AI-агентом:

| Хук | Событие | Назначение |
|-----|---------|------------|
| `gsd-statusline.js` | `statusLine` | Отображает модель, задачу, директорию и бар использования контекста |
| `gsd-context-monitor.js` | `PostToolUse` / `AfterTool` | Инжектит agent-facing предупреждения о контексте на 35%/25% остатка |
| `gsd-check-update.js` | `SessionStart` | Foreground-триггер для фоновой проверки обновлений |
| `gsd-check-update-worker.js` | (helper) | Фоновый worker, спавнящийся `gsd-check-update.js`; без прямой регистрации события |
| `gsd-prompt-guard.js` | `PreToolUse` | Сканит записи в `.planning/` на паттерны prompt injection (advisory) |
| `gsd-read-injection-scanner.js` | `PostToolUse` | Сканит вывод Read-инструмента на инжектированные инструкции в untrusted-контенте |
| `gsd-workflow-guard.js` | `PreToolUse` | Детектит правки файлов вне GSD-воркфлоу (advisory, opt-in через `hooks.workflow_guard`) |
| `gsd-read-guard.js` | `PreToolUse` | Advisory guard, предотвращающий Edit/Write на файлах, не прочитанных в сессии |
| `gsd-session-state.sh` | `PostToolUse` | Трекинг состояния сессии для shell-based рантаймов |
| `gsd-validate-commit.sh` | `PostToolUse` | Валидация коммитов для форсинга conventional commits |
| `gsd-phase-boundary.sh` | `PostToolUse` | Детекция границ фаз для workflow-переходов |

См. [`docs/INVENTORY.md`](INVENTORY.md#hooks-11-shipped) для авторитетного roster'а 11 хуков.

### SDK Runtime Bridge модуль (`sdk/src/query-runtime-bridge.ts`)

Программные SDK-вызовы (`GSDTools`) роутятся через один seam, владеющий политикой query dispatch:

- Предпочтение native registry dispatch
- Политика явного subprocess fallback (`allowFallbackToSubprocess`)
- Strict SDK режим (`strictSdk`) для fail-fast native-only форсинга
- Структурированная наблюдаемость диспатча (`onDispatchEvent`) с режимом, причиной, продолжительностью и исходом

Это держит вызывающую сторону тонкими адаптерами и централизует решения по транспорту для SDK-публикуемости.

### CLI-инструменты (`get-shit-done/bin/`)

Node.js CLI-утилита (`gsd-tools.cjs`) с доменными модулями, разделёнными по `get-shit-done/bin/lib/` (см. [`docs/INVENTORY.md`](INVENTORY.md#cli-modules-33-shipped) для авторитетного roster'а):

| Модуль                 | Ответственность                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| `core.cjs`             | Обработка ошибок, форматирование вывода, общие утилиты; re-exports совместимости для planning-хелперов |
| `planning-workspace.cjs` | Planning seam (`planningDir`, `planningPaths`, активный workstream-роутинг, `.planning/.lock`)    |
| `state.cjs`            | Парсинг, обновление, прогрессия STATE.md, метрики                                                   |
| `phase.cjs`            | Операции с phase-директориями, десятичная нумерация, индексация планов                              |
| `roadmap.cjs`          | Парсинг ROADMAP.md, извлечение фаз, прогресс планов                                                 |
| `config.cjs`           | Чтение/запись config.json, инициализация секций                                                     |
| `verify.cjs`           | Структура плана, полнота фазы, ссылочная и commit-валидация                                         |
| `template.cjs`         | Выбор и наполнение шаблонов с подстановкой переменных                                               |
| `frontmatter.cjs`      | CRUD-операции YAML frontmatter                                                                       |
| `init.cjs`             | Compound-загрузка контекста для каждого типа workflow                                               |
| `milestone.cjs`        | Архивация майлстоунов, разметка требований                                                          |
| `commands.cjs`         | Misc-команды (slug, timestamp, todos, scaffolding, stats)                                           |
| `model-profiles.cjs`   | Таблица резолва профилей моделей                                                                    |
| `security.cjs`         | Защита от path traversal, детекция prompt injection, безопасный парсинг JSON, валидация shell-аргументов |
| `uat.cjs`              | Парсинг UAT-файлов, трекинг verification debt, поддержка audit-uat                                  |
| `docs.cjs`             | Инициализация docs-update workflow, сканирование Markdown, детекция монорепо                        |
| `workstream.cjs`       | CRUD workstream'ов, миграция, session-scoped активный указатель                                     |
| `schema-detect.cjs`    | Детекция schema-дрейфа для ORM-паттернов (Prisma, Drizzle и т.д.)                                   |
| `profile-pipeline.cjs` | Data-пайплайн поведенческого профайлинга, сканирование session-файлов                                |
| `profile-output.cjs`   | Рендеринг профиля, генерация USER-PROFILE.md и dev-preferences.md                                   |

---

## Модель агентов

### Паттерн Orchestrator → Agent

```
Оркестратор (workflow .md)
    │
    ├── Загрузка контекста: gsd-sdk query init.<workflow> <phase> (или легаси gsd-tools.cjs init)
    │   Возвращает JSON с: project info, config, state, phase details
    │
    ├── Резолв модели: gsd-sdk query resolve-model <agent-name>
    │   Возвращает: opus | sonnet | haiku | inherit
    │
    ├── Спавн агента (вызов Task/SubAgent)
    │   ├── Промпт агента (agents/*.md)
    │   ├── Context payload (init JSON)
    │   ├── Назначение модели
    │   └── Tool permissions
    │
    ├── Сбор результата
    │
    └── Обновление состояния: gsd-sdk query state.update / state.patch / state.advance-plan (или легаси gsd-tools.cjs)
```

### Категории спавна первичных агентов

Концептуальная таксономия паттернов спавна для 21 первичного агента. Авторитетный roster из 31 агента (включая 10 продвинутых/специализированных таких как `gsd-pattern-mapper`, `gsd-code-reviewer`, `gsd-code-fixer`, `gsd-ai-researcher`, `gsd-domain-researcher`, `gsd-eval-planner`, `gsd-eval-auditor`, `gsd-framework-selector`, `gsd-debug-session-manager`, `gsd-intel-updater`) — см. [`docs/INVENTORY.md`](INVENTORY.md#agents-31-shipped).

| Категория       | Агенты                                                                                  | Параллелизм                                                                                |
| --------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Researchers** | gsd-project-researcher, gsd-phase-researcher, gsd-ui-researcher, gsd-advisor-researcher | 4 параллельных (stack, features, architecture, pitfalls); advisor спавнится в discuss-phase |
| **Synthesizers** | gsd-research-synthesizer                                                                | Последовательно (после researcher'ов)                                                       |
| **Planners**    | gsd-planner, gsd-roadmapper                                                             | Последовательно                                                                              |
| **Checkers**    | gsd-plan-checker, gsd-integration-checker, gsd-ui-checker, gsd-nyquist-auditor          | Последовательно (verification-цикл, макс 3 итерации)                                         |
| **Executors**   | gsd-executor                                                                            | Параллельно в волне, последовательно между волнами                                          |
| **Verifiers**   | gsd-verifier                                                                            | Последовательно (после всех исполнителей)                                                    |
| **Mappers**     | gsd-codebase-mapper                                                                     | 4 параллельных (tech, arch, quality, concerns)                                              |
| **Debuggers**   | gsd-debugger                                                                            | Последовательно (интерактивно)                                                               |
| **Auditors**    | gsd-ui-auditor, gsd-security-auditor                                                    | Последовательно                                                                              |
| **Doc Writers** | gsd-doc-writer, gsd-doc-verifier                                                        | Последовательно (writer потом verifier)                                                      |
| **Profilers**   | gsd-user-profiler                                                                       | Последовательно                                                                              |
| **Analyzers**   | gsd-assumptions-analyzer                                                                | Последовательно (во время discuss-phase)                                                     |

### Модель волнового исполнения

Во время `execute-phase` планы группируются в волны по зависимостям:

```
Wave Analysis:
  Plan 01 (no deps)      ─┐
  Plan 02 (no deps)      ─┤── Wave 1 (параллельно)
  Plan 03 (depends: 01)  ─┤── Wave 2 (ждёт Wave 1)
  Plan 04 (depends: 02)  ─┘
  Plan 05 (depends: 03,04) ── Wave 3 (ждёт Wave 2)
```

Каждый executor получает:

- Свежее 200K контекстное окно (или до 1M для моделей, поддерживающих это)
- Конкретный PLAN.md для исполнения
- Контекст проекта (PROJECT.md, STATE.md)
- Контекст фазы (CONTEXT.md, RESEARCH.md если доступен)

### Адаптивное обогащение контекста (1M-модели)

Когда контекстное окно ≥ 500K токенов (1M-class модели вроде Opus 4.6, Sonnet 4.6), промпты sub-агентов автоматически обогащаются дополнительным контекстом, не помещавшимся в стандартных 200K-окнах:

- **Executor-агенты** получают SUMMARY.md предыдущих волн и CONTEXT.md/RESEARCH.md фазы, что позволяет cross-plan awareness в пределах фазы
- **Verifier-агенты** получают все PLAN.md, SUMMARY.md, CONTEXT.md плюс REQUIREMENTS.md, что позволяет history-aware верификацию

Оркестратор читает `context_window` из конфига (`gsd-sdk query config-get context_window` или легаси `gsd-tools.cjs config-get`) и условно включает богатый контекст когда значение ≥ 500 000. Для стандартных 200K-окон промпты используют усечённые версии с cache-friendly порядком для максимизации эффективности контекста.

#### Безопасность параллельных коммитов

Когда несколько executor'ов работают в одной волне, два механизма предотвращают конфликты:

1. `--no-verify` коммиты — Параллельные агенты пропускают pre-commit хуки (которые могут вызывать build lock contention, например cargo lock fights в Rust-проектах). Оркестратор запускает `git hook run pre-commit` один раз после завершения каждой волны.
2. **STATE.md file locking** — Все вызовы `writeStateMd()` используют lockfile-based mutual exclusion (`STATE.md.lock` с атомарным созданием `O_EXCL`). Это предотвращает гонку read-modify-write, где два агента читают STATE.md, модифицируют разные поля, и последний писатель затирает изменения другого. Включает детекцию stale lock (10s таймаут) и spin-wait с jitter'ом.

---

## Поток данных

### Поток нового проекта

```
Пользовательский ввод (описание идеи)
    │
    ▼
Вопросы (философия questioning.md)
    │
    ▼
4x Project Researchers (параллельно)
    ├── Stack → STACK.md
    ├── Features → FEATURES.md
    ├── Architecture → ARCHITECTURE.md
    └── Pitfalls → PITFALLS.md
    │
    ▼
Research Synthesizer → SUMMARY.md
    │
    ▼
Извлечение требований → REQUIREMENTS.md
    │
    ▼
Roadmapper → ROADMAP.md
    │
    ▼
Одобрение пользователем → STATE.md инициализирован
```

### Поток исполнения фазы

```
discuss-phase → CONTEXT.md (предпочтения пользователя)
    │
    ▼
ui-phase → UI-SPEC.md (дизайн-контракт, опционально)
    │
    ▼
plan-phase
    ├── Research gate (блочит если RESEARCH.md имеет неразрешённые open questions)
    ├── Phase Researcher → RESEARCH.md
    │       └── Package Legitimacy Gate: slopcheck на каждый пакет; [SLOP] удалены,
    │           [SUS]/[ASSUMED] помечены; Audit-таблица записана в RESEARCH.md
    ├── Planner (с проверкой достижимости) → PLAN.md файлы
    │       └── checkpoint:human-verify инжектится перед [ASSUMED]/[SUS] install'ами;
    │           строка T-{phase}-SC STRIDE добавляется для install-несущих планов
    ├── Plan Checker → Verify-цикл (макс 3x)
    ├── Requirements coverage gate (REQ-ID → планы)
    └── Decision coverage gate (CONTEXT.md `<decisions>` → планы, БЛОКИРУЮЩИЙ — #2492)
    │
    ▼
state planned-phase → STATE.md (Planned/Ready to execute)
    │
    ▼
execute-phase (редукция контекста: усечённые промпты, cache-friendly порядок)
    ├── Wave analysis (группировка по зависимостям)
    ├── Executor на план → код + атомарные коммиты
    ├── SUMMARY.md на план
    └── Verifier → VERIFICATION.md
        └── Decision coverage gate (решения CONTEXT.md → зашиппленные артефакты, НЕ-БЛОКИРУЮЩИЙ — #2492)
    │
    ▼
verify-work → UAT.md (user acceptance testing)
    │
    ▼
ui-review → UI-REVIEW.md (визуальный аудит, опционально)
```

### Распространение контекста

Каждая стадия workflow производит артефакты, питающие последующие стадии:

```
PROJECT.md ────────────────────────────────────────────► Все агенты
REQUIREMENTS.md ───────────────────────────────────────► Planner, Verifier, Auditor
ROADMAP.md ────────────────────────────────────────────► Оркестраторы
STATE.md ──────────────────────────────────────────────► Все агенты (решения, блокеры)
CONTEXT.md (на фазу) ──────────────────────────────────► Researcher, Planner, Executor
RESEARCH.md (на фазу) ─────────────────────────────────► Planner, Plan Checker
PLAN.md (на план) ─────────────────────────────────────► Executor, Plan Checker
SUMMARY.md (на план) ──────────────────────────────────► Verifier, State tracking
UI-SPEC.md (на фазу) ──────────────────────────────────► Executor, UI Auditor
```

---

## Раскладка файловой системы

### Установочные файлы

```
~/.claude/                          # Claude Code (глобальная установка)
├── skills/gsd-*/SKILL.md           # Глобальные скиллы (авторитетный roster: docs/INVENTORY.md)
├── commands/gsd/*.md               # Локальные Claude-установки используют slash-команды вместо глобальных скиллов
├── get-shit-done/
│   ├── bin/gsd-tools.cjs           # CLI-утилита
│   ├── bin/lib/*.cjs               # Доменные модули (авторитетный roster: docs/INVENTORY.md)
│   ├── workflows/*.md              # Определения workflow (авторитетный roster: docs/INVENTORY.md)
│   ├── references/*.md             # Общие reference-документы (авторитетный roster: docs/INVENTORY.md)
│   └── templates/                  # Шаблоны планировочных артефактов
├── agents/*.md                     # Определения агентов (авторитетный roster: docs/INVENTORY.md)
├── hooks/*.js                      # Node.js хуки (statusline, guards, monitors, update check)
├── hooks/*.sh                      # Shell-хуки (session state, commit validation, phase boundary)
├── settings.json                   # Регистрации хуков
└── VERSION                         # Установленный номер версии
```

Эквивалентные пути для других рантаймов:

- **OpenCode:** `~/.config/opencode/` глобально или `./.opencode/` локально
- **Kilo:** `~/.config/kilo/` глобально или `./.kilo/` локально
- **Gemini CLI:** `~/.gemini/` глобально или `./.gemini/` локально
- **Codex:** `~/.codex/` глобально или `./.codex/` локально
- **Copilot:** `~/.copilot/` глобально или `./.github/` локально
- **Antigravity:** `~/.gemini/antigravity/` глобально или `./.agent/` локально
- **Cursor:** `~/.cursor/` глобально или `./.cursor/` локально
- **Windsurf:** `~/.codeium/windsurf/` глобально или `./.windsurf/` локально
- **Augment Code:** `~/.augment/` глобально или `./.augment/` локально
- **Trae:** `~/.trae/` глобально или `./.trae/` локально
- **Qwen Code:** `~/.qwen/` глобально или `./.qwen/` локально
- **Hermes Agent:** `~/.hermes/` глобально или `./.hermes/` локально
- **CodeBuddy:** `~/.codebuddy/` глобально или `./.codebuddy/` локально
- **Cline:** `~/.cline/` глобально или project-root `.clinerules` локально

### Файлы проекта (`.planning/`)

```
.planning/
├── PROJECT.md              # Видение проекта, ограничения, решения, правила эволюции
├── REQUIREMENTS.md         # Скоупленные требования (v1/v2/out-of-scope)
├── ROADMAP.md              # Разбивка по фазам с отслеживанием статуса
├── STATE.md                # Живая память: позиция, решения, блокеры, метрики
├── config.json             # Конфигурация workflow
├── MILESTONES.md           # Архив завершённых майлстоунов
├── research/               # Domain research от /gsd-new-project
│   ├── SUMMARY.md
│   ├── STACK.md
│   ├── FEATURES.md
│   ├── ARCHITECTURE.md
│   └── PITFALLS.md
├── codebase/               # Brownfield mapping (от /gsd-map-codebase)
│   ├── STACK.md            # YAML frontmatter несёт `last_mapped_commit`
│   ├── ARCHITECTURE.md     # для post-execute дрейф-gate (#2003)
│   ├── CONVENTIONS.md
│   ├── CONCERNS.md
│   ├── STRUCTURE.md
│   ├── TESTING.md
│   └── INTEGRATIONS.md
├── phases/
│   └── XX-phase-name/
│       ├── XX-CONTEXT.md       # Предпочтения пользователя (от discuss-phase)
│       ├── XX-RESEARCH.md      # Исследование экосистемы (от plan-phase)
│       ├── XX-YY-PLAN.md       # Планы исполнения
│       ├── XX-YY-SUMMARY.md    # Outcome'ы исполнения
│       ├── XX-VERIFICATION.md  # Постфактум-верификация
│       ├── XX-VALIDATION.md    # Маппинг тест-покрытия Nyquist
│       ├── XX-UI-SPEC.md       # UI дизайн-контракт (от ui-phase)
│       ├── XX-UI-REVIEW.md     # Оценки визуального аудита (от ui-review)
│       └── XX-UAT.md           # Результаты UAT
├── quick/                  # Трекинг quick-задач
│   └── YYMMDD-xxx-slug/
│       ├── PLAN.md
│       └── SUMMARY.md
├── todos/
│   ├── pending/            # Захваченные идеи
│   └── done/               # Завершённые todo
├── threads/               # Persistent контекстные треды (от /gsd-thread)
├── seeds/                 # Forward-looking идеи (от /gsd-capture --seed)
├── debug/                  # Активные debug-сессии
│   ├── *.md                # Активные сессии
│   ├── resolved/           # Архивные сессии
│   └── knowledge-base.md   # Persistent debug-уроки
├── ui-reviews/             # Скриншоты от /gsd-ui-review (gitignored)
└── continue-here.md        # Context handoff (от pause-work)
```

### Post-execute дрейф-gate кодовой базы (#2003)

После последней волны коммитов `/gsd-execute-phase` workflow запускает не-блокирующий шаг `codebase_drift_gate` (между `schema_drift_gate` и `verify_phase_goal`). Он сравнивает diff `last_mapped_commit..HEAD` против `.planning/codebase/STRUCTURE.md` и считает четыре типа структурных элементов:

1. Новые директории вне отмаппленных путей
2. Новые barrel-экспорты в `(packages|apps)/<name>/src/index.*`
3. Новые migration-файлы
4. Новые route-модули под `routes/` или `api/`

Если счёт достигает `workflow.drift_threshold` (дефолт 3), gate либо **предупреждает** (дефолт) предложенной командой `/gsd-map-codebase --paths …`, либо **авто-remap**ит (`workflow.drift_action = auto-remap`), спавня `gsd-codebase-mapper`, скоупленный на затронутые пути. Любая ошибка детекции или remap'а логируется и фаза продолжается — дрейф-детекция не может провалить верификацию.

`last_mapped_commit` живёт в YAML frontmatter в начале каждого файла `.planning/codebase/*.md`; `bin/lib/drift.cjs` предоставляет round-trip хелперы `readMappedCommit` и `writeMappedCommit`.

---

## Архитектура установщика

Установщик (`bin/install.js`, ~10 700 строк) обрабатывает:

1. **Детекция рантайма** — Интерактивный промпт или CLI-флаги (`--claude`, `--opencode`, `--gemini`, `--kilo`, `--codex`, `--copilot`, `--antigravity`, `--cursor`, `--windsurf`, `--augment`, `--trae`, `--qwen`, `--hermes`, `--codebuddy`, `--cline`, `--all`)
2. **Выбор расположения** — Глобально (`--global`) или локально (`--local`)
3. **Деплой файлов** — Копирует команды, скиллы, workflow, references, шаблоны, агенты и хуки
4. **Runtime-адаптация** — Трансформирует содержимое под рантайм:
   - Claude Code: Использует как есть
   - OpenCode: Конвертит команды/агентов в OpenCode-совместимый flat command + subagent формат
   - Kilo: Переиспользует pipeline конверсии OpenCode с Kilo-путями конфига
   - Codex: Генерит TOML-конфиг + скиллы из команд
   - Copilot: Маппит имена инструментов (Read→read, Bash→execute и т.д.)
   - Gemini: Подстраивает имена событий хуков (`AfterTool` вместо `PostToolUse`)
   - Antigravity: Skills-first с эквивалентами Google-моделей
   - Cursor: Skills-first с Cursor rule references
   - Windsurf: Skills-first с Windsurf rule references
   - Trae: Skills-first установка в `~/.trae` / `./.trae` без `settings.json` или хук-интеграции
   - Qwen Code: Skills-first с Qwen-брендованным путём и переписыванием промптов
   - Hermes Agent: Category-based скиллы под `skills/gsd/`
   - CodeBuddy: Skills-first с CodeBuddy-путём и переписыванием промптов
   - Cline: Пишет `.clinerules` для rule-based интеграции
   - Augment Code: Skills-first с полной конверсией скиллов и управлением конфигом
5. **Нормализация путей** — Заменяет `~/.claude/` пути на runtime-специфичные
6. **Интеграция настроек** — Регистрирует хуки в `settings.json` рантайма
7. **Patch-бекап** — С v1.17 бэкапит локально модифицированные файлы в `gsd-local-patches/` для `/gsd-update --reapply`
8. **Manifest tracking** — Пишет `gsd-file-manifest.json` для чистого uninstall
9. **Uninstall режим** — `--uninstall` удаляет все GSD-файлы, хуки и настройки

Установочные перемещения файлов, очистка устаревших артефактов, переписывание конфигов и сохранение пользовательских данных управляются модулем миграции установщика. См. [Installer Migrations](installer-migrations.md) и [ADR 0008](adr/0008-installer-migration-module.md).

Migration-модуль также владеет гейтнутым first-time baseline-сканом для legacy-установок, классифицируя известные runtime-инсталляционные поверхности до того как поздние миграции что-либо удалят или перепишут.

### Обработка платформ

- **Windows:** `windowsHide` на child-процессах, защита EPERM/EACCES на защищённых директориях, нормализация path-сепараторов
- **WSL:** Детектит Windows Node.js, работающий на WSL, и предупреждает о несовпадении путей
- **Docker/CI:** Поддерживает env-переменную `CLAUDE_CONFIG_DIR` для кастомных расположений конфиг-директории

---

## Система хуков

### Архитектура

```
Runtime Engine (Claude Code / Gemini CLI)
    │
    ├── statusLine event ──► gsd-statusline.js
    │   Читает: stdin (session JSON)
    │   Пишет: stdout (форматированный статус), /tmp/claude-ctx-{session}.json (мост)
    │
    ├── PostToolUse/AfterTool event ──► gsd-context-monitor.js
    │   Читает: stdin (tool event JSON), /tmp/claude-ctx-{session}.json (мост)
    │   Пишет: stdout (hookSpecificOutput с additionalContext предупреждением)
    │
    └── SessionStart event ──► gsd-check-update.js
        Читает: VERSION файл
        Пишет: ~/.claude/cache/gsd-update-check.json (спавнит фоновый процесс)
```

### Пороги Context Monitor

| Остаток контекста | Уровень  | Поведение агента                       |
| ----------------- | -------- | -------------------------------------- |
| > 35%             | Normal   | Предупреждение не инжектится           |
| ≤ 35%             | WARNING  | "Avoid starting new complex work"      |
| ≤ 25%             | CRITICAL | "Context nearly exhausted, inform user" |

Дебаунс: 5 tool-вызовов между повторными предупреждениями. Эскалация severity (WARNING→CRITICAL) обходит дебаунс.

### Свойства безопасности

- Все хуки обёрнуты в try/catch, при ошибке выходят молча
- Stdin timeout guard (3s) предотвращает зависания на pipe-проблемах
- Stale-метрики (>60s старые) игнорируются
- Отсутствующие bridge-файлы обрабатываются gracefully (sub-агенты, свежие сессии)
- Context monitor — advisory; никогда не выдаёт императивные команды, переопределяющие предпочтения пользователя

### Package Legitimacy Gate (v1.51)

Пайплайн researcher → planner → executor включает supply-chain gate против slopsquatting'а (AI-галлюцинированные имена пакетов, преднамеренно зарегистрированные со зловредными post-install скриптами).

**Модель угрозы:** GSD автоматизирует полный путь от «researcher называет пакет» до «executor запускает `npm install`». Галлюцинированное имя, проходящее `npm view` (доказывая только регистрацию, не легитимность), раньше прошло бы незамеченным. ~20% AI-генерируемых package-ссылок галлюцинированы; ~43% этих имён повторяются последовательно между промптами, делая пред-регистрацию экономически выгодной для атакующих.

**Слои gate'а:**

| Слой | Компонент | Действие |
|------|-----------|----------|
| Research | `gsd-phase-researcher` | Запускает `slopcheck install <pkgs> --json`; пишет таблицу `## Package Legitimacy Audit` в RESEARCH.md; убирает `[SLOP]` пакеты до записи RESEARCH.md |
| Planning | `gsd-planner` | Читает Audit-таблицу; вставляет `checkpoint:human-verify` перед любой `[ASSUMED]` или `[SUS]` install-задачей; добавляет строку `T-{phase}-SC` STRIDE supply-chain в `<threat_model>` |
| Execution | `gsd-executor` | RULE 3 исключает установку пакетов из auto-fix scope; упавшие install'ы всплывают как чекпоинты, никогда не молчаливые подмены |

**Интеграция claim-провенанса:** Имена пакетов, найденные через WebSearch, тегаются `[ASSUMED]` (не `[VERIFIED]`) независимо от результата `npm view`. Это расширяет существующую систему провенанса `[ASSUMED]` / `[VERIFIED]` / `[CITED]`, форся provenance-тег как hard-gate на install-границе — `[ASSUMED]` всегда генерит `checkpoint:human-verify` в PLAN.md.

**Покрытие экосистем:** Researcher использует registry-специфичные команды верификации — `npm view` (Node), `pip index versions` (Python), `cargo search` (Rust) — а не один generic-чек. Это ловит кросс-экосистемную галлюцинацию (~9% rate задокументировано в исследовании USENIX 2025).

**Graceful degradation:** Если `slopcheck` недоступен, каждый рекомендованный пакет тегается `[ASSUMED]` и гейтится чекпоинтом. Research и планирование продолжаются; система никогда не hard-fail'ит на отсутствующей tool-зависимости.

**Внешняя зависимость:** `slopcheck` (MIT, pip-installable). Если будет брошен, `[ASSUMED]`-gate fallback поддерживает human-checkpoint покрытие.

---

### Security-хуки (v1.27)

**Prompt Guard** (`gsd-prompt-guard.js`):

- Триггерится на Write/Edit в `.planning/` файлы
- Сканит контент на паттерны prompt injection (role override, instruction bypass, system tag injection)
- Advisory-only — логирует детекцию, не блокирует
- Паттерны заинлайнены (subset `security.cjs`) для независимости хука

**Workflow Guard** (`gsd-workflow-guard.js`):

- Триггерится на Write/Edit в не-`.planning/` файлы
- Детектит правки вне GSD workflow-контекста (нет активной `/gsd-` команды или Task sub-агента)
- Советует использовать `/gsd-quick` или `/gsd-fast` для state-отслеживаемых изменений
- Opt-in через `hooks.workflow_guard: true` (дефолт: false)

---

## Абстракция рантайма

GSD поддерживает несколько AI coding-рантаймов через унифицированную command/workflow архитектуру:

### Матрица install-контрактов рантаймов

Матрица описывает поверхности рантаймов, которые установщик материализует сегодня. Migration-specific владение и source-снапшоты живут в [Installer Migrations](installer-migrations.md#runtime-configuration-contract-registry).

| Рантайм | Глобальный root | Локальный root | Invocation surface | Agent surface | Конфиг и хуки |
| --- | --- | --- | --- | --- | --- |
| Claude Code | `~/.claude` | `./.claude` | Глобально `skills/gsd-*/SKILL.md`; локально `commands/gsd/*.md` | `agents/gsd-*.md` | Записи hook и statusLine в `settings.json` |
| OpenCode | `~/.config/opencode` | `./.opencode` | `command/gsd-*.md` | `agents/gsd-*.md` | `opencode.json` или `opencode.jsonc`; без GSD-хуков |
| Kilo | `~/.config/kilo` | `./.kilo` | `command/gsd-*.md` | `agents/gsd-*.md` | `kilo.json` или `kilo.jsonc`; без GSD-хуков |
| Gemini CLI | `~/.gemini` | `./.gemini` | `commands/gsd/*.toml` | `agents/gsd-*.md` | Feature flag, хуки и statusline в `settings.json` |
| Codex | `~/.codex` | `./.codex` | `skills/gsd-*/SKILL.md` | `agents/` source markdown + per-agent TOML | `config.toml` `[agents.gsd-*]`, `[features].codex_hooks` и hook-таблицы |
| GitHub Copilot | `~/.copilot` | `./.github` | `skills/gsd-*/SKILL.md` и `copilot-instructions.md` | `.agent.md` файлы | Без GSD-хуков или statusline |
| Antigravity | `~/.gemini/antigravity` | `./.agent` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Gemini-style записи хуков в `settings.json` когда установлено GSD |
| Cursor | `~/.cursor` | `./.cursor` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Rule references под `rules/`; без GSD-хуков |
| Windsurf | `~/.codeium/windsurf` | `./.windsurf` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Rule references под `rules/`; без GSD-хуков |
| Augment Code | `~/.augment` | `./.augment` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Без GSD-хуков или statusline |
| Trae | `~/.trae` | `./.trae` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Rule references под `rules/`; без GSD-хуков |
| Qwen Code | `~/.qwen` | `./.qwen` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Общие GSD-настройки и хуки где поддерживается |
| Hermes Agent | `~/.hermes` | `./.hermes` | `skills/gsd/DESCRIPTION.md` + `skills/gsd/gsd-*/SKILL.md` | `agents/gsd-*.md` | Общие GSD-настройки и хуки где поддерживается |
| CodeBuddy | `~/.codebuddy` | `./.codebuddy` | `skills/gsd-*/SKILL.md` | `agents/gsd-*.md` | Общие GSD-настройки и хуки где поддерживается |
| Cline | `~/.cline` | project root | `.clinerules` | Только правила | Без GSD-хуков или statusline |

### Upstream-контракт: источники

Ожидания установки рантаймов сверяются с первичной документацией где она доступна. Текущий source-snapshot — 2026-05-11:

- Claude Code: Anthropic slash commands, settings, hooks и subagents docs
- OpenCode и Kilo: docs конфига OpenCode и docs кастомных sub-агентов Kilo
- Gemini CLI и Qwen Code: docs команд/конфига; docs команд Qwen последний раз обновлены 2026-05-06
- Codex: OpenAI Codex docs и `config-schema.json`; установщик также несёт совместимость Codex 0.124.0 для формы agent-таблицы
- Copilot, Cursor, Cline, Augment, Hermes и CodeBuddy: вендор-доки для кастомных инструкций, правил, скиллов или конфига
- Antigravity, Windsurf и Trae: строки с ограниченными источниками. Установщик документирует текущие compat-shim'ы, миграции должны освежить эти источники до переписывания их конфига

### Точки абстракции

1. **Маппинг имён инструментов** — У каждого рантайма свои имена инструментов (например, Claude `Bash` → Copilot `execute`)
2. **Имена событий хуков** — Claude использует `PostToolUse`, Gemini — `AfterTool`
3. **Frontmatter агента** — У каждого рантайма свой формат определения агента
4. **Конвенции путей** — Каждый рантайм хранит конфиг в разных директориях
5. **Ссылки на модели** — Профиль `inherit` позволяет GSD откладывать выбор модели на рантайм

Установщик обрабатывает всю трансляцию во время установки. Workflow и агенты написаны в нативном формате Claude Code и трансформируются при деплое.
