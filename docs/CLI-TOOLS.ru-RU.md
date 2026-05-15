# GSD — Справочник CLI-инструментов

> Surface-area справочник по `get-shit-done/bin/gsd-tools.cjs` (легаси Node CLI). Workflow'ы и агенты должны предпочитать `gsd-sdk query` или `@gsd-build/sdk` где есть handler — см. [SDK и программный доступ](#sdk-и-программный-доступ). Slash-команды и пользовательские потоки — в [Command Reference](COMMANDS.ru-RU.md).

---

## Обзор

`gsd-tools.cjs` централизует парсинг конфига, резолв моделей, поиск фаз, git-коммиты, верификацию summary, управление состоянием и операции с шаблонами для команд, workflow'ов и агентов GSD.

|                       |                                                                                                                                                                                                        |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Поставочный путь**  | `get-shit-done/bin/gsd-tools.cjs`                                                                                                                                                                       |
| **Реализация**        | 20 доменных модулей под `get-shit-done/bin/lib/` (директория авторитетная)                                                                                                                              |
| **Статус**            | Поддерживается для тестов паритета и CJS-only entrypoint'ов; `gsd-sdk query` / SDK-реестр — поддерживаемый путь для новой оркестрации (см. [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md)). |

**Использование (CJS):**

```bash
node gsd-tools.cjs <command> [args] [--raw] [--cwd <path>]
```

**Глобальные флаги (CJS):**

| Флаг           | Описание                                                                          |
| -------------- | --------------------------------------------------------------------------------- |
| `--raw`        | Machine-readable вывод (JSON или plain text, без форматирования)                  |
| `--cwd <path>` | Override рабочей директории (для sandbox'енных sub-агентов)                       |
| `--ws <name>`  | Workstream-контекст (также соблюдается когда SDK спавнит этот бинарь; см. ниже)   |

---

## SDK и программный доступ

Используй когда пишешь workflow'ы, не когда нужен только листинг команд ниже.

**1. CLI — `gsd-sdk query <argv…>`**

- Резолвит argv по тем же правилам **longest-prefix** что и типизированный реестр (`resolveQueryArgv` в `sdk/src/query/registry.ts`). Незарегистрированные команды **падают быстро** — используй `node …/gsd-tools.cjs` только для хендлеров, отсутствующих в реестре.
- Полная матрица (CJS-команда → ключ реестра, CLI-only инструменты, алиасы, golden-tier'ы): [sdk/src/query/QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md).

**2. TypeScript — `@gsd-build/sdk` (`GSDTools`, `createRegistry`)**

- `GSDTools` теперь роутится через **SDK Runtime Bridge модуль** (`sdk/src/query-runtime-bridge.ts`). Native registry dispatch предпочитаемый; subprocess fallback — явная политика (`allowFallbackToSubprocess`), его можно отключить для строгого SDK-only исполнения.
- Режим `strictSdk` fail'ит быстро когда у команды нет native-адаптера, делая SDK publish/readiness-проверки детерминированными.
- Структурированная наблюдаемость моста доступна через `onDispatchEvent` (режим диспатча, причина fallback'а, длительность, исход, тип ошибки).
- Для прямого типизированного диспатча без `GSDTools` используй `createRegistry()` из `sdk/src/query/index.ts` или вызывай `gsd-sdk query` (см. [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md)).
- Конвенции: проводка mutation-событий, `GSDError` vs `{ data: { error } }`, локи и stub'ы — [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md).

**Примеры CJS → SDK (одна и та же директория проекта):**

| Легаси CJS                                | Предпочтительный `gsd-sdk query` (примеры) |
| ----------------------------------------- | ------------------------------------------ |
| `node gsd-tools.cjs init phase-op 12`     | `gsd-sdk query init phase-op 12`           |
| `node gsd-tools.cjs phase-plan-index 12`  | `gsd-sdk query phase-plan-index 12`        |
| `node gsd-tools.cjs state json`           | `gsd-sdk query state json`                 |
| `node gsd-tools.cjs roadmap analyze`      | `gsd-sdk query roadmap analyze`            |

**Чтения state через SDK:** `state.json` и `state.load` — оба зарегистрированные query-хендлера с паритет-покрытием. Их можно вызвать через `gsd-sdk query …` и через SDK Runtime Bridge (`GSDTools` → `sdk/src/query-runtime-bridge.ts`), соблюдая `allowFallbackToSubprocess` / `strictSdk` и эмитя `onDispatchEvent`-наблюдаемость. Для прямого типизированного диспатча используй `createRegistry()` из `sdk/src/query/index.ts`. Полный роутинг и golden-правила: [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md).

**Только CLI (не в реестре):** например **graphify**, **from-gsd2** / **gsd2-import** — вызывай `gsd-tools.cjs` пока не зарегистрировано.

**Mutation-события (SDK):** `QUERY_MUTATION_COMMANDS` в `sdk/src/query/index.ts` перечисляет команды, которые могут эмитить структурированные события после успешного диспатча. Исключения отмечены в QUERY-HANDLERS: `state validate` (read-only), `skill-manifest` (пишет только с `--write`), `intel update` (stub).

**Golden parity:** Политика и категории CJS↔SDK тестов задокументированы под **Golden parity** в [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md).

---

## State-команды

Управление `.planning/STATE.md` — живой памятью проекта.

```bash
# Загрузить полный project-config + state как JSON
node gsd-tools.cjs state load

# Вывести frontmatter STATE.md как JSON
node gsd-tools.cjs state json

# Обновить одно поле
node gsd-tools.cjs state update <field> <value>

# Получить содержимое STATE.md или конкретную секцию
node gsd-tools.cjs state get [section]

# Batch-обновление нескольких полей
node gsd-tools.cjs state patch --field1 val1 --field2 val2

# Инкрементить счётчик плана
node gsd-tools.cjs state advance-plan

# Записать метрики исполнения
node gsd-tools.cjs state record-metric --phase N --plan M --duration Xmin [--tasks N] [--files N]

# Пересчитать progress-бар
node gsd-tools.cjs state update-progress

# Добавить решение
node gsd-tools.cjs state add-decision --summary "..." [--phase N] [--rationale "..."]
# Или из файлов:
node gsd-tools.cjs state add-decision --summary-file path [--rationale-file path]

# Добавить/разрешить блокеры
node gsd-tools.cjs state add-blocker --text "..."
node gsd-tools.cjs state resolve-blocker --text "..."

# Записать continuity сессии
node gsd-tools.cjs state record-session --stopped-at "..." [--resume-file path]

# Начало фазы — обновить Status/Last activity в STATE.md под новую фазу
node gsd-tools.cjs state begin-phase --phase N --name SLUG --plans COUNT

# Agent-discoverable сигнал блокера (используется discuss-phase / UI-потоками)
node gsd-tools.cjs state signal-waiting --type TYPE --question "..." --options "A|B" --phase P
node gsd-tools.cjs state signal-resume
```

### State Snapshot

Структурированный парс полного STATE.md:

```bash
node gsd-tools.cjs state-snapshot
```

Возвращает JSON с: текущей позицией, фазой, планом, статусом, решениями, блокерами, метриками, последней активностью.

---

## Phase-команды

Управление фазами — директории, нумерация, синк с roadmap'ом.

```bash
# Найти phase-директорию по номеру
node gsd-tools.cjs find-phase <phase>

# Вычислить следующий десятичный номер фазы для вставок
node gsd-tools.cjs phase next-decimal <phase>

# Дописать новую фазу в roadmap + создать директорию
node gsd-tools.cjs phase add <description>

# Вставить десятичную фазу после существующей
node gsd-tools.cjs phase insert <after> <description>

# Удалить фазу, перенумеровать последующие
node gsd-tools.cjs phase remove <phase> [--force]

# Пометить фазу завершённой, обновить state + roadmap
node gsd-tools.cjs phase complete <phase>

# Индексировать планы с волнами и статусом
node gsd-tools.cjs phase-plan-index <phase>

# Листинг фаз с фильтрацией
node gsd-tools.cjs phases list [--type planned|executed|all] [--phase N] [--include-archived]
```

---

## Roadmap-команды

Парсинг и обновление `ROADMAP.md`.

```bash
# Извлечь секцию фазы из ROADMAP.md
node gsd-tools.cjs roadmap get-phase <phase>

# Полный парс roadmap'а со статусом с диска
node gsd-tools.cjs roadmap analyze

# Обновить строку progress-таблицы с диска
node gsd-tools.cjs roadmap update-plan-progress <N>
```

---

## Config-команды

Чтение и запись `.planning/config.json`.

```bash
# Инициализировать config.json дефолтами
node gsd-tools.cjs config-ensure-section

# Установить значение конфига (dot notation)
node gsd-tools.cjs config-set <key> <value>

# Получить значение конфига
node gsd-tools.cjs config-get <key>

# Установить профиль модели
node gsd-tools.cjs config-set-model-profile <profile>
```

---

## Резолв моделей

```bash
# Получить модель для агента на основе текущего профиля
node gsd-tools.cjs resolve-model <agent-name>
# Возвращает: opus | sonnet | haiku | inherit
```

Имена агентов: `gsd-planner`, `gsd-executor`, `gsd-phase-researcher`, `gsd-project-researcher`, `gsd-research-synthesizer`, `gsd-verifier`, `gsd-plan-checker`, `gsd-integration-checker`, `gsd-roadmapper`, `gsd-debugger`, `gsd-codebase-mapper`, `gsd-nyquist-auditor`

---

## Verification-команды

Валидировать планы, фазы, ссылки и коммиты.

```bash
# Верифицировать SUMMARY.md-файл
node gsd-tools.cjs verify-summary <path> [--check-count N]

# Проверить структуру PLAN.md + задачи
node gsd-tools.cjs verify plan-structure <file>

# Проверить, что у всех планов есть summary
node gsd-tools.cjs verify phase-completeness <phase>

# Проверить, что @-refs + пути резолвятся
node gsd-tools.cjs verify references <file>

# Batch-верификация commit-хешей
node gsd-tools.cjs verify commits <hash1> [hash2] ...

# Проверить must_haves.artifacts
node gsd-tools.cjs verify artifacts <plan-file>

# Проверить must_haves.key_links
node gsd-tools.cjs verify key-links <plan-file>
```

---

## Validation-команды

Проверка целостности проекта.

```bash
# Проверить нумерацию фаз, синк диск/roadmap
node gsd-tools.cjs validate consistency

# Проверить целостность .planning/, опционально починить
node gsd-tools.cjs validate health [--repair]

# Probe утилизации контекстного окна для status-line / hook caller'ов (v1.40.0)
node gsd-tools.cjs validate context
```

`validate context` эмитит структурированный envelope с `utilization`, `status` (`ok` / `warn` / `critical` на порогах 60 % / 70 %) и строкой `suggestion`. Те же данные стоят за `/gsd-health --context`.

---

## Template-команды

Выбор и наполнение шаблонов.

```bash
# Выбрать summary-шаблон на основе гранулярности
node gsd-tools.cjs template select <type>

# Наполнить шаблон переменными
node gsd-tools.cjs template fill <type> --phase N [--plan M] [--name "..."] [--type execute|tdd] [--wave N] [--fields '{json}']
```

Типы шаблонов для `fill`: `summary`, `plan`, `verification`

---

## Frontmatter-команды

CRUD-операции YAML frontmatter на любом Markdown-файле.

```bash
# Извлечь frontmatter как JSON
node gsd-tools.cjs frontmatter get <file> [--field key]

# Обновить одно поле
node gsd-tools.cjs frontmatter set <file> --field key --value jsonVal

# Замерджить JSON в frontmatter
node gsd-tools.cjs frontmatter merge <file> --data '{json}'

# Валидировать обязательные поля
node gsd-tools.cjs frontmatter validate <file> --schema plan|summary|verification
```

---

## Scaffold-команды

Создание пред-структурированных файлов и директорий.

```bash
# Создать CONTEXT.md по шаблону
node gsd-tools.cjs scaffold context --phase N

# Создать UAT.md по шаблону
node gsd-tools.cjs scaffold uat --phase N

# Создать VERIFICATION.md по шаблону
node gsd-tools.cjs scaffold verification --phase N

# Создать phase-директорию
node gsd-tools.cjs scaffold phase-dir --phase N --name "phase name"
```

---

## Init-команды (compound-загрузка контекста)

Загрузить весь контекст, нужный для конкретного workflow, одним вызовом. Возвращает JSON с project info, config, state и workflow-specific данными.

```bash
node gsd-tools.cjs init execute-phase <phase>
node gsd-tools.cjs init plan-phase <phase>
node gsd-tools.cjs init new-project
node gsd-tools.cjs init new-milestone
node gsd-tools.cjs init quick <description>
node gsd-tools.cjs init resume
node gsd-tools.cjs init verify-work <phase>
node gsd-tools.cjs init phase-op <phase>
node gsd-tools.cjs init todos [area]
node gsd-tools.cjs init milestone-op
node gsd-tools.cjs init map-codebase
node gsd-tools.cjs init progress

# Workstream-scoped init (SDK-флаг --ws)
node gsd-tools.cjs init execute-phase <phase> --ws <name>
node gsd-tools.cjs init plan-phase <phase> --ws <name>
```

**Обработка больших payload'ов:** Когда вывод превышает ~50KB, CLI пишет в temp-файл и возвращает `@file:/tmp/gsd-init-XXXXX.json`. Workflow'ы проверяют префикс `@file:` и читают с диска:

```bash
INIT=$(node gsd-tools.cjs init execute-phase "1")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

---

## Milestone-команды

```bash
# Архивировать майлстоун
node gsd-tools.cjs milestone complete <version> [--name <name>] [--archive-phases]

# Пометить требования как завершённые
node gsd-tools.cjs requirements mark-complete <ids>
# Принимает: REQ-01,REQ-02 или REQ-01 REQ-02 или [REQ-01, REQ-02]
```

---

## Skill Manifest

Пред-вычислить и кешировать skill-discovery для более быстрой загрузки команд.

```bash
# Сгенерировать skill-манифест (пишет в .claude/skill-manifest.json)
node gsd-tools.cjs skill-manifest

# Сгенерировать с кастомным output-путём
node gsd-tools.cjs skill-manifest --output <path>
```

Возвращает JSON-маппинг всех доступных GSD-скиллов с их метаданными (name, description, file path, argument hints). Используется установщиком и session-start хуками для избежания повторных сканов ФС.

---

## Утилитарные команды

```bash
# Конвертировать текст в URL-safe slug
node gsd-tools.cjs generate-slug "Some Text Here"
# → some-text-here

# Получить timestamp
node gsd-tools.cjs current-timestamp [full|date|filename]

# Посчитать и перечислить висящие todo
node gsd-tools.cjs list-todos [area]

# Проверить существование файла/директории
node gsd-tools.cjs verify-path-exists <path>

# Аггрегировать все SUMMARY.md
node gsd-tools.cjs history-digest

# Извлечь структурированные данные из SUMMARY.md
node gsd-tools.cjs summary-extract <path> [--fields field1,field2]

# Статистика проекта
node gsd-tools.cjs stats [json|table]

# Рендеринг прогресса
node gsd-tools.cjs progress [json|table|bar]

# Завершить todo
node gsd-tools.cjs todo complete <filename>

# UAT-аудит — сканировать все фазы на неразрешённые айтемы
node gsd-tools.cjs audit-uat

# Cross-artifact audit-очередь — сканить `.planning/` на неразрешённые audit-айтемы
node gsd-tools.cjs audit-open [--json]

# Обратная миграция GSD-2 проекта в текущую структуру (стоит за `/gsd-import --from-gsd2`)
node gsd-tools.cjs from-gsd2 [--path <dir>] [--force] [--dry-run]

# Git-коммит с проверками конфига
node gsd-tools.cjs commit <message> [--files f1 f2] [--amend] [--no-verify]
```

> `--no-verify`: Пропускает pre-commit хуки. Используется параллельными executor-агентами во время волнового исполнения чтобы избежать build lock contention (например, cargo lock fights в Rust-проектах). Оркестратор запускает хуки один раз после завершения каждой волны. Не используй `--no-verify` во время последовательного исполнения — пусть хуки работают нормально.

```bash
# Web search (требует Brave API-ключ)
node gsd-tools.cjs websearch <query> [--limit N] [--freshness day|week|month]
```

---

## Graphify

Строить, query'ить и инспектить граф знаний проекта в `.planning/graphs/`. Требует `graphify.enabled: true` в `config.json` (см. [Configuration Reference](CONFIGURATION.md#graphify-settings)). Graphify — **CJS-only**: `gsd-sdk query` пока не регистрирует graphify-хендлеры — всегда используй `node gsd-tools.cjs graphify …`.

```bash
# Построить или перестроить граф знаний
node gsd-tools.cjs graphify build

# Поиск по графу по термину
node gsd-tools.cjs graphify query <term>

# Свежесть и статистика графа
node gsd-tools.cjs graphify status

# Изменения с последней сборки
node gsd-tools.cjs graphify diff

# Записать именованный snapshot текущего графа
node gsd-tools.cjs graphify snapshot [name]
```

Пользовательская точка входа: `/gsd-graphify` (см. [Command Reference](COMMANDS.ru-RU.md)).

---

## Архитектура модулей

| Модуль | Файл | Экспорты |
|--------|------|----------|
| Core | `lib/core.cjs` | `error()`, `output()`, `parseArgs()`, общие утилиты, compat re-exports |
| State | `lib/state.cjs` | Все подкоманды `state`, `state-snapshot` |
| Phase | `lib/phase.cjs` | CRUD фаз, `find-phase`, `phase-plan-index`, `phases list` |
| Planning Workspace | `lib/planning-workspace.cjs` | Planning seam: `planningDir`, `planningPaths`, активный workstream-роутинг, `.planning/.lock` |
| Roadmap | `lib/roadmap.cjs` | Парсинг roadmap'а, извлечение фаз, обновление прогресса |
| Config | `lib/config.cjs` | Чтение/запись конфига, инициализация секций |
| Verify | `lib/verify.cjs` | Все verification- и validation-команды |
| Template | `lib/template.cjs` | Выбор шаблонов и наполнение переменными |
| Frontmatter | `lib/frontmatter.cjs` | CRUD YAML frontmatter |
| Init | `lib/init.cjs` | Compound-загрузка контекста для всех workflow |
| Milestone | `lib/milestone.cjs` | Архивация майлстоунов, разметка требований |
| Commands | `lib/commands.cjs` | Misc: slug, timestamp, todos, scaffold, stats, websearch |
| Model Profiles | `lib/model-profiles.cjs` | Таблица резолва профилей |
| UAT | `lib/uat.cjs` | Cross-фазовый UAT/verification аудит |
| Profile Output | `lib/profile-output.cjs` | Форматирование developer-профиля |
| Profile Pipeline | `lib/profile-pipeline.cjs` | Пайплайн анализа сессий |
| Graphify | `lib/graphify.cjs` | Knowledge graph build/query/status/diff/snapshot (стоит за `/gsd-graphify`) |
| Learnings | `lib/learnings.cjs` | Извлечение уроков из фаз/SUMMARY-артефактов (стоит за `/gsd-extract-learnings`) |
| Audit | `lib/audit.cjs` | Phase/milestone audit-очередь хендлеры; helper `audit-open` |
| GSD2 Import | `lib/gsd2-import.cjs` | Обратная миграция импортера из GSD-2 проектов (стоит за `/gsd-import --from-gsd2`) |
| Intel | `lib/intel.cjs` | Queryable интеллект кодовой базы (стоит за `/gsd-map-codebase --query`) |

---

## Reviewer CLI Routing

`review.models.<cli>` маппит флейвор ревьюера на shell-команду, вызываемую code-review воркфлоу. Установи через [`/gsd-config --integrations`](COMMANDS.ru-RU.md) или напрямую:

```bash
gsd-sdk query config-set review.models.codex    "codex exec --model gpt-5"
gsd-sdk query config-set review.models.gemini   "gemini -m gemini-2.5-pro"
gsd-sdk query config-set review.models.opencode "opencode run --model claude-sonnet-4"
gsd-sdk query config-set review.models.claude   ""   # очистить — fallback на session-модель
```

Slug'и валидируются против `[a-zA-Z0-9_-]+`; пустые или содержащие путь slug'и отвергаются. Полная справка по полям — в [`docs/CONFIGURATION.md`](CONFIGURATION.md#code-review-cli-routing).

## Обработка секретов

API-ключи, сконфигурированные через `/gsd-settings` (`brave_search`, `firecrawl`, `exa_search`), пишутся в открытом виде в `.planning/config.json`, но маскируются (`****<last-4>`) во всех `config-set` / `config-get` выводах, confirmation-таблицах и интерактивных промптах. См. `get-shit-done/bin/lib/secrets.cjs` для реализации маскирования. Сам файл `config.json` — граница безопасности — защити его правами ФС и держи вне git (`.planning/` gitignored по умолчанию).

---

## Смотри также

- [sdk/src/query/QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md) — матрица реестра, роутинг, golden parity, намеренные CJS-различия
- [Architecture](ARCHITECTURE.ru-RU.md) — где `gsd-sdk query` сидит в оркестрации
- [Command Reference](COMMANDS.ru-RU.md) — пользовательские `/gsd-`-команды
