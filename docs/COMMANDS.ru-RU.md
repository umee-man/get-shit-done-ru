# GSD — Справочник команд

> Синтаксис команд, флаги, опции и примеры для стабильных команд. Детали фич — в [Feature Reference](FEATURES.md). Сквозные разборы воркфлоу — в [User Guide](USER-GUIDE.ru-RU.md).

---

## Синтаксис команд

- **Claude Code / Copilot / OpenCode / Kilo:** `/gsd-command-name [args]` (форма через дефис)
- **Gemini CLI:** `/gsd:command-name [args]` (форма через двоеточие — Gemini неймспейсит команды под `gsd:`)
- **Codex:** `$gsd-command-name [args]`

Формы с дефисом и двоеточием — это *runtime-специфичные написания одной и той же команды*. На каком бы рантайме ты ни был, установщик пишет правильную форму в директорию команд твоего рантайма.

---

## Namespace мета-скиллы

Шесть namespace-роутеров поставляются как точки входа первого этапа в v1.40. Они держат токен-стоимость eager skill-листинга низкой (~120 токенов на 6 роутеров против ~2150 для плоского листинга 86 скиллов), при этом весь surface остаётся напрямую вызываемым. Модель выбирает namespace, потом роутит на конкретный sub-скилл. См. [#2792](https://github.com/gsd-build/get-shit-done/issues/2792).

| Команда | Куда роутит |
|---------|-------------|
| `/gsd-workflow` | Phase pipeline — discuss / plan / execute / verify / phase / progress |
| `/gsd-project` | Жизненный цикл проекта — milestones, audits, summary |
| `/gsd-quality` | Quality gates — code review, debug, audit, security, eval, ui |
| `/gsd-context` | Интеллект кодовой базы — map, graphify, docs, learnings |
| `/gsd-manage` | Менеджмент — config, workspace, workstreams, thread, update, ship, inbox |
| `/gsd-ideate` | Exploration & capture — explore, sketch, spike, spec, capture |

Namespace-скиллы **аддитивны** — каждая существующая конкретная команда (например `/gsd-plan-phase`, `/gsd-code-review --fix`) по-прежнему вызывается напрямую.

---

## Команды основного воркфлоу

### `/gsd-new-project`

Инициализировать новый проект с глубоким сбором контекста.

| Флаг | Описание |
|------|----------|
| `--auto @file.md` | Авто-извлечение из документа, пропуск интерактивных вопросов |

**Предусловия:** Нет существующего `.planning/PROJECT.md`
**Производит:** `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md`, `STATE.md`, `config.json`, `research/`, `CLAUDE.md`

```bash
/gsd-new-project                    # Интерактивный режим
/gsd-new-project --auto @prd.md     # Авто-извлечение из PRD
```

---

### `/gsd-workspace`

Управление workspace'ами GSD — создание, листинг, удаление изолированных окружений с копиями реп и независимыми `.planning/`.

| Флаг | Описание |
|------|----------|
| `--new` | Создать новый workspace (используй с `--name`, `--repos` и т.д.) |
| `--list` | Список активных workspace'ов GSD и их статус |
| `--remove <name>` | Удалить workspace и почистить git worktree'ы |
| `--name <name>` | Имя workspace'а (используется с `--new`) |
| `--repos repo1,repo2` | Пути или имена реп через запятую (с `--new`) |
| `--path /target` | Целевая директория (дефолт: `~/gsd-workspaces/<name>`) |
| `--strategy worktree\|clone` | Стратегия копирования (дефолт: `worktree`) |
| `--branch <name>` | Ветка для чекаута (дефолт: `workspace/<name>`) |
| `--auto` | Пропустить интерактивные вопросы |

**Кейсы:**
- Multi-repo: работа над подмножеством реп с изолированным состоянием GSD
- Изоляция фичи: `--repos .` создаёт worktree текущей репы

**Производит:** `WORKSPACE.md`, `.planning/`, копии реп (worktree'ы или клоны)

```bash
/gsd-workspace --new --name feature-b --repos hr-ui,ZeymoAPI
/gsd-workspace --new --name feature-b --repos . --strategy worktree  # Изоляция той же репы
/gsd-workspace --list
/gsd-workspace --remove feature-b
```

---

### `/gsd-discuss-phase`

Собрать контекст фазы через адаптивные вопросы до планирования.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы (по умолчанию — текущая) |

| Флаг | Описание |
|------|----------|
| `--all` | Пропустить выбор области — обсуждать все серые зоны интерактивно (без авто-продвижения) |
| `--auto` | Авто-выбрать рекомендованные дефолты на все вопросы |
| `--batch` | Группировать вопросы для batch-приёма вместо one-by-one |
| `--analyze` | Добавить trade-off анализ во время обсуждения |
| `--power` | Bulk-ответы на вопросы из подготовленного файла ответов |
| `--assumptions` | Всплыть предположения Claude по реализации фазы без интерактивной сессии |

**Предусловия:** `.planning/ROADMAP.md` существует
**Производит:** `{phase}-CONTEXT.md`, `{phase}-DISCUSSION-LOG.md` (audit trail)

```bash
/gsd-discuss-phase 1                # Интерактивное обсуждение фазы 1
/gsd-discuss-phase 1 --all          # Обсудить все серые зоны без шага выбора
/gsd-discuss-phase 3 --auto         # Авто-выбрать дефолты для фазы 3
/gsd-discuss-phase --batch          # Batch-режим для текущей фазы
/gsd-discuss-phase 2 --analyze      # Обсуждение с trade-off анализом
/gsd-discuss-phase 1 --power        # Bulk-ответы из файла
/gsd-discuss-phase 3 --assumptions  # Всплыть предположения Claude до планирования
```

---

### `/gsd-ui-phase`

Сгенерировать UI дизайн-контракт для фронтенд-фаз.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы (по умолчанию — текущая) |

**Предусловия:** `.planning/ROADMAP.md` существует, у фазы есть frontend/UI работа
**Производит:** `{phase}-UI-SPEC.md`

```bash
/gsd-ui-phase 2                     # Дизайн-контракт для фазы 2
```

---

### `/gsd-plan-phase`

Исследовать, спланировать и верифицировать фазу.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы (по умолчанию — следующая незапланированная) |

| Флаг | Описание |
|------|----------|
| `--auto` | Пропустить интерактивные подтверждения |
| `--research` | Принудительно перезапустить research даже если RESEARCH.md существует |
| `--skip-research` | Пропустить шаг доменного research'а |
| `--research-phase <N>` | Research-only режим: спавнить researcher'а для фазы `<N>`, написать RESEARCH.md, выйти до планировщика. Заменяет удалённую отдельную команду `gsd-research-phase` (#3042). |
| `--view` | Research-only модификатор: с `--research-phase` печатает существующий RESEARCH.md в stdout и выходит (без спавна) |
| `--gaps` | Режим закрытия пробелов (читает VERIFICATION.md, пропускает research) |
| `--skip-verify` | Пропустить цикл верификации plan-checker'а |
| `--prd <file>` | Использовать PRD-файл вместо discuss-phase для контекста |
| `--ingest <path-or-glob>` | Использовать ADR-файл(ы) вместо discuss-phase для синтеза контекста |
| `--ingest-format <auto\|nygard\|madr\|narrative>` | Опциональный override формата ADR-парсера для `--ingest` |
| `--reviews` | Перепланировать с фидбеком из cross-AI ревью в REVIEWS.md |
| `--validate` | Запустить валидацию состояния до начала планирования |
| `--bounce` | Запустить внешнюю валидацию plan bounce после планирования (использует `workflow.plan_bounce_script`) |
| `--skip-bounce` | Пропустить plan bounce даже если включено в конфиге |

**Предусловия:** `.planning/ROADMAP.md` существует
**Производит:** `{phase}-RESEARCH.md`, `{phase}-{N}-PLAN.md`, `{phase}-VALIDATION.md`

**Research-only режим (`--research-phase <N>`):**
- Без модификатора: предлагает `update / view / skip` если RESEARCH.md уже существует
- С `--research`: force-refresh — перезапуск researcher'а безусловно, без промпта
- С `--view`: печать существующего RESEARCH.md в stdout, без спавна. Ошибка если RESEARCH.md отсутствует.

**Package Legitimacy Gate (v1.51):**
Когда researcher рекомендует внешние пакеты, он запускает `slopcheck install <pkg> --json` на каждом и пишет таблицу `## Package Legitimacy Audit` в RESEARCH.md, фиксируя Registry, Age, Downloads, Source Repo и вердикт slopcheck. Вердикты:

- `[SLOP]` — пакет полностью удалён из RESEARCH.md; никогда не доходит до планировщика
- `[SUS]` — пакет помечен; планировщик вставляет `checkpoint:human-verify` перед install-задачей
- `[OK]` — пакет одобрен; чекпоинт не добавляется

Пакеты, найденные через WebSearch, тегаются `[ASSUMED]` (не `[VERIFIED]`) и обрабатываются как `[SUS]` — получают человеческий чекпоинт перед установкой. Если `slopcheck` не получается установить, каждый рекомендованный пакет тегается `[ASSUMED]` и гейтится.

См. [Package Legitimacy Gate в User Guide](USER-GUIDE.ru-RU.md#gate-легитимности-пакетов-v151) для полного формата чекпоинта, таблицы вердиктов и диагностики.

```bash
/gsd-plan-phase 1                              # Research + plan + verify фазы 1
/gsd-plan-phase 3 --skip-research              # План без research'а (знакомый домен)
/gsd-plan-phase --auto                         # Неинтерактивное планирование
/gsd-plan-phase 2 --validate                   # Валидация состояния до планирования
/gsd-plan-phase 1 --bounce                     # План + внешняя bounce-валидация
/gsd-plan-phase 2 --ingest docs/adr/0010.md   # ADR express-путь для синтеза контекста
/gsd-plan-phase 2 --ingest 'docs/adr/00*.md' --ingest-format auto
/gsd-plan-phase --research-phase 4             # Research только на фазе 4 (промпт если RESEARCH.md существует)
/gsd-plan-phase --research-phase 4 --view      # Печать существующего RESEARCH.md, без спавна
/gsd-plan-phase --research-phase 4 --research  # Force-refresh research, без промпта
```

---

### `/gsd-plan-review-convergence`

Cross-AI plan convergence loop — перепланирование с фидбеком ревью пока не останется HIGH-замечаний. Запускает циклы `plan-phase → review → replan → re-review` (макс 3 цикла по умолчанию). Спавнит изолированных агентов для планирования и ревью; оркестратор управляет циклом, считает HIGH-замечания, детектит застой и эскалирует.

| Аргумент / Флаг | Обязательный | Описание |
|-----------------|--------------|----------|
| `N` | **Да** | Номер фазы для планирования и ревью |
| `--codex` / `--gemini` / `--claude` / `--opencode` | Нет | Выбор одного ревьюера |
| `--all` | Нет | Запустить всех сконфигурированных ревьюеров параллельно |
| `--max-cycles N` | Нет | Override cap'а циклов (дефолт 3) |

**Поведение выхода:** Цикл завершается когда HIGH-счёт достигает нуля. Детекция застоя предупреждает когда HIGH-счёт не убывает между циклами. Gate эскалации спрашивает пользователя продолжить или ревьюить вручную когда достигнут `--max-cycles` с открытыми HIGH-замечаниями.

```bash
/gsd-plan-review-convergence 3                    # Дефолтные ревьюеры, 3 цикла
/gsd-plan-review-convergence 3 --codex            # Только Codex-ревью
/gsd-plan-review-convergence 3 --all --max-cycles 5
```

---

### `/gsd-ultraplan-phase`

**[BETA]** Выгрузить plan-фазу в облако ultraplan от Claude Code; ревью в браузере и импорт обратно. План рисуется удалённо, чтобы терминал оставался свободным; ревьюишь inline-комментарии в браузере, потом импортируешь финализированный план обратно в `.planning/` через `/gsd-import`.

| Флаг | Обязательный | Описание |
|------|--------------|----------|
| `N` | **Да** | Номер фазы для удалённого планирования |

**Изоляция:** Намеренно отделено от `/gsd-plan-phase`, чтобы upstream-изменения ultraplan не могли влиять на основной planning-пайплайн.

```bash
/gsd-ultraplan-phase 4                  # Выгрузить планирование фазы 4
```

---

### `/gsd-execute-phase`

Исполнить все планы фазы с волновой параллелизацией или запустить конкретную волну.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | **Да** | Номер фазы для исполнения |
| `--wave N` | Нет | Исполнить только волну `N` в фазе |
| `--validate` | Нет | Валидация состояния до начала исполнения |
| `--cross-ai` | Нет | Делегировать исполнение внешнему AI CLI (использует `workflow.cross_ai_command`) |
| `--no-cross-ai` | Нет | Принудительно локальное исполнение, даже если cross-AI включён в конфиге |

**Предусловия:** В фазе есть PLAN.md файлы
**Производит:** per-plan `{phase}-{N}-SUMMARY.md`, git-коммиты и `{phase}-VERIFICATION.md` когда фаза полностью завершена

**Сбои установки пакетов (v1.51):** Если install-шаг плана падает, executor поднимает `checkpoint:human-verify` и останавливается. Он не автоустанавливает похоже названную альтернативу. Это намеренно — молчаливая подмена имён пакетов — это как раз способ распространения slopsquatting'а. Отвечай на чекпоинт после проверки пакета на странице реестра.

```bash
/gsd-execute-phase 1                # Исполнить фазу 1
/gsd-execute-phase 1 --wave 2       # Исполнить только волну 2
/gsd-execute-phase 1 --validate     # Валидация состояния до исполнения
/gsd-execute-phase 2 --cross-ai     # Делегировать фазу 2 внешнему AI CLI
```

---

### `/gsd-verify-work`

User acceptance testing с авто-диагностикой.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы (по умолчанию — последняя исполненная) |

**Предусловия:** Фаза была исполнена
**Производит:** `{phase}-UAT.md`, fix-планы если найдены проблемы

```bash
/gsd-verify-work 1                  # UAT для фазы 1
```

---

### `/gsd-ship`

Создать PR из завершённой работы фазы с авто-генерируемым телом.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы или версия майлстоуна (например, `4` или `v1.0`) |
| `--draft` | Нет | Создать как черновой PR |

**Предусловия:** Фаза верифицирована (`/gsd-verify-work` прошёл), CLI `gh` установлен и авторизован
**Производит:** GitHub PR с богатым телом из планировочных артефактов, STATE.md обновлён

```bash
/gsd-ship 4                         # Зашиппить фазу 4
/gsd-ship 4 --draft                 # Зашиппить как черновой PR
```

**Тело PR включает:**
- Цель фазы из ROADMAP.md
- Сводку изменений из SUMMARY.md файлов
- Покрытые требования (REQ-ID)
- Статус верификации
- Ключевые решения
- Опциональные сконфигурированные секции в стиле PRD из `ship.pr_body_sections`

См. [Custom PR Body Sections](ship-pr-body-sections.md) для онбординга, примеров и правил валидации.

---

### `/gsd-ui-review`

Ретроактивный 6-компонентный визуальный аудит имплементированного фронтенда.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы (по умолчанию — последняя исполненная) |

**Предусловия:** В проекте есть фронтенд-код (работает standalone, GSD-проект не нужен)
**Производит:** `{phase}-UI-REVIEW.md`, скриншоты в `.planning/ui-reviews/`

```bash
/gsd-ui-review                      # Аудит текущей фазы
/gsd-ui-review 3                    # Аудит фазы 3
```

---

### `/gsd-audit-uat`

Cross-фазовый аудит всех висящих UAT и verification айтемов.

**Предусловия:** Хотя бы одна фаза исполнена с UAT или верификацией
**Производит:** Классифицированный аудит-отчёт с планом ручных тестов

```bash
/gsd-audit-uat
```

---

### `/gsd-audit-milestone`

Проверить, что майлстоун выполнил свой definition of done.

**Предусловия:** Все фазы исполнены
**Производит:** Аудит-отчёт с gap-анализом

```bash
/gsd-audit-milestone
```

---

### `/gsd-complete-milestone`

Заархивировать майлстоун, затегать релиз.

**Предусловия:** Аудит майлстоуна завершён (рекомендуется)
**Производит:** Запись в `MILESTONES.md`, git-тег

```bash
/gsd-complete-milestone
```

---

### `/gsd-milestone-summary`

Сгенерировать комплексное summary проекта из артефактов майлстоуна для онбординга команды и ревью.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `version` | Нет | Версия майлстоуна (по умолчанию — текущий/последний) |

**Предусловия:** Хотя бы один завершённый или активный майлстоун
**Производит:** `.planning/reports/MILESTONE_SUMMARY-v{version}.md`

**Summary включает:**
- Обзор, архитектурные решения, разбивку по фазам
- Ключевые решения и trade-off'ы
- Покрытие требований
- Технический долг и отложенные айтемы
- Гайд для начала работы новых членов команды
- Интерактивное Q&A предлагается после генерации

```bash
/gsd-milestone-summary                # Summary текущего майлстоуна
/gsd-milestone-summary v1.0           # Summary конкретного майлстоуна
```

---

### `/gsd-new-milestone`

Начать следующий цикл версии.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `name` | Нет | Имя майлстоуна |
| `--reset-phase-numbers` | Нет | Перезапустить новый майлстоун с фазы 1 и заархивировать старые phase-директории до roadmap'а |

**Предусловия:** Предыдущий майлстоун завершён
**Производит:** Обновлённый `PROJECT.md`, новый `REQUIREMENTS.md`, новый `ROADMAP.md`

```bash
/gsd-new-milestone                  # Интерактивно
/gsd-new-milestone "v2.0 Mobile"    # Именованный майлстоун
/gsd-new-milestone --reset-phase-numbers "v2.0 Mobile"  # Перезапуск нумерации с 1
```

---

## Команды управления фазами

### `/gsd-phase`

CRUD для фаз в ROADMAP.md — добавить, вставить, удалить, отредактировать фазы одной консолидированной командой.

| Флаг | Описание |
|------|----------|
| (нет) | Дописать новую целочисленную фазу в конец текущего майлстоуна |
| `--insert <N>` | Вставить срочную работу как десятичную фазу (например, 3.1) после фазы N |
| `--remove <N>` | Удалить будущую фазу и перенумеровать следующие |
| `--edit <N>` | Отредактировать любое поле существующей фазы на месте |
| `--force` | Разрешить редактирование in-progress или завершённых фаз (с `--edit`) |

**Предусловия:** `.planning/ROADMAP.md` существует
**Производит:** Обновлённый ROADMAP.md

```bash
/gsd-phase "Add authentication system"          # Дописать новую фазу с описанием
/gsd-phase --insert 3 "Fix auth race condition" # Вставить между фазами 3 и 4 → создаёт 3.1
/gsd-phase --remove 7               # Удалить фазу 7, перенумеровать 8→7, 9→8 и т.д.
/gsd-phase --edit 5                 # Редактировать любое поле фазы 5
/gsd-phase --edit 5 --force         # Редактировать фазу 5 даже если в работе или завершена
```

---

### `/gsd-validate-phase`

Ретроактивно проаудить и закрыть пробелы валидации Nyquist.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы |

```bash
/gsd-validate-phase 2               # Аудит покрытия тестами для фазы 2
```

---

## Навигационные команды

### `/gsd-progress`

Показать статус, следующие шаги и автоматически продвинуться к следующему логическому шагу воркфлоу. Читает состояние проекта и определяет подходящее действие.

| Флаг | Описание |
|------|----------|
| `--next` | Автоматически продвинуться к следующему логическому шагу без выбора маршрута вручную |
| `--do "task description"` | Анализ freeform intent'а и диспатч в самую подходящую команду GSD |
| `--forensic` | Добавить 6-проверочный integrity-аудит после стандартного отчёта (консистентность STATE, осиротевшие handoff'ы, дрейф отложенного скоупа, memory-флагнутые висящие работы, блокирующие todos, незакоммиченный код) |

**Поведение авто-роутинга (`--next`):**
- Нет проекта → предлагает `/gsd-new-project`
- Фаза нуждается в обсуждении → запускает `/gsd-discuss-phase`
- Фаза нуждается в планировании → запускает `/gsd-plan-phase`
- Фаза нуждается в исполнении → запускает `/gsd-execute-phase`
- Фаза нуждается в верификации → запускает `/gsd-verify-work`
- Все фазы завершены → предлагает `/gsd-complete-milestone`

```bash
/gsd-progress                       # "Где я? Что дальше?" с авто-роутингом
/gsd-progress --next                # Продвинуться к следующему шагу автоматически
/gsd-progress --do "fix the auth bug"  # Диспатч freeform intent'а в лучшую команду GSD
/gsd-progress --forensic            # Стандартный отчёт + integrity-аудит
```

### `/gsd-resume-work`

Восстановить полный контекст из последней сессии.

```bash
/gsd-resume-work                    # После сброса контекста или новой сессии
```

### `/gsd-pause-work`

Сохранить хэндофф контекста при остановке посреди фазы.

| Флаг | Описание |
|------|----------|
| `--report` | Сгенерировать post-session summary в `.planning/reports/` — коммиты, изменения файлов, прогресс по фазам |

```bash
/gsd-pause-work                     # Создаёт continue-here.md
/gsd-pause-work --report            # Создаёт continue-here.md + отчёт сессии
```

### `/gsd-manager`

Интерактивный командный центр для управления несколькими фазами из одного терминала.

**Предусловия:** `.planning/ROADMAP.md` существует
**Поведение:**
- Дашборд всех фаз с визуальными индикаторами статуса
- Рекомендует оптимальные следующие действия на основе зависимостей и прогресса
- Диспатчит работу: discuss идёт инлайн, plan/execute — как фоновые агенты
- Спроектирован для power-юзеров, параллелящих работу по фазам из одного терминала
- Поддерживает per-шаг passthrough-флаги через конфиг `manager.flags` (см. [Configuration](CONFIGURATION.md#manager-passthrough-flags))

```bash
/gsd-manager                        # Открыть дашборд командного центра
/gsd-manager --analyze-deps         # Сканировать фазы ROADMAP на связи зависимостей до параллельного исполнения
```

**Checkpoint heartbeat'ы (#2410):**

Фоновые запуски `execute-phase` эмитят маркеры `[checkpoint]` на каждой границе волны и плана, чтобы Claude API SSE-стрим никогда не простаивал достаточно долго, чтобы триггернуть `Stream idle timeout - partial response received` на multi-plan фазах. Формат:

```
[checkpoint] phase {N} wave {W}/{M} starting, {count} plan(s), {P}/{Q} plans done
[checkpoint] phase {N} wave {W}/{M} plan {plan_id} starting ({P}/{Q} plans done)
[checkpoint] phase {N} wave {W}/{M} plan {plan_id} complete ({P}/{Q} plans done)
[checkpoint] phase {N} wave {W}/{M} complete, {P}/{Q} plans done ({ok}/{count} ok)
```

Если фоновая фаза падает на полпути, грепни в transcript'е `[checkpoint]` чтобы увидеть последнюю подтверждённую границу. Background-completion handler менеджера использует эти маркеры чтобы отрепортить частичный прогресс когда агент падает.

**Manager passthrough-флаги:**

Конфигурируй per-step флаги в `.planning/config.json` под `manager.flags`. Эти флаги дописываются к каждой диспатченной команде:

```json
{
  "manager": {
    "flags": {
      "discuss": "--auto",
      "plan": "--skip-research",
      "execute": "--validate"
    }
  }
}
```

---

### `/gsd-help`

Показать все команды и гайд по использованию.

```bash
/gsd-help                           # Быстрый справочник
```

---

## Утилитарные команды

### `/gsd-explore`

Сократическая ideation-сессия — провести идею через зондирующие вопросы, опционально спавнить research, потом сроутить output в правильный артефакт GSD (заметки, todos, seeds, research-вопросы, требования или новая фаза).

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `topic` | Нет | Тема для исследования (например, `/gsd-explore authentication strategy`) |

```bash
/gsd-explore                        # Open-ended ideation-сессия
/gsd-explore authentication strategy  # Изучить конкретную тему
```

---

### `/gsd-undo`

Безопасный git revert — откат коммитов фазы или плана GSD через phase-манифест с проверкой зависимостей и confirmation-gate.

| Флаг | Обязательный | Описание |
|------|--------------|----------|
| `--last N` | (один из трёх обязателен) | Показать недавние GSD-коммиты для интерактивного выбора |
| `--phase NN` | (один из трёх обязателен) | Откатить все коммиты фазы |
| `--plan NN-MM` | (один из трёх обязателен) | Откатить все коммиты конкретного плана |

**Безопасность:** Проверяет зависимые фазы/планы перед откатом; всегда показывает confirmation-gate.

```bash
/gsd-undo --last 5                  # Выбор из 5 последних GSD-коммитов
/gsd-undo --phase 03                # Откатить все коммиты фазы 3
/gsd-undo --plan 03-02              # Откатить коммиты плана 02 фазы 3
```

---

### `/gsd-import`

Принять внешний файл плана в planning-систему GSD с детекцией конфликтов против решений в `PROJECT.md` до того как что-либо записано.

| Флаг | Обязательный | Описание |
|------|--------------|----------|
| `--from <filepath>` | Да (или `--from-gsd2`) | Путь к внешнему файлу плана для импорта |
| `--from-gsd2` | Да (или `--from`) | Обратная миграция GSD-2 (`.gsd/`) проекта обратно в формат GSD v1 (`.planning/`) |
| `--path <dir>` | Нет | С `--from-gsd2`: путь к директории GSD-2 проекта (дефолт — текущая) |

**Процесс:** Детект конфликтов → промпт на резолв → запись как GSD PLAN.md → валидация через `gsd-plan-checker`

```bash
/gsd-import --from /tmp/team-plan.md    # Импорт и валидация внешнего плана
/gsd-import --from-gsd2                # Миграция с GSD-2 обратно в v1 (текущая дир.)
/gsd-import --from-gsd2 --path ~/old-project  # Миграция с другого пути
```

---

### `/gsd-ingest-docs`

Bootstrap или мердж `.planning/` сетапа из существующих ADR, PRD, SPEC и доков репы. Запускает параллельную классификацию (`gsd-doc-classifier`) плюс синтез с правилами приоритета и детекцией циклов (`gsd-doc-synthesizer`). Производит трёхбакетный конфликт-репорт (`INGEST-CONFLICTS.md`: auto-resolved, competing-variants, unresolved-blockers) и жёстко блокирует на противоречиях LOCKED-vs-LOCKED ADR.

| Аргумент / Флаг | Обязательный | Описание |
|-----------------|--------------|----------|
| `path` | Нет | Целевая директория для скана (дефолт — корень репы) |
| `--mode new\|merge` | Нет | Override авто-детекта (дефолты: `new` если `.planning/` нет, `merge` если есть) |
| `--manifest <file>` | Нет | YAML с `{path, type, precedence?}` на док; переопределяет эвристическую классификацию |
| `--resolve auto` | Нет | Режим резолва конфликтов (v1: только `auto`; `interactive` зарезервирован) |

**Лимиты:** v1 кап 50 доков на вызов. Извлекает общий контракт детекции конфликтов в `references/doc-conflict-engine.md`, которым также пользуется `/gsd-import`.

```bash
/gsd-ingest-docs                            # Скан корня репы, авто-детект режима
/gsd-ingest-docs docs/                      # Только под docs/
/gsd-ingest-docs --manifest ingest.yaml     # Явный precedence-манифест
```

---

### `/gsd-quick`

Исполнить разовую задачу с гарантиями GSD.

| Флаг | Описание |
|------|----------|
| `--full` | Включить полный quality-пайплайн — обсуждение + research + plan-checking + верификация |
| `--validate` | Только plan-checking (макс 2 итерации) + post-execution верификация; без обсуждения и research'а |
| `--discuss` | Лёгкое pre-planning обсуждение |
| `--research` | Спавнить фокусного researcher'а перед планированием |

Гранулярные флаги композируются: `--discuss --research --validate` эквивалентно `--full`.

| Подкоманда | Описание |
|------------|----------|
| `list` | Список всех quick-задач со статусом |
| `status <slug>` | Статус конкретной quick-задачи |
| `resume <slug>` | Возобновить конкретную quick-задачу по slug'у |

```bash
/gsd-quick                          # Базовая quick-задача
/gsd-quick --discuss --research     # Обсуждение + research + планирование
/gsd-quick --validate               # Только plan-checking + верификация
/gsd-quick --full                   # Полный quality-пайплайн
/gsd-quick list                     # Список всех quick-задач
/gsd-quick status my-task-slug      # Статус quick-задачи
/gsd-quick resume my-task-slug      # Возобновить quick-задачу
```

### `/gsd-autonomous`

Запустить все оставшиеся фазы автономно.

| Флаг | Описание |
|------|----------|
| `--from N` | Начать с конкретного номера фазы |
| `--to N` | Остановиться после конкретного номера фазы |
| `--interactive` | Лёгкий контекст с пользовательским вводом |

```bash
/gsd-autonomous                     # Запустить все оставшиеся фазы
/gsd-autonomous --from 3            # Начать с фазы 3
/gsd-autonomous --to 5              # Запустить до фазы 5 включительно
/gsd-autonomous --from 3 --to 5     # Фазы 3–5
```

### `/gsd-debug`

Систематический дебаггинг с persistent-стейтом.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `description` | Нет | Описание бага |

| Флаг | Описание |
|------|----------|
| `--diagnose` | Diagnosis-only режим — расследовать без попыток фикса |

**Подкоманды:**
- `/gsd-debug list` — Список всех активных debug-сессий со статусом, гипотезой и следующим действием
- `/gsd-debug status <slug>` — Полное summary сессии (счёт Evidence, счёт Eliminated, Resolution, TDD чекпоинт) без спавна агента
- `/gsd-debug continue <slug>` — Возобновить конкретную сессию по slug'у (всплывает Current Focus, потом спавн continuation-агента)
- `/gsd-debug [--diagnose] <description>` — Начать новую debug-сессию (существующее поведение; `--diagnose` останавливается на root cause без применения фикса)

**TDD режим:** Когда `tdd_mode: true` в `.planning/config.json`, debug-сессии требуют написания и верификации падающего теста до применения любого фикса (red → green → done).

```bash
/gsd-debug "Login button not responding on mobile Safari"
/gsd-debug --diagnose "Intermittent 500 errors on /api/users"
/gsd-debug list
/gsd-debug status auth-token-null
/gsd-debug continue form-submit-500
```

### `/gsd-add-tests`

Сгенерить тесты для завершённой фазы.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | Нет | Номер фазы |

```bash
/gsd-add-tests 2                    # Сгенерить тесты для фазы 2
```

### `/gsd-stats`

Показать статистику проекта.

```bash
/gsd-stats                          # Дашборд метрик проекта
```

### `/gsd-profile-user`

Сгенерить поведенческий профиль разработчика из анализа сессий Claude Code по 8 измерениям (стиль коммуникации, паттерны решений, подход к дебагу, UX-предпочтения, выбор вендоров, триггеры фрустрации, стиль обучения, глубина объяснений). Производит артефакты, персонализирующие ответы Claude.

| Флаг | Описание |
|------|----------|
| `--questionnaire` | Использовать интерактивный опросник вместо анализа сессий |
| `--refresh` | Перепроанализировать сессии и пересгенерить профиль |

**Сгенерированные артефакты:**
- `USER-PROFILE.md` — Полный поведенческий профиль
- Секция профиля в `CLAUDE.md` — Авто-обнаруживается Claude Code

```bash
/gsd-profile-user                   # Анализ сессий и сборка профиля
/gsd-profile-user --questionnaire   # Fallback на интерактивный опросник
/gsd-profile-user --refresh         # Пересгенерить из свежего анализа
```

### `/gsd-health`

Валидировать целостность директории `.planning/`. С `--context` зондирует guard утилизации контекстного окна против порогов 60 % / 70 % (добавлено в v1.40.0, [#2792](https://github.com/gsd-build/get-shit-done/issues/2792)).

| Флаг | Описание |
|------|----------|
| `--repair` | Автофикс recoverable-проблем |
| `--context` | Зондировать утилизацию контекстного окна; warning на 60 %, critical на 70 % |

```bash
/gsd-health                         # Проверка целостности
/gsd-health --repair                # Проверка и фикс
/gsd-health --context               # Triage утилизации контекста
```

### `/gsd-cleanup`

Заархивировать накопленные phase-директории от завершённых майлстоунов.

```bash
/gsd-cleanup
```

---

## Команды Spike и Sketch

### `/gsd-spike`

Запустить 2–5 фокусных экспериментов осуществимости до коммита к подходу. Каждый эксперимент использует Given/When/Then-frame, производит исполнимый код и возвращает вердикт VALIDATED / INVALIDATED / PARTIAL.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `idea` | Нет | Технический вопрос или подход для расследования |
| `--quick` | Нет | Пропустить intake-разговор; использовать `idea` напрямую |
| `--wrap-up` | Нет | Упаковать завершённые spike-находки в reusable project-local скилл |

**Производит:** `.planning/spikes/NNN-experiment-name/` с кодом, результатами и README; `.planning/spikes/MANIFEST.md`
**`--wrap-up` производит:** Файл скилла в `.claude/skills/spike-findings-[project]/`

```bash
/gsd-spike                              # Интерактивный intake
/gsd-spike "can we stream LLM tokens through SSE"
/gsd-spike --quick websocket-vs-polling
/gsd-spike --wrap-up                    # Упаковать находки в reusable-скилл
```

---

### `/gsd-sketch`

Исследовать дизайн-направления через одноразовые HTML-моки до коммита к реализации. Производит 2–3 варианта на дизайн-вопрос для прямого браузерного сравнения.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `idea` | Нет | UI дизайн-вопрос или направление для исследования |
| `--quick` | Нет | Пропустить mood-intake; использовать `idea` напрямую |
| `--text` | Нет | Text-mode fallback — заменить интерактивные промпты нумерованными списками (для не-Claude рантаймов) |
| `--wrap-up` | Нет | Упаковать решения победившего sketch'а в reusable project-local скилл |

**Производит:** `.planning/sketches/NNN-descriptive-name/index.html` (2–3 интерактивных варианта), `README.md`, общий `themes/default.css`; `.planning/sketches/MANIFEST.md`
**`--wrap-up` производит:** Файл скилла в `.claude/skills/sketch-findings-[project]/`

```bash
/gsd-sketch                             # Интерактивный mood-intake
/gsd-sketch "dashboard layout"
/gsd-sketch --quick "sidebar navigation"
/gsd-sketch --text "onboarding flow"    # Не-Claude рантайм
/gsd-sketch --wrap-up                   # Упаковать победивший sketch в скилл
```

---

## Команды диагностики

### `/gsd-forensics`

Post-mortem расследование для упавших GSD-воркфлоу — диагностирует что пошло не так.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `description` | Нет | Описание проблемы (промпт если пропущено) |

**Предусловия:** Директория `.planning/` существует
**Производит:** `.planning/forensics/report-{timestamp}.md`

**Расследование покрывает:**
- Анализ git-истории (недавние коммиты, застрявшие паттерны, временные пробелы)
- Целостность артефактов (ожидаемые файлы для завершённых фаз)
- Аномалии STATE.md и история сессии
- Незакоммиченную работу, конфликты, брошенные изменения
- Не менее 4 типов аномалий (застрявший цикл, отсутствующие артефакты, брошенная работа, краш/прерывание)
- Создание GitHub issue предлагается если есть actionable-находки

```bash
/gsd-forensics                              # Интерактивно — промпт по проблеме
/gsd-forensics "Phase 3 execution stalled"  # С описанием проблемы
```

---

### `/gsd-extract-learnings`

Извлечь reusable паттерны, анти-паттерны и архитектурные решения из завершённой работы фазы.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | **Да** | Номер фазы для извлечения уроков |

| Флаг | Описание |
|------|----------|
| `--all` | Извлечь уроки из всех завершённых фаз |
| `--format` | Формат вывода: `markdown` (дефолт), `json` |

**Предусловия:** Фаза исполнена (есть SUMMARY.md файлы)
**Производит:** `.planning/learnings/{phase}-LEARNINGS.md`

**Извлекает:**
- Архитектурные решения и их обоснование
- Паттерны, сработавшие хорошо (reusable в будущих фазах)
- Анти-паттерны, с которыми столкнулись, и как их решили
- Технологические инсайты
- Наблюдения по производительности и тестированию

```bash
/gsd-extract-learnings 3                    # Извлечь уроки из фазы 3
/gsd-extract-learnings --all                # Извлечь из всех завершённых фаз
```

---

## Управление workstream'ами

### `/gsd-workstreams`

Управление параллельными workstream'ами для конкурентной работы над разными областями майлстоуна.

**Подкоманды:**

| Подкоманда | Описание |
|------------|----------|
| `list` | Список всех workstream'ов со статусом (дефолт если нет подкоманды) |
| `create <name>` | Создать новый workstream |
| `status <name>` | Детальный статус одного workstream'а |
| `switch <name>` | Установить активный workstream |
| `progress` | Сводка прогресса по всем workstream'ам |
| `complete <name>` | Заархивировать завершённый workstream |
| `resume <name>` | Возобновить работу в workstream'е |

**Предусловия:** Активный GSD-проект
**Производит:** Workstream-директории под `.planning/`, отслеживание состояния на workstream

```bash
/gsd-workstreams                    # Список всех workstream'ов
/gsd-workstreams create backend-api # Создать новый workstream
/gsd-workstreams switch backend-api # Установить активный workstream
/gsd-workstreams status backend-api # Детальный статус
/gsd-workstreams progress           # Обзор cross-workstream прогресса
/gsd-workstreams complete backend-api  # Заархивировать завершённый workstream
/gsd-workstreams resume backend-api    # Возобновить работу
```

---

## Команды конфигурации

### `/gsd-settings`

Интерактивная конфигурация переключателей воркфлоу и профиля моделей. Вопросы сгруппированы в шесть визуальных секций:

- **Planning** — Research, Plan Checker, Pattern Mapper, Nyquist, UI Phase, UI Gate, AI Phase
- **Execution** — Verifier, TDD Mode, Code Review, Code Review Depth _(условно — только когда Code Review включён)_, UI Review
- **Docs & Output** — Commit Docs, Skip Discuss, Worktrees
- **Features** — Intel, Graphify
- **Model & Pipeline** — Model Profile, Auto-Advance, Branching
- **Misc** — Context Warnings, Research Qs

Все ответы мерджатся через `gsd-sdk query config-set` в резолвенный путь конфига проекта (`.planning/config.json` для стандартной установки, или `.planning/workstreams/<active>/config.json` когда активен workstream), сохраняя не связанные ключи. После подтверждения пользователь может сохранить полный объект настроек в `~/.gsd/defaults.json`, чтобы будущие `/gsd-new-project` стартовали с того же baseline'а.

```bash
/gsd-settings                       # Интерактивная конфигурация
```

### `/gsd-config`

Конфигурировать настройки GSD интерактивно — переключатели воркфлоу, advanced-ручки, интеграции и профиль моделей — одной консолидированной командой.

| Флаг | Описание |
|------|----------|
| (нет) | Common-case переключатели: model, research, plan_check, verifier, branching |
| `--advanced` | Ручки для power-юзеров: тюнинг планирования, таймауты, branch-шаблоны, cross-AI исполнение, runtime/output |
| `--integrations` | Сторонние API-ключи, code-review CLI routing, инъекция скиллов агентов |
| `--profile <name>` | Быстрое переключение профиля: `quality`, `balanced`, `budget` или `inherit` |

**Секции `--advanced`:**

| Секция | Ключи |
|--------|-------|
| Planning Tuning | `workflow.plan_bounce`, `workflow.plan_bounce_passes`, `workflow.plan_bounce_script`, `workflow.subagent_timeout`, `workflow.inline_plan_threshold` |
| Execution Tuning | `workflow.node_repair`, `workflow.node_repair_budget`, `workflow.auto_prune_state` |
| Discussion Tuning | `workflow.max_discuss_passes` |
| Cross-AI Execution | `workflow.cross_ai_execution`, `workflow.cross_ai_command`, `workflow.cross_ai_timeout` |
| Git Customization | `git.base_branch`, `git.phase_branch_template`, `git.milestone_branch_template` |
| Runtime / Output | `response_language`, `context_window`, `search_gitignored`, `graphify.build_timeout` |

Все ответы мерджатся через `gsd-sdk query config-set`, сохраняя не связанные ключи. API-ключи маскируются (`****<last-4>`) во всём output'е.

```bash
/gsd-config                         # Common-case интерактивный конфиг
/gsd-config --advanced              # Power-user ручки (шестисекционный промпт)
/gsd-config --integrations          # API-ключи, review-CLI routing, скиллы агентов
/gsd-config --profile budget        # Переключиться на budget-профиль
/gsd-config --profile quality       # Переключиться на quality-профиль
```

Полная схема и дефолты — в [CONFIGURATION.md](CONFIGURATION.md).

### `/gsd-surface`

Переключать, какие скиллы поверхностно доступны — применить профиль, листинг или отключить кластер без переустановки.

| Подкоманда | Описание |
|------------|----------|
| `list` | Показать включённые и отключённые кластеры и скиллы |
| `status` | Алиас `list` + сводка по токен-стоимости |
| `profile <name>` | Записать `baseProfile` и пере-стейджить скиллы |
| `disable <cluster>` | Добавить кластер в disabled-список и пере-стейджить |
| `enable <cluster>` | Убрать кластер из disabled-списка и пере-стейджить |
| `reset` | Удалить surface-дельту; вернуться к install-time профилю |

```bash
/gsd-surface list                   # Показать текущую поверхность
/gsd-surface profile standard       # Переключиться на standard-профиль
/gsd-surface disable utility        # Отключить кластер utility
/gsd-surface reset                  # Восстановить install-time профиль
```

---

## Brownfield-команды

### `/gsd-map-codebase`

Проанализировать существующую кодовую базу параллельными mapper-агентами. Используй `--fast` для быстрого одно-агентного скана или `--query` для поиска по существующему intel'у.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `area` | Нет | Скоупировать маппинг под конкретную область |
| `--fast` | Нет | Быстрая single-focus оценка — спавнит одного mapper-агента вместо четырёх (лёгкая альтернатива) |
| `--query <term>` | Нет | Поиск queryable intel-файлов кодовой базы в `.planning/intel/` (требует `intel.enabled: true`) |

| Флаг | Описание |
|------|----------|
| `--focus tech\|arch\|quality\|concerns\|tech+arch` | Фокусная область для `--fast` (дефолт: `tech+arch`) |

**Производит:** Документы анализа в `.planning/codebase/` (полный режим); таргетные документ(ы) в `.planning/codebase/` (`--fast`); результаты intel-query (`--query`)

```bash
/gsd-map-codebase                   # Полный анализ кодовой базы (4 параллельных агента)
/gsd-map-codebase auth              # Фокус на auth-области
/gsd-map-codebase --fast            # Быстрый обзор tech + arch (1 агент)
/gsd-map-codebase --fast --focus quality  # Только качество и здоровье кода
/gsd-map-codebase --query authentication  # Поиск по intel'у по термину
```

### `/gsd-graphify`

Строить, query'ить и инспектить граф знаний проекта, хранимый в `.planning/graphs/`. Opt-in через `graphify.enabled: true` в `config.json` (см. [Configuration Reference](CONFIGURATION.md#graphify-settings)); когда отключено, команда печатает подсказку по активации и останавливается.

| Подкоманда | Описание |
|------------|----------|
| `build` | Построить или перестроить граф знаний (запускает `graphify update .` инлайн и обновляет `.planning/graphs/`) |
| `query <term>` | Поиск по графу по термину |
| `status` | Свежесть и статистика графа |
| `diff` | Изменения с последней сборки |

**Производит:** Графовые артефакты в `.planning/graphs/` (узлы, рёбра, снапшоты)

```bash
/gsd-graphify build                 # Построить или перестроить граф знаний
/gsd-graphify query authentication  # Поиск по графу
/gsd-graphify status                # Свежесть и статистика
/gsd-graphify diff                  # Изменения с последней сборки
```

**Программный доступ:** `node gsd-tools.cjs graphify <build|query|status|diff|snapshot>` — см. [CLI Tools Reference](CLI-TOOLS.md).

---

## Команды AI-интеграции

### `/gsd-ai-integration-phase`

Сгенерить AI-SPEC.md дизайн-контракт для фаз, включающих построение AI-систем. Презентует интерактивную decision-matrix, всплывает domain-specific failure modes и eval-критерии, производит `AI-SPEC.md` с рекомендацией фреймворка, гайдом по реализации и стратегией оценки.

**Производит:** `{phase}-AI-SPEC.md` в директории фазы

**Спавнит:** 3 параллельных специалист-агента: domain-researcher, framework-selector, ai-researcher, eval-planner

```bash
/gsd-ai-integration-phase              # Wizard для текущей фазы
/gsd-ai-integration-phase 3            # Wizard для конкретной фазы
```

---

### `/gsd-eval-review`

Проаудить покрытие оценки исполненной AI-фазы и произвести EVAL-REVIEW.md remediation-план. Проверяет реализацию против evaluation-плана в `AI-SPEC.md`, произведённого `/gsd-ai-integration-phase`. Оценивает каждое eval-измерение как COVERED/PARTIAL/MISSING.

**Предусловия:** Фаза исполнена и имеет `AI-SPEC.md`
**Производит:** `{phase}-EVAL-REVIEW.md` с находками, пробелами и remediation-гайдом

```bash
/gsd-eval-review                       # Аудит текущей фазы
/gsd-eval-review 3                     # Аудит конкретной фазы
```

---

## Команды обновления

### `/gsd-update`

Обновить GSD с превью changelog'а и опционально синком скиллов или повторного применения локальных патчей.

| Флаг | Описание |
|------|----------|
| `--sync` | Синкнуть скиллы из GSD-реестра после обновления |
| `--reapply` | Восстановить локальные модификации (patches) после обновления |

```bash
/gsd-update                         # Проверить обновления и установить
/gsd-update --sync                  # Обновить и синкнуть скиллы
/gsd-update --reapply               # Обновить и переприменить локальные патчи
```

---

## Команды качества кода

### `/gsd-code-review`

Ревью source-файлов, изменённых во время фазы, на баги, уязвимости безопасности и проблемы качества кода. Используй `--fix` чтобы автофиксить находки после ревью.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `N` | **Да** | Номер фазы, изменения которой ревьюим (например, `2` или `02`) |
| `--depth=quick\|standard\|deep` | Нет | Уровень глубины ревью (override `workflow.code_review_depth` в конфиге). `quick`: только pattern-matching (~2 мин). `standard`: per-file анализ с language-specific проверками (~5–15 мин, дефолт). `deep`: cross-file анализ, графы импортов, цепи вызовов (~15–30 мин) |
| `--files file1,file2,...` | Нет | Явный список файлов через запятую; пропускает SUMMARY/git-скоупинг |
| `--fix` | Нет | Автофикс проблем после ревью — читает REVIEW.md, спавнит fixer-агента, коммитит каждый фикс атомарно |
| `--fix --all` | Нет | Включить Info-находки в scope фиксов (дефолт: только Critical + Warning) |
| `--fix --auto` | Нет | Цикл fix + re-review, кап 3 итерации |

**Предусловия:** Фаза исполнена и имеет SUMMARY.md или git-историю
**Производит:** `{phase}-REVIEW.md` с severity-классифицированными находками; `{phase}-REVIEW-FIX.md` когда используется `--fix`
**Спавнит:** агент `gsd-code-reviewer`; агент `gsd-code-fixer` (с `--fix`)

```bash
/gsd-code-review 3                          # Стандартное ревью фазы 3
/gsd-code-review 2 --depth=deep             # Глубокое cross-file ревью
/gsd-code-review 4 --files src/auth.ts,src/token.ts  # Явный список файлов
/gsd-code-review 3 --fix                    # Ревью потом фикс Critical + Warning
/gsd-code-review 3 --fix --all             # Ревью потом фикс всех включая Info
/gsd-code-review 3 --fix --auto            # Ревью, фикс и re-review пока чисто (макс 3 итерации)
```

---

### `/gsd-audit-fix`

Автономный audit-to-fix пайплайн — запускает аудит, классифицирует находки, чинит autofix-проблемы с верификацией тестами и коммитит каждый фикс атомарно.

| Флаг | Описание |
|------|----------|
| `--source <audit>` | Какой аудит запускать (дефолт: `audit-uat`) |
| `--severity high\|medium\|all` | Минимальная серьёзность для обработки (дефолт: `medium`) |
| `--max N` | Максимум находок для фикса (дефолт: 5) |
| `--dry-run` | Классифицировать находки без фикса (показывает таблицу классификации) |

**Предусловия:** Хотя бы одна фаза исполнена с UAT или верификацией
**Производит:** Fix-коммиты с тест-верификацией; classification-отчёт

```bash
/gsd-audit-fix                              # Запустить audit-uat, фиксить medium+ (макс 5)
/gsd-audit-fix --severity high             # Только high-severity
/gsd-audit-fix --dry-run                   # Превью классификации без фикса
/gsd-audit-fix --max 10 --severity all     # Фиксить до 10 проблем любой severity
```

---

## Быстрые и инлайн-команды

### `/gsd-fast`

Исполнить тривиальную задачу инлайн — без sub-агентов, без накладных планирования. Для фиксов опечаток, изменений конфига, мелких рефакторингов, забытых коммитов.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `task description` | Нет | Что делать (промпт если пропущено) |

**Не замена `/gsd-quick`** — используй `/gsd-quick` для всего, что требует research'а, многошагового планирования или верификации.

```bash
/gsd-fast "fix typo in README"
/gsd-fast "add .env to gitignore"
```

---

### `/gsd-review`

Cross-AI peer review планов фазы от внешних AI CLI.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `--phase N` | **Да** | Номер фазы для ревью |

| Флаг | Описание |
|------|----------|
| `--gemini` | Включить Gemini CLI ревью |
| `--claude` | Включить Claude CLI ревью (отдельная сессия) |
| `--codex` | Включить Codex CLI ревью |
| `--coderabbit` | Включить CodeRabbit ревью |
| `--opencode` | Включить OpenCode ревью (через GitHub Copilot) |
| `--qwen` | Включить Qwen Code ревью (Alibaba Qwen-модели) |
| `--cursor` | Включить Cursor-агент ревью |
| `--ollama` | Включить Ollama-сервер ревью |
| `--lm-studio` | Включить LM Studio ревью |
| `--llama-cpp` | Включить llama.cpp ревью |
| `--all` | Включить всех доступных ревьюеров (CLI + локальные model-серверы) |

**Дефолтное поведение ревьюеров (без флагов):**
- Если `review.default_reviewers` **не установлен**, `/gsd-review` запускает всех детектированных ревьюеров (текущее дефолтное поведение).
- Если `review.default_reviewers` **установлен**, `/gsd-review` запускает только этот subset (например `["gemini","codex"]`).
- `--all` всегда переопределяет конфиг и запускает полный детектированный набор.
- Явные флаги (например `--cursor`) переопределяют и `--all`, и дефолты конфига для этого запуска.

**Производит:** `{phase}-REVIEWS.md` — потребляемый `/gsd-plan-phase --reviews`

```bash
# Установить проектные дефолтные ревьюеры для no-flag запусков /gsd-review
gsd config-set review.default_reviewers '["gemini","codex"]'

/gsd-review --phase 2             # запускает gemini+codex из конфига
/gsd-review --phase 3 --all
/gsd-review --phase 2 --gemini
/gsd-review --phase 2 --cursor    # разовый override
```

---

### `/gsd-pr-branch`

Создать чистую PR-ветку, отфильтровав коммиты `.planning/`.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `target branch` | Нет | Базовая ветка (дефолт: `main`) |

**Назначение:** Ревьюеры видят только изменения кода, не планировочные артефакты GSD.

```bash
/gsd-pr-branch                     # Фильтровать против main
/gsd-pr-branch develop             # Фильтровать против develop
```

---

### `/gsd-secure-phase`

Ретроактивно верифицировать threat-mitigations для завершённой фазы.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `phase number` | Нет | Фаза для аудита (дефолт: последняя завершённая) |

**Предусловия:** Фаза должна быть исполнена. Работает и с, и без существующего SECURITY.md.
**Производит:** `{phase}-SECURITY.md` с результатами верификации угроз
**Спавнит:** агент `gsd-security-auditor`

Три режима работы:
1. SECURITY.md существует — аудит и верификация существующих mitigations
2. SECURITY.md нет, но в PLAN.md есть threat model — сгенерировать из артефактов
3. Фаза не исполнена — выйти с подсказкой

```bash
/gsd-secure-phase                   # Аудит последней завершённой фазы
/gsd-secure-phase 5                 # Аудит конкретной фазы
```

---

### `/gsd-docs-update`

Сгенерить или обновить документацию проекта, верифицированную против кодовой базы.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| `--force` | Нет | Пропустить preservation-промпты, перегенерить все доки |
| `--verify-only` | Нет | Проверить существующие доки на точность, без генерации |

**Производит:** До 9 doc-файлов (README, architecture, API, getting started, development, testing, configuration, deployment, contributing)
**Спавнит:** агенты `gsd-doc-writer` (по одному на тип дока), потом агенты `gsd-doc-verifier` для фактической верификации

Каждый doc-writer исследует кодовую базу напрямую — никаких галлюцинированных путей или устаревших сигнатур. Doc-verifier проверяет утверждения против живой ФС.

```bash
/gsd-docs-update                    # Сгенерить/обновить доки интерактивно
/gsd-docs-update --force            # Перегенерить все доки
/gsd-docs-update --verify-only      # Только верифицировать существующие
```

---

## Команды захвата задач и бэклога

### `/gsd-capture`

Захватить идеи, задачи, заметки и seeds в подходящее место. Дефолтный режим добавляет структурированный todo; флаги роутят в специализированные capture-воркфлоу.

| Флаг | Описание |
|------|----------|
| (нет) | Захват как структурированный todo для последующей работы |
| `--note [text]` | Zero-friction заметка — append, list (`--note list`) или promote (`--note promote N`) |
| `--backlog <description>` | Добавить в бэклог-парковку с нумерацией 999.x |
| `--seed [idea summary]` | Захватить forward-looking идею с триггер-условиями |
| `--list` | Список висящих todo и выбор одного для работы |
| `--global` | Использовать глобальный скоуп (для операций с заметками) |

**Бэклог:** 999.x нумерация держит айтемы вне активной последовательности фаз; phase-директории создаются сразу, чтобы `/gsd-discuss-phase` и `/gsd-plan-phase` работали на них.
**Seeds:** Сохраняют полное WHY, WHEN-всплыть и breadcrumbs — потребляются `/gsd-new-milestone`.

**Производит:** `.planning/todos/` (дефолт), файлы заметок (--note), backlog-секция ROADMAP.md (--backlog), `.planning/seeds/SEED-NNN-slug.md` (--seed)

```bash
/gsd-capture "Consider adding dark mode support"   # Добавить todo
/gsd-capture --note "Caching strategy idea"        # Быстрая заметка
/gsd-capture --note list                           # Список заметок
/gsd-capture --note promote 3                      # Продвинуть заметку 3 в todo
/gsd-capture --backlog "GraphQL API layer"         # Добавить в бэклог
/gsd-capture --seed "Add real-time collaboration when WebSocket infra is in place"
/gsd-capture --list                                # Просмотреть и действовать с todo
```

---

### `/gsd-review-backlog`

Просмотреть и продвинуть бэклог-айтемы в активный майлстоун.

**Действия per айтем:** Promote (перенести в активную последовательность), Keep (оставить в бэклоге), Remove (удалить).

```bash
/gsd-review-backlog
```

---

### `/gsd-thread`

Управление постоянными контекстными тредами для cross-session работы.

| Аргумент | Обязательный | Описание |
|----------|--------------|----------|
| (нет) / `list` | — | Список всех тредов |
| `list --open` | — | Только треды со статусом `open` или `in_progress` |
| `list --resolved` | — | Только треды со статусом `resolved` |
| `status <slug>` | — | Статус конкретного треда |
| `close <slug>` | — | Пометить тред как resolved |
| `name` | — | Возобновить существующий тред по имени |
| `description` | — | Создать новый тред |

Треды — лёгкие cross-session хранилища знаний для работы, которая растягивается на несколько сессий, но не принадлежит конкретной фазе. Легче чем `/gsd-pause-work`.

```bash
/gsd-thread                         # Список всех тредов
/gsd-thread list --open             # Только open/in-progress
/gsd-thread list --resolved         # Только resolved
/gsd-thread status fix-deploy-key   # Статус треда
/gsd-thread close fix-deploy-key    # Пометить как resolved
/gsd-thread fix-deploy-key-auth     # Возобновить тред
/gsd-thread "Investigate TCP timeout in pasta service"  # Создать новый
```

---

## Команды управления состоянием

### `state validate`

Обнаружить дрейф между STATE.md и фактической ФС.

**Предусловия:** `.planning/STATE.md` существует
**Производит:** Отчёт о валидации с дрейфом между полями STATE.md и реальностью ФС

```bash
node gsd-tools.cjs state validate
```

---

### `state sync [--verify]`

Реконструировать STATE.md из фактического состояния проекта на диске.

| Флаг | Описание |
|------|----------|
| `--verify` | Dry-run режим — показать предложенные изменения без записи |

**Предусловия:** Директория `.planning/` существует
**Производит:** Обновлённый `STATE.md`, отражающий реальность ФС

```bash
node gsd-tools.cjs state sync             # Реконструировать STATE.md с диска
node gsd-tools.cjs state sync --verify    # Dry-run: показать изменения без записи
```

---

### `state planned-phase`

Зафиксировать переход состояния после завершения plan-фазы (Planned/Ready to execute).

| Флаг | Описание |
|------|----------|
| `--phase N` | Номер запланированной фазы |
| `--plans N` | Число сгенерированных планов |

**Предусловия:** Фаза спланирована
**Производит:** Обновлённый `STATE.md` с post-planning состоянием

```bash
node gsd-tools.cjs state planned-phase --phase 3 --plans 2
```

---

## Community-команды

### Community-хуки

Опциональные git- и session-хуки, гейтнутые за `hooks.community: true` в `.planning/config.json`. Все no-op'ы пока явно не включены.

| Хук | Назначение |
|-----|------------|
| `gsd-validate-commit.sh` | Форсить формат Conventional Commits на git-коммит-сообщениях |
| `gsd-session-state.sh` | Отслеживать переходы состояния сессии |
| `gsd-phase-boundary.sh` | Форсить проверки границ фаз |

Включение:
```json
{ "hooks": { "community": true } }
```

---

### Community-приглашение

Чтобы присоединиться к Discord-сообществу GSD, посети ссылку в README GSD или запусти `/gsd-help` и пройди по Discord-ссылке.

---

## Контрибьюторам: стандарты описаний скиллов

Описания скиллов (поле `description:` во frontmatter каждого `commands/gsd/*.md`) инжектятся в system prompt каждой сессии. Чтобы держать per-session overhead низким, описания должны быть ≤ 100 символов и не должны дублировать документацию флагов, уже в `argument-hint:`.

Lint-gate форсит бюджет:

```bash
npm run lint:descriptions
```

Проверка также запускается как часть `npm test` через `tests/enh-2789-description-budget.test.cjs`.
