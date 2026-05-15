# GSD — Руководство пользователя

Детальный справочник по воркфлоу, диагностике и конфигурации. Быстрый старт — в [README](../README.ru-RU.md).

---

## Оглавление

- [Сквозной разбор](#сквозной-разбор)
- [Диаграммы воркфлоу](#диаграммы-воркфлоу)
- [Контракт UI-дизайна](#контракт-ui-дизайна)
- [Spike и Sketch](#spike-и-sketch)
- [Бэклог и треды](#бэклог-и-треды)
- [Workstream'ы](#workstreamы)
- [Безопасность](#безопасность)
- [Справочник команд и конфигурации](#справочник-команд-и-конфигурации)
- [Примеры использования](#примеры-использования)
- [Решение проблем](#решение-проблем)
- [Быстрый справочник восстановления](#быстрый-справочник-восстановления)

Чтобы управлять GSD прямо из тикета в GitHub / Linear / Jira — см. [гайд по issue-driven оркестрации](issue-driven-orchestration.md) — рецепт, как накладывать тикеты трекера на цикл workspace → discuss → plan → execute → verify → review → ship, используя примитивы GSD.

---

## Формы slash-команд (дефис vs двоеточие)

GSD доставляет **один и тот же набор скиллов** во все поддерживаемые рантаймы, но есть два варианта написания slash-форм:

- **Через дефис** — `/gsd-command-name` — используется Claude Code, Copilot, OpenCode, Kilo, Cursor, Windsurf, Augment, Antigravity и Trae.
- **Через двоеточие** — `/gsd:command-name` — используется **только Gemini CLI**. Gemini неймспейсит команды каждого плагина под id плагина, поэтому путь установки переписывает все ссылки в тексте и файлы команд на форму с двоеточием во время установки `--gemini`.

Тебе не нужно выбирать — установщик пишет нужную форму в директорию команд каждого целевого рантайма. Когда читаешь гайд на терминале Gemini, заменяй дефис после `gsd` на двоеточие.

## Введение в namespace-роутинг (`gsd:<namespace>`, v1.40)

В v1.40 поставляется шесть **namespace-метаскилов** как точки входа первого уровня для иерархического роутинга — они держат токен-стоимость eager skill-листинга низкой (~120 токенов на 6 роутеров против ~2150 для плоского листинга 86 скиллов), при этом каждый конкретный sub-скилл остаётся напрямую вызываемым. Тело каждого роутера содержит таблицу роутинга, маппящую твой intent на правильный sub-скилл.

| Namespace | Роутер | Куда роутит |
|-----------|--------|-------------|
| Phase pipeline | `/gsd-workflow` | discuss / plan / execute / verify / phase / progress |
| Project lifecycle | `/gsd-project` | milestones, audits, summary |
| Quality gates | `/gsd-quality` | code review, debug, audit, security, eval, ui |
| Codebase intelligence | `/gsd-context` | map, graphify, docs, learnings |
| Management | `/gsd-manage` | config, workspace, workstreams, thread, update, ship, inbox |
| Exploration & capture | `/gsd-ideate` | explore, sketch, spike, spec, capture |

Тебе почти никогда не нужно вручную набирать namespace-роутер. Их ценность — в слое роутинга, который модель использует чтобы найти правильный sub-скилл. Они существуют, чтобы system prompt мог перечислить 6 записей вместо 86. Если ты уже знаешь конкретную команду (например `/gsd-plan-phase`) — вызывай её напрямую.

---

## Сквозной разбор

Этот разбор показывает, как фазы GSD стыкуются для типичного однофазного проекта — небольшое Node.js REST API, валидирующее подписи вебхуков. Пройди его, чтобы понять что делает каждая команда, что она создаёт и как следующая команда это потребляет.

### 1. Создать проект

```
/gsd-new-project
```

GSD задаёт вопросы про твою идею, спавнит параллельных агентов исследования, извлекает требования и создаёт дорожную карту. Ты одобряешь карту до того, как пишется любой код.

**Пример вывода (сокращённо):**

```
> Что строим?
  Express-middleware для валидации подписей вебхуков.

> Кто пользователь?
  Backend-разработчики, интегрирующие сторонние вебхуки (Stripe, GitHub, Shopify).

[Агенты исследования работают параллельно...]
[Требования извлечены...]

Дорожная карта (1 фаза):
  Phase 1 — Core middleware: валидация подписи HMAC-SHA256,
             timing-safe сравнение, настраиваемое окно толерантности.

Одобрить? [y/n]
```

**Что создаётся:**

```
.planning/
  PROJECT.md          # "Webhook validator middleware — Express, HMAC-SHA256..."
  REQUIREMENTS.md     # REQ-001: валидировать signature-заголовок; REQ-002: timing-safe...
  ROADMAP.md          # Phase 1 status: pending
  STATE.md            # Память сессии, текущая позиция
```

Выдержка из `ROADMAP.md`:
```markdown
## Phase 1 — Core middleware
**Status:** pending
**Goal:** Валидация подписи HMAC-SHA256 с timing-safe сравнением и
настраиваемым окном защиты от replay.
**Requirements:** REQ-001, REQ-002, REQ-003
```

### 2. Обсудить и спланировать фазу

```
/gsd-discuss-phase 1
```

GSD читает цель фазы и спрашивает про твои предпочтения реализации до того как начнётся планирование. Это место, где ты формируешь *как* строить — а не только *что* строить.

```
> Как обрабатывать невалидные подписи?
  Сразу отклонять с 401, логировать сырой заголовок для дебага.

> Окно толерантности — per-route или глобально?
  Глобальный конфиг, но per-route override через опции middleware.

> Предпочтения по HMAC-библиотеке?
  Только Node built-in crypto — никаких лишних зависимостей.
```

**Что создаётся:** `.planning/phases/01-core-middleware/CONTEXT.md`

Выдержка из `CONTEXT.md`:
```markdown
## Implementation Decisions
- Invalid signatures → 401, log raw header
- Tolerance window → global default, per-route override via options object
- HMAC library → Node built-in crypto (no external deps)
- Error format → { error: "invalid_signature", ts: <epoch> }
```

Теперь планируем фазу:

```
/gsd-plan-phase 1
```

GSD спавнит четырёх параллельных агентов исследования (stack, features, architecture, pitfalls), потом планировщик читает `CONTEXT.md` + находки исследования и создаёт атомарные task-планы. Plan-checker проверяет, что каждый план достигает цели фазы, до сохранения.

**Что создаётся:**

```
.planning/phases/01-core-middleware/
  RESEARCH.md         # Находки: документация crypto.timingSafeEqual, паттерны replay-атак...
  01-01-PLAN.md       # Задача: создать core-функцию validateSignature()
  01-02-PLAN.md       # Задача: Express middleware-обёртка + обработка ошибок
```

Выдержка из `01-01-PLAN.md`:
```xml
<task type="auto">
  <name>Create validateSignature core function</name>
  <files>src/validate.js, src/validate.test.js</files>
  <action>
    Use crypto.createHmac('sha256', secret).update(rawBody).digest('hex').
    Compare with crypto.timingSafeEqual() — never === or ==.
    Accept tolerance window in ms; reject if |timestamp - now| exceeds it.
  </action>
  <verify>npm test -- --grep "validateSignature"</verify>
  <done>All timing-safe comparison tests pass; replay outside window returns false</done>
</task>
```

### 3. Выполнение

```
/gsd-execute-phase 1
```

GSD группирует планы в волны (параллельно где независимы, последовательно где есть зависимости), спавнит свежего исполнителя на 200k контекста на каждый план и коммитит каждую задачу атомарно.

```
Wave 1 (parallel):
  [Executor A] → 01-01-PLAN.md (core function)  ✓ committed
  [Executor B] → 01-02-PLAN.md (middleware)      ✓ committed

[Verifier] Checking codebase against phase goals...
  REQ-001 validateSignature() ✓
  REQ-002 timing-safe compare ✓
  REQ-003 tolerance window    ✓
  Status: PASS
```

**Git-история после выполнения:**

```
a1b2c3d feat(01-01): implement validateSignature with timingSafeEqual
d4e5f6g feat(01-02): add Express middleware wrapper and 401 error format
h7i8j9k chore(01): phase 1 verification — all requirements met
```

**Что создаётся:**

```
.planning/phases/01-core-middleware/
  01-01-SUMMARY.md    # "Implemented validateSignature(), 47 lines, tests passing"
  01-02-SUMMARY.md    # "Middleware wraps validator, logs raw header on 401"
  VERIFICATION.md     # REQ-001 ✓, REQ-002 ✓, REQ-003 ✓ — PASS
```

`STATE.md` обновляется автоматически, отражая новый статус фазы:

```markdown
## Current Position
Phase 1 — Core middleware: executed, pending verify
```

### 4. Верификация

```
/gsd-verify-work 1
```

GSD извлекает тестируемые поставки из цели фазы и проводит тебя через них по одной.

```
[1/3] Можешь подключить middleware и передать его в Express-роут?
> да

[2/3] Запрос с валидной подписью возвращает 200?
> да

[3/3] Запрос с невалидной подписью возвращает 401 с { error: "invalid_signature" }?
> нет — я получаю 500

[Диагностика...]
Root cause: middleware ловит TypeError от crypto.timingSafeEqual когда
буферы разной длины. Fix: нормализовать в одинаковую длину перед сравнением.

Fix plan создан: .planning/phases/01-core-middleware/01-03-PLAN.md
Запусти /gsd-execute-phase 1 чтобы применить.
```

После перезапуска execute и повторной верификации:

```
Все 3 проверки прошли. Phase 1 verified.
```

**Что создаётся:** `.planning/phases/01-core-middleware/UAT.md`

### Что дальше

Когда фаза верифицирована, отправляй её:

```
/gsd-ship 1          # Создаёт PR с автогенерируемым телом
```

Тело PR всегда включает обязательные секции GSD: `Summary`, `Changes`, `Requirements Addressed`, `Verification`, `Key Decisions`. В ходе `/gsd-new-project` ты также можешь включить опциональные секции в стиле PRD — user stories, acceptance criteria, риски, release-критерии, утверждение стейкхолдерами. Они добавляются через `ship.pr_body_sections` и не меняют обязательные ядро-секции.

Примеры настройки, определения полей и диагностика — в [Custom PR Body Sections](ship-pr-body-sections.md).

Для многофазных проектов повторяй цикл:

```
/gsd-discuss-phase 2
/gsd-plan-phase 2
/gsd-execute-phase 2
/gsd-verify-work 2
```

Или пусть GSD сам определит следующий шаг:

```
/gsd-progress --next
```

Когда все фазы готовы:

```
/gsd-audit-milestone     # Проверить что все требования зарелизились
/gsd-complete-milestone  # Архив, тег релиза
```

**Релевантные флаги, упомянутые в разборе:**

| Флаг | Команда | Когда использовать |
|------|---------|---------------------|
| `--auto` | `/gsd-new-project` | Пропустить интерактивные вопросы, забрать из PRD-файла |
| `--research` | `/gsd-quick` | Добавить research-агента к разовой задаче |
| `--validate` | `/gsd-quick` | Добавить plan-checking и постфактум-верификацию |
| `--chain` | `/gsd-discuss-phase` | Авто-цепочка discuss → plan → execute без остановок |
| `--skip-research` | `/gsd-plan-phase` | Пропустить research-агентов когда домен уже знаком |
| `--draft` | `/gsd-ship` | Создать черновой PR вместо ready-for-review |

Полный справочник команд со всеми флагами — в [`docs/COMMANDS.md`](COMMANDS.md). Опции конфигурации (профили моделей, workflow-агенты, git-ветвление) — в [`docs/CONFIGURATION.md`](CONFIGURATION.md).

---

## Диаграммы воркфлоу

### Полный жизненный цикл проекта

```
  ┌──────────────────────────────────────────────────┐
  │                   НОВЫЙ ПРОЕКТ                   │
  │  /gsd-new-project                                │
  │  Вопросы -> Research -> Требования -> Roadmap    │
  └─────────────────────────┬────────────────────────┘
                            │
             ┌──────────────▼─────────────┐
             │     ДЛЯ КАЖДОЙ ФАЗЫ:       │
             │                            │
             │  ┌────────────────────┐    │
             │  │ /gsd-discuss-phase │    │  <- Зафиксировать предпочтения
             │  └──────────┬─────────┘    │
             │             │              │
             │  ┌──────────▼─────────┐    │
             │  │ /gsd-ui-phase      │    │  <- Дизайн-контракт (frontend)
             │  └──────────┬─────────┘    │
             │             │              │
             │  ┌──────────▼─────────┐    │
             │  │ /gsd-plan-phase    │    │  <- Research + Plan + Verify
             │  └──────────┬─────────┘    │
             │             │              │
             │  ┌──────────▼─────────┐    │
             │  │ /gsd-execute-phase │    │  <- Параллельное исполнение
             │  └──────────┬─────────┘    │
             │             │              │
             │  ┌──────────▼─────────┐    │
             │  │ /gsd-verify-work   │    │  <- Ручной UAT
             │  └──────────┬─────────┘    │
             │             │              │
             │  ┌──────────▼─────────┐    │
             │  │ /gsd-ship          │    │  <- Создать PR (опционально)
             │  └──────────┬─────────┘    │
             │             │              │
             │  Следующая фаза?───────────┘
             │             │ Нет
             └─────────────┼──────────────┘
                            │
            ┌───────────────▼──────────────┐
            │  /gsd-audit-milestone        │
            │  /gsd-complete-milestone     │
            └───────────────┬──────────────┘
                            │
                   Ещё майлстоун?
                       │          │
                      Да          Нет -> Готово!
                       │
               ┌───────▼──────────────┐
               │  /gsd-new-milestone  │
               └──────────────────────┘
```

### Координация агентов планирования

```
  /gsd-plan-phase N
         │
         ├── Phase Researcher (x4 параллельно)
         │     ├── Stack researcher
         │     ├── Features researcher
         │     ├── Architecture researcher
         │     └── Pitfalls researcher
         │           │
         │     ┌──────▼──────┐
         │     │ RESEARCH.md │
         │     └──────┬──────┘
         │            │
         │     ┌──────▼──────┐
         │     │   Planner   │  <- Читает PROJECT.md, REQUIREMENTS.md,
         │     │             │     CONTEXT.md, RESEARCH.md
         │     └──────┬──────┘
         │            │
         │     ┌──────▼───────────┐     ┌────────┐
         │     │   Plan Checker   │────>│ PASS?  │
         │     └──────────────────┘     └───┬────┘
         │                                  │
         │                             Да   │  Нет
         │                              │   │   │
         │                              │   └───┘  (цикл, до 3x)
         │                              │
         │                        ┌─────▼──────┐
         │                        │ PLAN files │
         │                        └────────────┘
         └── Готово
```

### Архитектура валидации (Nyquist Layer)

Во время research'а plan-фазы GSD теперь маппит покрытие автотестами на каждое требование фазы до того как написана хоть строчка кода. Это гарантирует, что когда исполнитель Claude коммитит задачу, механизм обратной связи уже существует, чтобы верифицировать её за секунды.

Researcher детектит твою существующую тест-инфраструктуру, маппит каждое требование на конкретную тест-команду и идентифицирует тест-каркас, который надо создать до начала имплементации (задачи Wave 0).

Plan-checker форсит это как 8-е измерение верификации: планы, где задачи лишены автоматических verify-команд, не одобряются.

**Output:** `{phase}-VALIDATION.md` — контракт обратной связи на фазу.

**Отключение:** Поставь `workflow.nyquist_validation: false` в `/gsd-settings` для фаз быстрого прототипирования, где тест-инфраструктура не в фокусе.

### Ретроактивная валидация (`/gsd-validate-phase`)

Для фаз, исполненных до появления Nyquist-валидации, или для существующих кодовых баз с только традиционными тест-наборами, можно ретроактивно проаудитить и закрыть пробелы покрытия:

```
  /gsd-validate-phase N
         |
         +-- Детектим состояние (есть VALIDATION.md? SUMMARY.md?)
         |
         +-- Discover: сканим имплементацию, маппим требования на тесты
         |
         +-- Анализируем пробелы: какие требования лишены автопроверок?
         |
         +-- Презентуем план закрытия пробелов на одобрение
         |
         +-- Спавним auditor: генерим тесты, прогоняем, дебажим (макс 3 попытки)
         |
         +-- Обновляем VALIDATION.md
               |
               +-- COMPLIANT -> у всех требований есть автопроверки
               +-- PARTIAL -> часть пробелов эскалирована в manual-only
```

Auditor никогда не модифицирует имплементационный код — только тестовые файлы и VALIDATION.md. Если тест вскрывает баг имплементации, он флагается как эскалация для тебя.

**Когда использовать:** После выполнения фаз, спланированных до включения Nyquist, или после того как `/gsd-audit-milestone` всплыл пробелы Nyquist-комплаенса.

### Режим обсуждения через assumptions

По умолчанию `/gsd-discuss-phase` задаёт open-ended вопросы про твои предпочтения реализации. Режим assumptions инвертирует это: GSD сначала читает кодовую базу, всплывает структурированные предположения про то, как он построил бы фазу, и просит только корректировки.

**Включение:** Поставь `workflow.discuss_mode` в `'assumptions'` через `/gsd-settings`.

**Как работает:**

1. Читает PROJECT.md, маппинг кодовой базы и существующие конвенции
2. Генерирует структурированный список предположений (выбор технологий, паттерны, расположение файлов)
3. Презентует предположения, чтобы ты их подтвердил, поправил или расширил
4. Пишет CONTEXT.md из подтверждённых предположений

**Когда использовать:**

- Опытные разработчики, хорошо знающие свою кодовую базу
- Быстрая итерация, где open-ended вопросы тормозят
- Проекты с устоявшимися и предсказуемыми паттернами

Полная справка по discuss-mode — в [docs/workflow-discuss-mode.md](workflow-discuss-mode.md).

### Gate'ы покрытия решений

Discuss-phase фиксирует решения по реализации в CONTEXT.md в блоке `<decisions>` как нумерованные пункты (`- **D-01:** …`). Два gate'а — добавлены для issue #2492 — гарантируют, что эти решения переживают переход в планы и в код.

**Gate трансляции в plan-фазе (блокирующий).** После планирования GSD отказывается пометить фазу как planned, пока каждое отслеживаемое решение не появится хотя бы в одном `must_haves`, `truths` или теле плана. Gate называет каждое пропущенное решение по id (`D-07: …`), чтобы ты точно знал, что добавить, переместить или переклассифицировать.

**Gate валидации verify-фазы (не блокирующий).** Во время верификации GSD ищет в планах, SUMMARY.md, изменённых файлах и недавних коммит-сообщениях каждое отслеживаемое решение. Промахи логируются в VERIFICATION.md как warning-секция; статус верификации не меняется. Асимметрия намеренная — блокирующий gate дешёв на plan-time, но враждебен на verify-time.

**Как писать решения, которые gate сматчит.** Два режима матча:

1. **Строгий матч по id (рекомендуется).** Цитируй id решения где угодно в плане, который его имплементит — `must_haves.truths: ["D-12: bit offsets exposed"]`, пункт в теле плана, комментарий во frontmatter. Это детерминированно и однозначно.
2. **Soft phrase match (запасной).** Если фраза из 6+ слов из текста решения появляется дословно в любом плане или зарелиженом артефакте — засчитывается. Прощает парафраз, но менее надёжно.

**Как исключить решение.** Если решение действительно не должно отслеживаться — заметка про дискрецию имплементации, информационный capture, уже отложенное решение — пометь его одним из способов:

- Перенеси под заголовок `### Claude's Discretion` внутри `<decisions>`.
- Тегни в пункте: `- **D-08 [informational]:** …`, `- **D-09 [folded]:** …`, `- **D-10 [deferred]:** …`.

**Отключение gate'ов.** Поставь `workflow.context_coverage_gate: false` в `.planning/config.json` (или через `/gsd-settings`), чтобы пропустить оба gate'а молча. По умолчанию `true`.

---

## Контракт UI-дизайна

### Зачем

AI-сгенерированные фронтенды визуально несогласованны не потому, что Claude Code плох в UI, а потому что до исполнения не существовало дизайн-контракта. Пять компонентов, построенных без общей шкалы spacing'а, цветового контракта или стандарта копирайтинга, дадут пять чуть разных визуальных решений.

`/gsd-ui-phase` фиксирует дизайн-контракт до планирования. `/gsd-ui-review` аудитит результат после исполнения.

### Команды

| Команда              | Описание                                                       |
| -------------------- | -------------------------------------------------------------- |
| `/gsd-ui-phase [N]`  | Сгенерировать UI-SPEC.md дизайн-контракт для frontend-фазы     |
| `/gsd-ui-review [N]` | Ретроактивный 6-компонентный визуальный аудит имплементированного UI |

### Воркфлоу `/gsd-ui-phase`

**Когда запускать:** После `/gsd-discuss-phase`, до `/gsd-plan-phase` — для фаз с frontend/UI работой.

**Поток:**

1. Читает CONTEXT.md, RESEARCH.md, REQUIREMENTS.md на существующие решения
2. Детектит состояние дизайн-системы (shadcn components.json, Tailwind-конфиг, существующие токены)
3. Gate инициализации shadcn — предлагает инициализировать, если в React/Next.js/Vite-проекте его нет
4. Спрашивает только неотвеченные вопросы дизайн-контракта (spacing, типографика, цвет, копирайтинг, безопасность реестра)
5. Пишет `{phase}-UI-SPEC.md` в директорию фазы
6. Валидирует по 6 измерениям (Copywriting, Visuals, Color, Typography, Spacing, Registry Safety)
7. Цикл доработки если BLOCKED (макс 2 итерации)

**Output:** `{padded_phase}-UI-SPEC.md` в `.planning/phases/{phase-dir}/`

### Воркфлоу `/gsd-ui-review`

**Когда запускать:** После `/gsd-execute-phase` или `/gsd-verify-work` — для любого проекта с фронтенд-кодом.

**Standalone:** Работает на любом проекте, не только GSD-managed. Если UI-SPEC.md нет, аудитит против абстрактных 6-компонентных стандартов.

**6 столпов (каждый оценивается 1–4):**

1. Copywriting — лейблы CTA, empty states, error states
2. Visuals — фокальные точки, визуальная иерархия, доступность иконок
3. Color — дисциплина использования акцентов, соблюдение 60/30/10
4. Typography — соблюдение ограничений по размеру/весу шрифта
5. Spacing — выравнивание по сетке, согласованность токенов
6. Experience Design — покрытие loading/error/empty состояний

**Output:** `{padded_phase}-UI-REVIEW.md` в директории фазы с оценками и топ-3 приоритетными фиксами.

### Конфигурация

| Настройка                 | По умолчанию | Описание                                                    |
| ------------------------- | ------------ | ----------------------------------------------------------- |
| `workflow.ui_phase`       | `true`       | Генерировать UI-дизайн-контракты для frontend-фаз           |
| `workflow.ui_safety_gate` | `true`       | plan-phase предлагает запустить /gsd-ui-phase для frontend-фаз |

Обе следуют паттерну absent=enabled. Отключение через `/gsd-settings`.

### Инициализация shadcn

Для React/Next.js/Vite-проектов UI-researcher предложит инициализировать shadcn, если `components.json` не найден. Поток:

1. Зайди на `ui.shadcn.com/create` и сконфигурируй свой пресет
2. Скопируй preset-строку
3. Запусти `npx shadcn init --preset {paste}`
4. Пресет кодирует всю дизайн-систему — цвета, border radius, шрифты

Preset-строка становится first-class планировочным артефактом GSD, воспроизводимым между фазами и майлстоунами.

### Gate безопасности реестра

Сторонние реестры shadcn могут инжектить произвольный код. Safety gate требует:

- `npx shadcn view {component}` — осмотри перед установкой
- `npx shadcn diff {component}` — сравни с официальным

Управляется конфигом `workflow.ui_safety_gate`.

### Хранилище скриншотов

`/gsd-ui-review` снимает скриншоты через Playwright CLI в `.planning/ui-reviews/`. `.gitignore` создаётся автоматически чтобы бинари не попали в git. Скриншоты чистятся во время `/gsd-complete-milestone`.

---

## Spike и Sketch

Используй `/gsd-spike` чтобы валидировать техническую осуществимость до планирования, и `/gsd-sketch` чтобы исследовать визуальное направление до проектирования. Оба хранят артефакты в `.planning/` и интегрируются с системой project-skills через их wrap-up companions.

### Когда делать Spike

Spike, когда ты не уверен, осуществим ли технический подход, или хочешь сравнить две имплементации перед тем, как привязать фазу к одной из них.

```
/gsd-spike                              # Интерактивный intake — описывает вопрос, ты подтверждаешь
/gsd-spike "можем ли стримить LLM-токены через SSE"
/gsd-spike --quick "websocket vs SSE latency"
```

Каждый spike прогоняет 2–5 экспериментов. У каждого эксперимента есть:
- Гипотеза **Given / When / Then**, написанная до любого кода
- **Рабочий код** (не псевдокод)
- Вердикт **VALIDATED / INVALIDATED / PARTIAL** с доказательствами

Результаты ложатся в `.planning/spikes/NNN-name/README.md` и индексируются в `.planning/spikes/MANIFEST.md`.

Когда у тебя есть сигнал, запусти `/gsd-spike --wrap-up`, чтобы упаковать находки в `.claude/skills/spike-findings-[project]/` — будущие сессии подгрузят их автоматически через discovery project-skills.

### Когда делать Sketch

Sketch, когда нужно сравнить лейаутные структуры, модели взаимодействия или визуальные решения до написания настоящего компонент-кода.

```
/gsd-sketch                             # Mood intake — изучает ощущение, референсы, основное действие
/gsd-sketch "dashboard layout"
/gsd-sketch --quick "sidebar navigation"
/gsd-sketch --text "onboarding flow"    # Для не-Claude рантаймов (Codex, Gemini и т.д.)
```

Каждый sketch отвечает на **один дизайн-вопрос** 2–3 вариантами в одном `index.html`, который ты открываешь прямо в браузере — без сборки. Варианты используют tab-навигацию и общие CSS-переменные из `themes/default.css`. Все интерактивные элементы (hover, click, transitions) функциональны.

После выбора победителя, запусти `/gsd-sketch --wrap-up` чтобы зафиксировать визуальные решения в `.claude/skills/sketch-findings-[project]/`.

### Поток Spike → Sketch → Phase

```
/gsd-spike "SSE vs WebSocket"     # Валидируем подход
/gsd-spike --wrap-up              # Упаковываем уроки

/gsd-sketch "real-time feed UI"   # Исследуем дизайн
/gsd-sketch --wrap-up             # Упаковываем решения

/gsd-discuss-phase N              # Фиксируем предпочтения (теперь информированные spike + sketch)
/gsd-plan-phase N                 # Планируем уверенно
```

---

## Бэклог и треды

### Бэклог как парковка

Идеи, которые ещё не готовы к активному планированию, идут в бэклог с нумерацией 999.x, оставаясь вне активной последовательности фаз.

```
/gsd-capture --backlog "GraphQL API layer"     # Создаёт 999.1-graphql-api-layer/
/gsd-capture --backlog "Mobile responsive"     # Создаёт 999.2-mobile-responsive/
```

Бэклог-айтемы получают полные phase-директории, так что ты можешь использовать `/gsd-discuss-phase 999.1` чтобы дальше разобрать идею или `/gsd-plan-phase 999.1` когда она готова.

**Ревью и продвижение** через `/gsd-review-backlog` — показывает все бэклог-айтемы и даёт продвинуть (перенести в активную последовательность), оставить (в бэклоге) или удалить.

### Seeds

Seeds — это идеи на будущее с триггер-условиями. В отличие от бэклог-айтемов, seeds всплывают автоматически когда наступает правильный майлстоун.

```
/gsd-capture --seed "Добавить real-time collab когда WebSocket-инфра будет готова"
```

Seeds сохраняют полное WHY и WHEN-всплыть. `/gsd-new-milestone` сканит все seeds и презентует подходящие.

**Хранение:** `.planning/seeds/SEED-NNN-slug.md`

### Постоянные контекстные треды

Треды — это лёгкие cross-session хранилища знаний для работы, которая растягивается на несколько сессий, но не принадлежит конкретной фазе.

```
/gsd-thread                              # Список всех тредов
/gsd-thread fix-deploy-key-auth          # Возобновить существующий тред
/gsd-thread "Investigate TCP timeout"    # Создать новый тред
```

Треды легче чем `/gsd-pause-work` — нет phase-state, нет plan-context. Каждый файл треда включает секции Goal, Context, References, Next Steps.

Треды можно продвинуть в фазы (`/gsd-phase`) или бэклог-айтемы (`/gsd-capture --backlog`) когда они созревают.

**Хранение:** `.planning/threads/{slug}.md`

---

## Workstream'ы

Workstream'ы позволяют работать над несколькими областями майлстоуна одновременно без коллизий состояния. У каждого workstream'а — своя изолированная `.planning/`-стейт, так что переключение между ними не затирает прогресс.

**Когда использовать:** Ты работаешь над фичами майлстоуна, которые охватывают разные concern-области (например, backend API и frontend dashboard), и хочешь планировать/исполнять/обсуждать их независимо без context bleed.

### Команды

| Команда                              | Назначение                                              |
| ------------------------------------ | ------------------------------------------------------- |
| `/gsd-workstreams create <name>`     | Создать новый workstream с изолированным состоянием планирования |
| `/gsd-workstreams switch <name>`     | Переключить активный контекст на другой workstream      |
| `/gsd-workstreams list`              | Показать все workstream'ы и какой активен                |
| `/gsd-workstreams complete <name>`   | Пометить workstream как готовый и заархивировать состояние |

### Как работает

Каждый workstream держит своё поддерево `.planning/`. Когда ты переключаешь workstream, GSD свапает активный планировочный контекст, чтобы `/gsd-progress`, `/gsd-discuss-phase`, `/gsd-plan-phase` и другие команды работали на стейте этого workstream'а. Активный контекст session-scoped когда рантайм предоставляет стабильный session ID — это не даёт одному терминалу или AI-инстансу переписать `STATE.md` другого инстанса.

Это легче чем `/gsd-workspace --new` (создаёт отдельные repo-worktree). Workstream'ы шарят ту же кодовую базу и git-историю, но изолируют планировочные артефакты.

---

## Безопасность

### Глубокая защита (Defense-in-Depth, v1.27)

GSD генерирует markdown-файлы, которые становятся LLM system prompt'ами. Это значит, что любой пользовательский текст, попадающий в планировочные артефакты — потенциальный вектор косвенной prompt injection. В v1.27 введено централизованное усиление безопасности:

**Защита от path traversal:**
Все пользовательские пути файлов (`--text-file`, `--prd`) валидируются на резолв внутрь project-директории. Резолв macOS-symlink `/var` → `/private/var` обрабатывается.

**Детекция prompt injection:**
Модуль `security.cjs` сканит известные injection-паттерны (role overrides, instruction bypasses, system tag injections) в пользовательском тексте до того как он попадёт в планировочные артефакты.

**Runtime-хуки:**

- `gsd-prompt-guard.js` — сканит Write/Edit-вызовы в `.planning/` на injection-паттерны (всегда активен, advisory-only)
- `gsd-workflow-guard.js` — предупреждает при правках файлов вне GSD-воркфлоу (opt-in через `hooks.workflow_guard`)

**CI-сканер:**
`prompt-injection-scan.test.cjs` сканит все agent-, workflow- и command-файлы на встроенные injection-векторы. Запускается как часть тест-набора.

---

### Gate легитимности пакетов (v1.51)

AI-инструменты галлюцинируют имена пакетов. Атакующие предварительно регистрируют эти имена на npm, PyPI и crates.io со зловредными post-install скриптами — техника называется *slopsquatting*. Галлюцинированное имя, прошедшее `npm view`, выглядит легитимно, поэтому оно может незаметно пройти через research → plan → execute пайплайн GSD вплоть до `npm install <malicious-pkg>` на твоей машине.

v1.51 добавляет трёхслойный gate, который останавливает это до того, как оно дойдёт до твоего шелла.

#### Что увидишь

**В RESEARCH.md** — каждая фаза, рекомендующая внешние пакеты, теперь включает таблицу `## Package Legitimacy Audit`:

```markdown
## Package Legitimacy Audit

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| express | npm | 13 yrs | 100M+/wk | github.com/expressjs/express | [OK] | Approved |
| some-new-util | npm | 3 days | 47 | none | [SLOP] | REMOVED |
| api-bridge | npm | 6 mo | 1.2k/wk | github.com/user/api-bridge | [SUS] | Flagged |

**Packages removed due to slopcheck:** some-new-util
**Packages flagged as suspicious:** api-bridge — planner will require human verification before install
```

Пакеты `[SLOP]` удаляются из RESEARCH.md целиком. Они никогда не доходят до планировщика.

**В PLAN.md** — если пакет помечен `[ASSUMED]` (источник — WebSearch, не подтверждён реестром) или `[SUS]` (slopcheck-подозрительный), план включает чекпоинт верификации *до* install-задачи:

```xml
<task type="checkpoint:human-verify">
  <what-built>Package verification required before install</what-built>
  <how-to-verify>
    Verify these packages before proceeding:
    - `api-bridge` [SUS — 6 months old, 1.2k downloads/week, GitHub repo present]
      Check: https://npmjs.com/package/api-bridge
      Look for: maintainer history, issue tracker activity, no suspicious install scripts
  </how-to-verify>
  <resume-signal>Type "verified" once you've confirmed all packages are legitimate</resume-signal>
</task>
```

**Во время исполнения** — если install падает, executor поднимает чекпоинт и останавливается. Он не пытается молча взять похоже названную альтернативу (что может быть ещё опаснее).

#### Вердикты slopcheck

| Вердикт | Значение | Действие GSD |
|---------|----------|--------------|
| `[OK]` | Пакет проходит все проверки легитимности | Идёт дальше — чекпоинт не добавляется |
| `[SUS]` | Подозрительные сигналы (новый, мало скачиваний, нет source-repo и т.д.) | Флаг в Audit-таблице; планировщик добавляет `checkpoint:human-verify` перед install |
| `[SLOP]` | Высокая уверенность в галлюцинации или регистрации атакующим | Удалён из RESEARCH.md; никогда не доходит до планировщика |

#### Провенанс утверждений и WebSearch-пакеты

Имена пакетов, найденные через WebSearch, всегда тегаются `[ASSUMED]` в RESEARCH.md, независимо от того, успешен ли `npm view`. Пакет, существующий в реестре — не то же что пакет, безопасный для установки. `npm view` доказывает только регистрацию, но не легитимность.

`[ASSUMED]`-пакеты триггерят тот же gate `checkpoint:human-verify`, что и `[SUS]`-пакеты. Ты увидишь чекпоинт со ссылкой на страницу реестра и подсказку, на что смотреть.

#### Если slopcheck не установлен

GSD пытается `pip install slopcheck` во время research'а. Если не получается:

- Каждый рекомендованный пакет тегается `[ASSUMED]`
- Планировщик гейтит каждый install задачей `checkpoint:human-verify`
- Research и planning завершаются нормально — ничего жёстко не падает

Это намеренно строже, чем нормальный поток: недоступность slopcheck означает, что у каждого install'а есть человеческий чекпоинт, что является самым безопасным fallback'ом.

Поставить slopcheck вручную:

```bash
pip install slopcheck
# проверка: slopcheck install express --json
```

#### Зависимость slopcheck

`slopcheck` — это MIT-лицензированный Python-инструмент, поддерживаемый ToxSec (исследователем, задокументировавшим slopsquatting). Он проверяет пакеты на npm, PyPI, crates.io, RubyGems, Go modules, Maven и Packagist по мультисигнальной эвристике: возраст в реестре, число загрузок, привязка к source-repo, наименовательная близость к популярным пакетам, registry-специфические паттерны подозрений.

Если `slopcheck` когда-либо станет недоступен или брошен, `[ASSUMED]`-gate fallback GSD гарантирует что ты всегда получишь человеческий чекпоинт перед любым install'ом — система никогда не деградирует молча к pre-v1.51 поведению.

---

### Координация волн исполнения

```
  /gsd-execute-phase N
         │
         ├── Анализ зависимостей планов
         │
         ├── Wave 1 (независимые планы):
         │     ├── Executor A (свежий 200K-контекст) -> commit
         │     └── Executor B (свежий 200K-контекст) -> commit
         │
         ├── Wave 2 (зависит от Wave 1):
         │     └── Executor C (свежий 200K-контекст) -> commit
         │
         └── Verifier
               ├── Проверка кодовой базы против целей фазы
               ├── Аудит качества тестов (отключённые тесты, циклические паттерны, сила assertion'ов)
               │
               ├── PASS -> VERIFICATION.md (успех)
               └── FAIL -> Issues логируются для /gsd-verify-work
```

### Brownfield-воркфлоу (существующая кодовая база)

```
  /gsd-map-codebase
         │
         ├── Stack Mapper      -> codebase/STACK.md
         ├── Arch Mapper       -> codebase/ARCHITECTURE.md
         ├── Convention Mapper -> codebase/CONVENTIONS.md
         └── Concern Mapper    -> codebase/CONCERNS.md
                │
        ┌───────▼──────────┐
        │ /gsd-new-project │  <- Вопросы фокусируются на том что ДОБАВЛЯЕШЬ
        └──────────────────┘
```

---

## Воркфлоу code review

### Code Review фазы

После исполнения фазы запускай структурный code-review до UAT:

```bash
/gsd-code-review 3               # Ревью всех изменённых файлов фазы 3
/gsd-code-review 3 --depth=deep  # Глубокий cross-file review (графы импортов, цепи вызовов)
```

Reviewer определяет scope файлов автоматически через SUMMARY.md (предпочтительно) или fallback на git diff. Находки классифицируются как Critical, Warning или Info в `{phase}-REVIEW.md`.

```bash
/gsd-code-review 3 --fix           # Чинит Critical + Warning находки атомарно
/gsd-code-review 3 --fix --auto    # Чинит и пере-ревьюит пока не чисто (макс 3 итерации)
```

### Автономный Audit-to-Fix

Запустить аудит и починить все автофиксимые проблемы за один проход:

```bash
/gsd-audit-fix                   # Аудит + классификация + фикс (medium+ severity, макс 5)
/gsd-audit-fix --dry-run         # Превью классификации без фикса
```

### Code Review в полном цикле фазы

Шаг review встраивается после execution и до UAT:

```
/gsd-execute-phase N   ->  /gsd-code-review N  ->  /gsd-code-review N --fix  ->  /gsd-verify-work N
```

---

## Исследование и discovery

### Сократические исследования

Перед тем как привязаться к новой фазе или плану, используй `/gsd-explore` чтобы продумать идею:

```bash
/gsd-explore                           # Open-ended ideation
/gsd-explore "caching strategy"        # Изучить конкретную тему
```

Сессия исследования проводит тебя через зондирующие вопросы, опционально спавнит research-агента и роутит выход в подходящий артефакт GSD: заметка, todo, seed, research question, обновление требований или новая фаза.

### Codebase Intelligence

Для query'ируемых insights по кодовой базе без чтения всего кода, включи intel-систему:

```json
{ "intel": { "enabled": true } }
```

Потом построй индекс:

```bash
/gsd-map-codebase --query refresh             # Анализ кодовой базы и запись .planning/intel/
/gsd-map-codebase --query auth                # Поиск термина по всем intel-файлам
/gsd-map-codebase --query status              # Проверка свежести intel-файлов
/gsd-map-codebase --query diff                # Что изменилось с последнего snapshot'а
```

Intel-файлы покрывают стек, API surface, граф зависимостей, роли файлов и архитектурные решения.

### Быстрое сканирование

Для фокусной оценки без полных накладных `/gsd-map-codebase`:

```bash
/gsd-map-codebase --fast                        # Быстрый обзор tech + arch
/gsd-map-codebase --fast --focus quality        # Только качество и здоровье кода
/gsd-map-codebase --fast --focus concerns       # Зоны риска и concerns
```

---

## Справочник команд и конфигурации

- **Справочник команд:** см. [`docs/COMMANDS.md`](COMMANDS.md) для флагов, подкоманд и примеров каждой стабильной команды. Авторитетный список зашипленных команд — в [`docs/INVENTORY.md`](INVENTORY.md#commands-75-shipped).
- **Справочник конфигурации:** см. [`docs/CONFIGURATION.md`](CONFIGURATION.md) для полной схемы `config.json`, дефолтов и происхождения каждой настройки, per-agent таблицы профилей моделей (включая опцию `inherit` для не-Claude рантаймов), стратегий git-ветвления и настроек безопасности.
- **Discuss Mode:** см. [`docs/workflow-discuss-mode.md`](workflow-discuss-mode.md) для режимов interview vs assumptions.

Этот гайд намеренно не пере-документирует команды или настройки: ведение двух копий ранее приводило к дрейфу (дефолт `workflow.discuss_mode`, дефолт `claude_md_path`, покрытие агентов в таблице профилей моделей). Правило single-source-of-truth форсится механически drift-guard тестами, привязанными к `docs/INVENTORY.md`.

---

## Примеры использования

### Новый проект (полный цикл)

```bash
claude --dangerously-skip-permissions
/gsd-new-project            # Ответить на вопросы, сконфигурировать, одобрить roadmap
/clear
/gsd-discuss-phase 1        # Зафиксировать предпочтения
/gsd-ui-phase 1             # Дизайн-контракт (для frontend-фаз)
/gsd-plan-phase 1           # Research + plan + verify
/gsd-execute-phase 1        # Параллельное исполнение
/gsd-verify-work 1          # Ручной UAT
/gsd-ship 1                 # Создать PR из проверенной работы
/gsd-ui-review 1            # Визуальный аудит (frontend-фазы)
/clear
/gsd-progress --next        # Авто-определить и запустить следующий шаг
...
/gsd-audit-milestone        # Проверить что всё отгружено
/gsd-complete-milestone     # Архив, тег, готово
/gsd-pause-work --report    # Сгенерировать summary сессии
```

### Новый проект из существующего документа

```bash
/gsd-new-project --auto @prd.md   # Авто-прогон research/requirements/roadmap из твоего документа
/clear
/gsd-discuss-phase 1               # Дальше — нормальный поток
```

### Существующая кодовая база

```bash
/gsd-map-codebase           # Анализ того что есть (параллельные агенты)
/gsd-new-project            # Вопросы фокусируются на том что ДОБАВЛЯЕШЬ
# (далее обычный phase-воркфлоу)
```

**Post-execute детекция дрейфа (#2003).** После каждого `/gsd-execute-phase`, GSD проверяет внесла ли фаза достаточно структурных изменений (новые директории, barrel-экспорты, миграции, route-модули) чтобы сделать `.planning/codebase/STRUCTURE.md` устаревшим. Если да, дефолтное поведение — печать однократного предупреждения с предложением точной команды `/gsd-map-codebase --paths …` для рефреша только затронутых поддеревьев. Поведение меняется так:

```bash
/gsd-settings workflow.drift_action auto-remap       # авто-remap
/gsd-settings workflow.drift_threshold 5             # настройка чувствительности
```

Gate не блокирующий: любой внутренний сбой логируется, фаза продолжается.

### Быстрый багфикс

```bash
/gsd-quick
> "Fix the login button not responding on mobile Safari"
```

### Возобновление после перерыва

```bash
/gsd-progress               # Где остановился и что дальше
# или
/gsd-resume-work            # Полное восстановление контекста из последней сессии
```

### Подготовка к релизу

```bash
/gsd-audit-milestone        # Проверить покрытие требований, найти заглушки
/gsd-complete-milestone     # Архив, тег, готово
```

### Пресеты скорость vs качество

| Сценарий       | Mode          | Granularity | Profile    | Research | Plan Check | Verifier |
| -------------- | ------------- | ----------- | ---------- | -------- | ---------- | -------- |
| Прототипирование | `yolo`      | `coarse`    | `budget`   | off      | off        | off      |
| Нормальная разработка | `interactive` | `standard` | `balanced` | on    | on         | on       |
| Production     | `interactive` | `fine`      | `quality`  | on       | on         | on       |

**Пропуск discuss-phase в автономном режиме:** Когда работаешь в `yolo`-режиме с уже зафиксированными в PROJECT.md устойчивыми предпочтениями, поставь `workflow.skip_discuss: true` через `/gsd-settings`. Это полностью обходит discuss-phase и пишет минимальный CONTEXT.md, выведенный из цели фазы в ROADMAP. Полезно когда твой PROJECT.md и конвенции достаточно полны, что обсуждение не добавляет новой информации.

### Смена скоупа посреди майлстоуна

```bash
/gsd-phase                  # Дописать новую фазу в roadmap (режим по умолчанию)
# или
/gsd-phase --insert 3       # Вставить срочную работу между фазами 3 и 4
# или
/gsd-phase --remove 7       # Убрать фазу 7 из скоупа и перенумеровать
# или
/gsd-phase --edit 4         # Отредактировать любое поле фазы 4 на месте
```

### Многопроектные workspace'ы

Работа над несколькими репами или фичами параллельно с изолированным состоянием GSD.

```bash
# Создать workspace с репами из твоего монорепо
/gsd-workspace --new --name feature-b --repos hr-ui,ZeymoAPI

# Изоляция feature-ветки — worktree текущего репо со своим .planning/
/gsd-workspace --new --name feature-b --repos .

# Затем cd в workspace и инициализируй GSD
cd ~/gsd-workspaces/feature-b
/gsd-new-project

# Список и управление workspace'ами
/gsd-workspace --list
/gsd-workspace --remove feature-b
```

Каждый workspace получает:

- Свой `.planning/` (полностью независимый от исходных реп)
- Git worktree'ы (по умолчанию) или клоны указанных реп
- Манифест `WORKSPACE.md`, отслеживающий участвующие репы

---

## Решение проблем

### Программный CLI (`gsd-sdk query` vs `gsd-tools.cjs`)

Для автоматизации и copy-paste из доков предпочитай **`gsd-sdk query`** с зарегистрированной подкомандой (см. [CLI-TOOLS.md — SDK and programmatic access](CLI-TOOLS.md#sdk-and-programmatic-access) и [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md)). Легаси-CLI `node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs` остаётся поддерживаемым для dual-mode операций.

**Только CLI (не в query-реестре):** **graphify**, **from-gsd2** / **gsd2-import** — вызывать через `gsd-tools.cjs` (см. [QUERY-HANDLERS.md](../sdk/src/query/QUERY-HANDLERS.md)). **Две разные формы JSON `state` в легаси-CLI:** `state json` (rebuild frontmatter) vs `state load` (`config` + `state_raw` + flags). **`gsd-sdk query` сегодня:** и `state.json`, и `state.load` резолвятся в frontmatter-rebuild handler — используй `node …/gsd-tools.cjs state load`, когда нужна форма CJS `state load`. См. [CLI-TOOLS.md](CLI-TOOLS.md#sdk-and-programmatic-access) и QUERY-HANDLERS.

### STATE.md рассинхронизирован

Если STATE.md показывает неверный статус или позицию фазы, используй команды state-консистентности (**только CJS** пока не портировано в query-слой):

```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state validate          # Найти дрейф между STATE.md и файловой системой
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state sync --verify     # Превью того что изменит sync
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state sync              # Реконструировать STATE.md с диска
```

Эти команды добавлены в v1.32 и заменяют ручное редактирование STATE.md.

### Бесконечный retry-цикл Read-Before-Edit

Некоторые не-Claude рантаймы (Cline, Augment Code) могут попасть в бесконечный retry-цикл, когда агент пытается отредактировать файл, который не читал. Хук `gsd-read-before-edit.js` (v1.32) детектит этот паттерн и советует сначала прочитать файл. Если твой рантайм не поддерживает PreToolUse-хуки, добавь в `CLAUDE.md` проекта:

```markdown
## Edit Safety Rule
Always read a file before editing it. Never call Edit or Write on a file you haven't read in this session.
```

### "Project already initialized"

Ты запустил `/gsd-new-project`, но `.planning/PROJECT.md` уже существует. Это safety-check. Если хочешь начать заново — удали директорию `.planning/`.

### Деградация контекста в долгих сессиях

Чисти контекстное окно между крупными командами: `/clear` в Claude Code. GSD спроектирован вокруг свежих контекстов — каждый sub-агент получает чистое 200K-окно. Если качество падает в основной сессии, очисти и используй `/gsd-resume-work` или `/gsd-progress` для восстановления состояния.

### Планы выглядят неправильно или не согласуются

Запускай `/gsd-discuss-phase [N]` до планирования. Большинство проблем с качеством планов идёт от того, что Claude делает предположения, которые `CONTEXT.md` бы предотвратил. Также можно запустить `/gsd-discuss-phase --assumptions [N]`, чтобы увидеть что Claude собирается делать до фиксации в плане.

### Discuss-Phase использует технический жаргон, который я не понимаю

`/gsd-discuss-phase` адаптирует свой язык под твой `USER-PROFILE.md`. Если профиль указывает на нетехнического владельца — `learning_style: guided`, `jargon` в списке триггеров фрустрации или `explanation_depth: high-level` — gray-area вопросы автоматически переформулируются в язык продуктовых исходов вместо терминологии имплементации.

Чтобы включить: запусти `/gsd-profile-user` чтобы сгенерировать профиль. Профиль хранится в `~/.claude/get-shit-done/USER-PROFILE.md` и читается автоматически на каждом вызове `/gsd-discuss-phase`. Никакой другой конфигурации не нужно.

### Выполнение падает или производит заглушки

Проверь, что план не был слишком амбициозен. У плана должно быть максимум 2–3 задачи. Если задачи слишком крупные, они превышают то, что одно контекстное окно может надёжно произвести. Пере-планируй с меньшим scope'ом.

### Потерял где находишься

Запусти `/gsd-progress`. Он читает все state-файлы и говорит точно, где ты и что делать дальше.

### Нужно что-то поменять после исполнения

Не запускай заново `/gsd-execute-phase`. Используй `/gsd-quick` для точечных фиксов или `/gsd-verify-work` чтобы системно найти и починить проблемы через UAT.

### Слишком дорого по модели

Переключись на budget-профиль: `/gsd-config --profile budget`. Отключи research- и plan-check агентов через `/gsd-settings` если домен знаком тебе (или Claude).

### Тюнинг стоимости моделей по фазам (`models`) — добавлено в v1.40

Если ты слышал «используй Opus для планирования, Sonnet для верификации» и хочешь применить это, не разбираясь в таксономии агентов, добавь блок `models` в `.planning/config.json`:

```json
{
  "model_profile": "balanced",
  "models": {
    "planning": "opus",
    "discuss": "opus",
    "research": "sonnet",
    "execution": "opus",
    "verification": "sonnet",
    "completion": "sonnet"
  }
}
```

Шесть слотов (`planning` / `discuss` / `research` / `execution` / `verification` / `completion`) принимают tier-алиасы (`opus`, `sonnet`, `haiku`, `inherit`). Каждый слот покрывает группу агентов — например, `models.research = "sonnet"` применяется к `gsd-phase-researcher`, `gsd-codebase-mapper`, `gsd-research-synthesizer` и другим research-агентам разом.

Нужно per-agent исключение? Добавь рядом `model_overrides` — оно выигрывает над `models`:

```json
{
  "models": { "research": "sonnet" },
  "model_overrides": {
    "gsd-codebase-mapper": "haiku"
  }
}
```

Это даёт sonnet всем research-агентам *кроме* codebase mapper'а, который катит haiku для дешёвого, но широкого fan-out скана.

Полная таблица маппинга и правила приоритета — в [Per-Phase-Type Models](CONFIGURATION.md#per-phase-type-models-models--added-in-v140) справочника конфигурации.

### Cheap-by-default через `dynamic_routing` — добавлено в v1.40

Если ты платил opus-тарифы везде как страховку от одной сложной верификации — dynamic_routing переворачивает это: каждый агент стартует на более дешёвом tier'е и эскалируется только когда оркестратор маркирует soft-failure (верификация неоднозначна, plan-check FLAG и т.д.).

```json
{
  "dynamic_routing": {
    "enabled": true,
    "tier_models": {
      "light":    "haiku",
      "standard": "sonnet",
      "heavy":    "opus"
    },
    "escalate_on_failure": true,
    "max_escalations": 1
  }
}
```

У каждого агента дефолтный tier (`light`, `standard` или `heavy`). На первой попытке GSD выбирает `tier_models[default_tier]`. Если оркестратор детектит soft-failure, агент перезапускается на следующем tier'е. `max_escalations` ограничивает общее число retry, чтобы runaway-цикл не сжёг бюджет.

Конкретно:
- `gsd-codebase-mapper` (default `light`) → первая попытка = `haiku`. При эскалации → `sonnet`.
- `gsd-verifier` (default `standard`) → первая попытка = `sonnet`. При эскалации → `opus`.
- `gsd-planner` (default `heavy`) → всегда `opus`. Выше tier'а нет, эскалироваться некуда.

Чтобы выключить — `dynamic_routing.enabled: false` (дефолт) — поведение идентично сегодняшнему.

Полное маппинг агент → tier и правила приоритета — в [Dynamic Routing](CONFIGURATION.md#dynamic-routing-with-failure-tier-escalation-dynamic_routing--added-in-v140) справочника конфигурации.

### Урезать MCP-серверы чтобы снизить per-turn стоимость (самый большой рычаг, которым GSD не владеет)

До тюнинга `model_profile` или `models.<phase_type>` проаудить, какие **MCP-серверы** включены в твоей harness. Каждый включённый MCP-сервер инжектит свою tool-схему в каждый turn — тяжёлые серверы вроде browser/playwright или платформо-специфичных хелперов могут стоить 20k+ токенов каждый, часто перевешивая всё что может сэкономить резолвер GSD.

Это **harness-настройка**, не GSD-настройка. Переключатель живёт в `.claude/settings.json`:

```json
{
  "enabledMcpjsonServers": ["context7"],
  "disabledMcpjsonServers": ["playwright", "mac-tools"]
}
```

Быстрый аудит перед длинной фазой:

- Включены ли какие-то browser/playwright инструменты, когда у фазы нет UI-работы?
- Включены ли платформо-специфичные инструменты (Mac-tools, Windows-tools, OS-specific), когда они не нужны?
- Включены ли какие-то проектные MCP из другого проекта здесь?

Каждый отключённый сервер убирает свою схему из каждого последующего turn до конца сессии. Тримминг MCP **компонуется** с тюнингом `model_profile` — оба рычага аддитивны, экономия от MCP всплывает мгновенно через каждого sub-агента, которого спавнит оркестратор.

Полный аудит, harness-справочник и нота композиции с `model_profile` — в [MCP Tool Schema Cost](../get-shit-done/references/context-budget.md#mcp-tool-schema-cost-harness-concern) в bundled-референсе `context-budget.md`.

### Использование не-Claude рантаймов (Codex, OpenCode, Gemini CLI, Kilo)

Если ты установил GSD под не-Claude рантайм, установщик уже сконфигурировал резолв моделей так, чтобы все агенты использовали дефолтную модель рантайма. Ручной настройки не нужно. Конкретно: установщик ставит `resolve_model_ids: "omit"` в твоём конфиге, что говорит GSD пропустить резолв Anthropic-model-id и дать рантайму выбирать свою дефолтную модель.

Чтобы назначить разные модели разным агентам на не-Claude рантайме, добавь `model_overrides` в `.planning/config.json` с полностью квалифицированными model ID, которые твой рантайм понимает:

```json
{
  "resolve_model_ids": "omit",
  "model_overrides": {
    "gsd-planner": "o3",
    "gsd-executor": "o4-mini",
    "gsd-debugger": "o3"
  }
}
```

Установщик авто-конфигурит `resolve_model_ids: "omit"` для Gemini CLI, OpenCode, Kilo и Codex. Если ты вручную настраиваешь не-Claude рантайм, добавь это сам в `.planning/config.json`.

#### Переключение с Claude на Codex одним изменением конфига (#2517)

Если хочешь tier-моделей на Codex без писания большого блока `model_overrides`, поставь `runtime: "codex"` и выбери профиль:

```json
{
  "runtime": "codex",
  "model_profile": "balanced"
}
```

GSD будет резолвить tier каждого агента (`opus`/`sonnet`/`haiku`) в Codex-нативную модель и reasoning effort, определённые в runtime tier map (`gpt-5.4` xhigh / `gpt-5.3-codex` medium / `gpt-5.4-mini` medium). Codex-установщик встраивает и `model`, и `model_reasoning_effort` в TOML каждого агента автоматически. Чтобы переопределить один tier — добавь `model_profile_overrides.codex.<tier>`. См. [Runtime-Aware Profiles](CONFIGURATION.md#runtime-aware-profiles-2517).

См. [справочник конфигурации](CONFIGURATION.md#non-claude-runtimes-codex-opencode-gemini-cli-kilo) для полного объяснения.

### Установка для Cline

Cline использует rules-based интеграцию — GSD ставится как `.clinerules`, а не как slash-команды.

```bash
# Глобальная установка (на все проекты)
npx get-shit-done-cc --cline --global

# Локальная установка (только этот проект)
npx get-shit-done-cc --cline --local
```

Глобальная пишет в `~/.cline/`. Локальная — в `./.cline/`. Кастомные slash-команды не регистрируются — правила GSD грузятся Cline'ом автоматически из rules-файла.

### Установка для CodeBuddy

CodeBuddy использует skills-based интеграцию.

```bash
npx get-shit-done-cc --codebuddy --global
```

Скиллы ставятся в `~/.codebuddy/skills/gsd-*/SKILL.md`.

### Установка для Qwen Code

Qwen Code использует тот же open skills стандарт, что Claude Code 2.1.88+.

```bash
npx get-shit-done-cc --qwen --global
```

Скиллы ставятся в `~/.qwen/skills/gsd-*/SKILL.md`. Используй переменную окружения `QWEN_CONFIG_DIR` чтобы переопределить дефолтный путь установки.

### Установка для preрелизных редакций (Next / Nightly / Insiders / Preview)

Многие поддерживаемые рантаймы поставляют prerelease-редакцию рядом со стабильным релизом — Windsurf Next, Cursor Nightly, VS Code Insiders, Codex preview-каналы, JetBrains EAP и т.д. Prerelease-редакции читают из соседней конфиг-директории, так что дефолтный путь установки до них не дотянется.

GSD не перечисляет prerelease-редакции как отдельные именованные рантаймы. Они обслуживаются через существующие переменные окружения `<RUNTIME>_CONFIG_DIR` и политику free-string рантайма (см. [#2517](https://github.com/gsd-build/get-shit-done/issues/2517)) — установка работает, пути резолвятся, GSD функционирует. Prerelease-редакции — **best-effort и отдельно не тестируются** в release-CI.

**Паттерн.** Установи переменную окружения `*_CONFIG_DIR` рантайма в prerelease-директорию перед запуском установщика:

```bash
WINDSURF_CONFIG_DIR=~/.codeium/windsurf-next npx get-shit-done-cc@latest --windsurf --global
```

Выбери соответствующий стабильный рантайм в промпте установщика. Скиллы лягут в prerelease-директорию; команды появятся в prerelease-редакторе.

**Справочник env-vars для поддерживаемых рантаймов:**

| Рантайм | Стабильный дефолт | Override env var |
|---|---|---|
| Claude Code | `~/.claude` | `CLAUDE_CONFIG_DIR` |
| Gemini CLI | `~/.gemini` | `GEMINI_CONFIG_DIR` |
| OpenCode | `XDG_CONFIG_HOME/opencode` | `OPENCODE_CONFIG_DIR` |
| Codex | (per Codex CLI) | флаг `--config-dir` |
| Copilot | `~/.copilot` | `COPILOT_CONFIG_DIR` |
| Cursor | `~/.cursor` | `CURSOR_CONFIG_DIR` |
| Windsurf | `~/.codeium/windsurf` | `WINDSURF_CONFIG_DIR` |
| Antigravity | `~/.gemini/antigravity` | `ANTIGRAVITY_CONFIG_DIR` |
| Augment | `~/.augment` | `AUGMENT_CONFIG_DIR` |
| Trae | `~/.trae` | `TRAE_CONFIG_DIR` |
| Qwen Code | `~/.qwen` | `QWEN_CONFIG_DIR` |
| Kilo | `~/.config/kilo` | `KILO_CONFIG_DIR` |
| CodeBuddy | `~/.codebuddy` | `CODEBUDDY_CONFIG_DIR` |
| Cline | `~/.cline` | `CLINE_CONFIG_DIR` |

Если prerelease-канал твоего рантайма не указан, направь подходящий env var на его конфиг-директорию и заведи issue если установка падает по причине, не связанной с маппингом пути.

### Использование Claude Code с не-Anthropic провайдерами (OpenRouter, локальный)

Если sub-агенты GSD вызывают Anthropic-модели, а ты платишь через OpenRouter или локальный провайдер, переключись на профиль `inherit`: `/gsd-config --profile inherit`. Это заставит всех агентов использовать модель твоей текущей сессии вместо конкретных Anthropic-моделей. См. также `/gsd-settings` → Model Profile → Inherit.

### Работа над чувствительным/приватным проектом

Поставь `commit_docs: false` во время `/gsd-new-project` или через `/gsd-settings`. Добавь `.planning/` в `.gitignore`. Планировочные артефакты останутся локальными и никогда не попадут в git.

### Обновление GSD затёрло мои локальные изменения

С v1.17 установщик бекапит локально модифицированные файлы в `gsd-local-patches/`. Запусти `/gsd-update --reapply` чтобы вмержить твои изменения обратно.

### Не получается обновиться через npm

Если `npx get-shit-done-cc` падает из-за npm outage или сетевых ограничений, см. [docs/manual-update.md](manual-update.md) — пошаговое ручное обновление, работающее без доступа к npm.

### Показывать уведомления об обновлениях GSD без statusline GSD

GSD проверяет новые версии в фоне и пишет результат в `~/.cache/gsd/gsd-update-check.json`. По умолчанию statusline GSD (`hooks/gsd-statusline.js`) читает этот кеш и показывает индикатор обновления. Если используешь другой statusline (например `ccstatusline`) или вообще никакой — инфа об обновлении невидима.

**Opt-in fix:** во время интерактивной установки, когда ты отказываешься (или оставляешь свой существующий) statusline, установщик предлагает одноразовый промпт:

```text
Optional: GSD update banner
  1) No banner (default)
  2) Install update banner
```

Выбери `2` (или набери `y`/`yes`), и установщик зарегистрирует `hooks/gsd-update-banner.js` как `SessionStart`-хук. Со следующей сессии GSD печатает одну строку `systemMessage` только когда кеш сообщает что обновление доступно:

```text
GSD update available: 1.39.0 → 1.40.0. Run /gsd-update.
```

Banner молчит когда обновлений нет. Если cache-файл повреждён, GSD выдаёт одну диагностическую строку (`GSD update check failed.`) и молчит 24 часа, чтобы битый кеш не зудел каждую сессию.

**Opt-out / удаление:** удали SessionStart-хук, ссылающийся на `gsd-update-banner.js`, из `settings.json` твоего рантайма (Claude Code: `~/.claude/settings.json`; Gemini: `~/.gemini/settings.json`). `npx get-shit-done-cc --uninstall` удалит и скрипт, и регистрацию за один проход.

Banner не предлагается когда установлен statusline GSD — этот канал уже всплывает инфу про обновления, повторный prompt был бы шумом.

### Диагностика воркфлоу (`/gsd-forensics`)

Когда воркфлоу падает неочевидным образом — планы ссылаются на несуществующие файлы, исполнение даёт неожиданные результаты, состояние выглядит повреждённым — запусти `/gsd-forensics` чтобы сгенерировать диагностический отчёт.

**Что проверяет:**

- Аномалии git-истории (осиротевшие коммиты, неожиданное состояние ветки, артефакты rebase)
- Целостность артефактов (отсутствующие/малформед планировочные файлы, битые cross-references)
- Несоответствия состояния (статус в ROADMAP vs фактическое наличие файлов, дрейф конфига)

**Output:** Диагностический отчёт в `.planning/forensics/` с находками и предложенными шагами remediation.

### Sub-агент Executor получает "Permission denied" на Bash-командах

Sub-агентам `gsd-executor` нужен write-capable Bash-доступ к стандартному инструментарию проекта — `git commit`, `bin/rails`, `bundle exec`, `npm run`, `uv run` и подобным. Дефолтный `~/.claude/settings.json` Claude Code разрешает только узкий набор read-only git-команд, поэтому свежая установка натыкается на "Permission to use Bash has been denied" в первый раз, когда executor пытается сделать commit или запустить build-инструмент.

**Фикс: добавь нужные паттерны в `~/.claude/settings.json`.**

Какие паттерны нужны — зависит от твоего стека. Скопируй блок под свой стек и добавь в массив `permissions.allow`.

#### Необходимое для всех стеков (git + gh)

```json
"Bash(git add:*)",
"Bash(git commit:*)",
"Bash(git merge:*)",
"Bash(git worktree:*)",
"Bash(git rebase:*)",
"Bash(git reset:*)",
"Bash(git checkout:*)",
"Bash(git switch:*)",
"Bash(git restore:*)",
"Bash(git stash:*)",
"Bash(git rm:*)",
"Bash(git mv:*)",
"Bash(git fetch:*)",
"Bash(git cherry-pick:*)",
"Bash(git apply:*)",
"Bash(gh:*)"
```

#### Rails / Ruby

```json
"Bash(bin/rails:*)",
"Bash(bin/brakeman:*)",
"Bash(bin/bundler-audit:*)",
"Bash(bin/importmap:*)",
"Bash(bundle:*)",
"Bash(rubocop:*)",
"Bash(erb_lint:*)"
```

#### Python / uv

```json
"Bash(uv:*)",
"Bash(python:*)",
"Bash(pytest:*)",
"Bash(ruff:*)",
"Bash(mypy:*)"
```

#### Node / npm / pnpm / bun

```json
"Bash(npm:*)",
"Bash(npx:*)",
"Bash(pnpm:*)",
"Bash(bun:*)",
"Bash(node:*)"
```

#### Rust / Cargo

```json
"Bash(cargo:*)"
```

**Пример куска `~/.claude/settings.json` (Rails-проект):**

```json
{
  "permissions": {
    "allow": [
      "Write",
      "Edit",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git merge:*)",
      "Bash(git worktree:*)",
      "Bash(git rebase:*)",
      "Bash(git reset:*)",
      "Bash(git checkout:*)",
      "Bash(git switch:*)",
      "Bash(git restore:*)",
      "Bash(git stash:*)",
      "Bash(git rm:*)",
      "Bash(git mv:*)",
      "Bash(git fetch:*)",
      "Bash(git cherry-pick:*)",
      "Bash(git apply:*)",
      "Bash(gh:*)",
      "Bash(bin/rails:*)",
      "Bash(bin/brakeman:*)",
      "Bash(bin/bundler-audit:*)",
      "Bash(bundle:*)",
      "Bash(rubocop:*)"
    ]
  }
}
```

**Per-project права (только для одной репы):** Если предпочитаешь дать эти паттерны только одному проекту, а не глобально, добавь тот же блок `permissions.allow` в `.claude/settings.local.json` в корне проекта вместо `~/.claude/settings.json`. Claude Code сначала проверяет project-local настройки.

**Интерактивная подсказка:** Когда executor заблокирован посреди фазы, он сообщит точно нужный паттерн (например `"Bash(bin/rails:*)"`), чтобы ты добавил и перезапустил `/gsd-execute-phase`.

### Sub-агент выглядит как упавший, но работа сделана

Известный workaround для бага классификации Claude Code. Оркестраторы GSD (execute-phase, quick) делают spot-check фактического output'а до того как репортить failure. Если видишь сообщение о failure'е, но коммиты сделаны — проверь `git log`. Работа могла пройти успешно.

### Параллельное исполнение даёт build lock ошибки

Если видишь падения pre-commit хуков, contention `cargo lock` или 30+ минут исполнения во время параллельных волн — это вызвано тем, что несколько агентов одновременно триггерят build-инструменты. GSD обрабатывает это автоматически с v1.26 — параллельные агенты используют `--no-verify` на коммитах, а оркестратор запускает хуки один раз после каждой волны. Если у тебя более старая версия, добавь в `CLAUDE.md` проекта:

```markdown
## Git Commit Rules for Agents
All subagent/executor commits MUST use `--no-verify`.
```

Чтобы полностью отключить параллельное исполнение: `/gsd-settings` → поставить `parallelization.enabled` в `false`.

### Windows: установщик падает на защищённых директориях

Если установщик падает с `EPERM: operation not permitted, scandir` на Windows, это вызвано OS-защищёнными директориями (например, профилями Chromium-браузеров). Починено с v1.24 — обнови до последней версии. Как workaround: временно переименуй проблемную директорию перед запуском установщика.

---

## Быстрый справочник восстановления

| Проблема                                  | Решение                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------ |
| Потерян контекст / новая сессия           | `/gsd-resume-work` или `/gsd-progress`                                   |
| Фаза пошла не так                         | `git revert` коммиты фазы, потом пере-планировать                        |
| Нужно сменить scope                       | `/gsd-phase` (дефолт), `/gsd-phase --insert` или `/gsd-phase --remove`   |
| Что-то сломалось                          | `/gsd-debug "описание"` (добавь `--diagnose` для анализа без фиксов)     |
| STATE.md рассинхронизирован               | `state validate` потом `state sync`                                      |
| Состояние воркфлоу выглядит повреждённым  | `/gsd-forensics`                                                         |
| Быстрый точечный фикс                     | `/gsd-quick`                                                             |
| План не совпадает с твоим видением        | `/gsd-discuss-phase [N]`, потом пере-планировать                         |
| Стоимость растёт                          | `/gsd-config --profile budget` и `/gsd-settings` чтобы отключить агентов |
| Обновление сломало локальные изменения    | `/gsd-update --reapply`                                                  |
| Нужен summary сессии для стейкхолдера     | `/gsd-pause-work --report`                                               |
| Не знаешь какой следующий шаг             | `/gsd-progress --next`                                                   |
| Build-ошибки параллельного исполнения     | Обновить GSD или поставить `parallelization.enabled: false`              |

---

## Структура файлов проекта

Для справки — вот что GSD создаёт в твоём проекте:

```
.planning/
  PROJECT.md              # Видение и контекст проекта (всегда загружается)
  REQUIREMENTS.md         # Скоупленные v1/v2 требования с ID
  ROADMAP.md              # Разбивка по фазам с отслеживанием статуса
  STATE.md                # Решения, блокеры, память сессии
  config.json             # Конфигурация воркфлоу
  MILESTONES.md           # Архив завершённых майлстоунов
  HANDOFF.json            # Структурированный хэндофф сессии (от /gsd-pause-work)
  research/               # Domain research от /gsd-new-project
  reports/                # Отчёты сессий (от /gsd-pause-work --report)
  todos/
    pending/              # Захваченные идеи, ждущие работы
    done/                 # Завершённые todo
  debug/                  # Активные debug-сессии
    resolved/             # Архивные debug-сессии
  spikes/                 # Эксперименты осуществимости (от /gsd-spike)
    NNN-name/             # Код эксперимента + README с вердиктом
    MANIFEST.md           # Индекс всех spike'ов
  sketches/               # HTML-mockups (от /gsd-sketch)
    NNN-name/             # index.html (2-3 варианта) + README
    themes/
      default.css         # Общие CSS-переменные для всех скетчей
    MANIFEST.md           # Индекс всех скетчей с победителями
  codebase/               # Brownfield codebase mapping (от /gsd-map-codebase)
  phases/
    XX-phase-name/
      XX-YY-PLAN.md       # Атомарные execution-планы
      XX-YY-SUMMARY.md    # Outcome'ы и решения исполнения
      CONTEXT.md          # Твои предпочтения реализации
      RESEARCH.md         # Находки research'а экосистемы
      VERIFICATION.md     # Результаты постфактум-верификации
      XX-UI-SPEC.md       # UI дизайн-контракт (от /gsd-ui-phase)
      XX-UI-REVIEW.md     # Оценки визуального аудита (от /gsd-ui-review)
  ui-reviews/             # Скриншоты от /gsd-ui-review (gitignored)
```
