# GSD — Справочник конфигурации

> Полная схема конфигурации, переключатели workflow, профили моделей и опции git-ветвления. Контекст фич — в [Feature Reference](FEATURES.md).

---

## Файл конфигурации

GSD хранит project-настройки в `.planning/config.json`. Создаётся во время `/gsd-new-project`, обновляется через `/gsd-settings`.

### Полная схема

```json
{
  "mode": "interactive",
  "granularity": "standard",
  "model_profile": "balanced",
  "model_overrides": {},
  "models": {},
  "dynamic_routing": null,
  "planning": {
    "commit_docs": true,
    "search_gitignored": false,
    "sub_repos": []
  },
  "context": null,
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true,
    "auto_advance": false,
    "nyquist_validation": true,
    "ui_phase": true,
    "ui_safety_gate": true,
    "ui_review": true,
    "node_repair": true,
    "node_repair_budget": 2,
    "research_before_questions": false,
    "discuss_mode": "discuss",
    "max_discuss_passes": 3,
    "skip_discuss": false,
    "tdd_mode": false,
    "text_mode": false,
    "use_worktrees": true,
    "code_review": true,
    "code_review_depth": "standard",
    "plan_bounce": false,
    "plan_bounce_script": null,
    "plan_bounce_passes": 2,
    "plan_chunked": false,
    "code_review_command": null,
    "cross_ai_execution": false,
    "cross_ai_command": null,
    "cross_ai_timeout": 300,
    "security_enforcement": true,
    "security_asvs_level": 1,
    "security_block_on": "high",
    "post_planning_gaps": true,
    "build_command": null,
    "test_command": null
  },
  "code_quality": {
    "fallow": {
      "enabled": false,
      "scope": "phase",
      "profile": "standard",
      "mcp": false
    }
  },
  "ship": {
    "pr_body_sections": []
  },
  "hooks": {
    "context_warnings": true,
    "workflow_guard": false
  },
  "review": {
    "default_reviewers": null,
    "models": {}
  },
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "skip_checkpoints": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  },
  "git": {
    "branching_strategy": "none",
    "phase_branch_template": "gsd/phase-{phase}-{slug}",
    "milestone_branch_template": "gsd/{milestone}-{slug}",
    "quick_branch_template": null
  },
  "gates": {
    "confirm_project": true,
    "confirm_phases": true,
    "confirm_roadmap": true,
    "confirm_breakdown": true,
    "confirm_plan": true,
    "execute_next_plan": true,
    "issues_review": true,
    "confirm_transition": true
  },
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  },
  "project_code": null,
  "agent_skills": {},
  "response_language": null,
  "features": {
    "thinking_partner": false,
    "global_learnings": false
  },
  "learnings": {
    "max_inject": 10
  },
  "intel": {
    "enabled": false
  },
  "claude_md_path": "./CLAUDE.md"
}
```

---

## Core-настройки

| Настройка | Тип | Опции | Дефолт | Описание |
|-----------|-----|-------|--------|----------|
| `mode` | enum | `interactive`, `yolo` | `interactive` | `yolo` авто-одобряет решения; `interactive` подтверждает на каждом шаге |
| `granularity` | enum | `coarse`, `standard`, `fine` | `standard` | Контролирует число фаз: `coarse` (3-5), `standard` (5-8), `fine` (8-12) |
| `model_profile` | enum | `quality`, `balanced`, `budget`, `adaptive`, `inherit` | `balanced` | Tier модели на каждого агента (см. [Профили моделей](#профили-моделей)). `adaptive` добавлен по [#1713](https://github.com/gsd-build/get-shit-done/issues/1713) / [#1806](https://github.com/gsd-build/get-shit-done/issues/1806) и резолвится так же как другие tier'ы под runtime-aware профилями. |
| `runtime` | string | `claude`, `codex` или любая строка | (нет) | Активный рантайм для [runtime-aware резолва профиля](#runtime-aware-профили-2517). Когда установлен, tier'ы профиля (opus/sonnet/haiku) резолвятся в runtime-нативные model ID. Сегодня только Codex-путь установки эмитит per-agent model ID из этого резолвера; другие рантаймы (`opencode`, `gemini`, `qwen`, `copilot`, …) потребляют резолвер на spawn-time и получают выделенную поддержку install-path в [#2612](https://github.com/gsd-build/get-shit-done/issues/2612). Когда не установлен (дефолт), поведение не меняется с предыдущих версий. Добавлено в v1.39 |
| `model_profile_overrides.<runtime>.<tier>` | string \| object | per-runtime tier override | (нет) | Переопределить runtime-aware tier-маппинг для конкретного `(runtime, tier)`. Tier — один из `opus`, `sonnet`, `haiku`. Значение — либо строка model ID (например, `"gpt-5-pro"`), либо `{ model, reasoning_effort }`. См. [Runtime-Aware Profiles](#runtime-aware-профили-2517). Добавлено в v1.39 |
| `models.<phase_type>` | enum | `opus`, `sonnet`, `haiku`, `inherit` | (нет) | Tier модели на per-phase-type. Шесть принимаемых слотов: `planning`, `discuss`, `research`, `execution`, `verification`, `completion`. Позволяет тюнить на уровне фазы («Opus для планирования, Sonnet для остального») без изучения имён агентов. Резолвится между `model_overrides` (выше) и `model_profile` (ниже); см. [Per-Phase-Type Models](#per-phase-type-models-models). Добавлено в v1.40 |
| `dynamic_routing.enabled` | boolean | `true`, `false` | `false` | Главный переключатель для [dynamic routing с failure-tier эскалацией](#dynamic-routing-с-failure-tier-эскалацией). Когда `true`, агенты резолвятся в `tier_models[default_tier]` и эскалируются на один tier вверх на soft failure детектированном оркестратором. Добавлено в v1.40 |
| `dynamic_routing.tier_models.<tier>` | enum | `opus`, `sonnet`, `haiku` | (нет) | Tier-alias для `light`, `standard` или `heavy`. Используется когда `dynamic_routing.enabled: true`. Добавлено в v1.40 |
| `dynamic_routing.escalate_on_failure` | boolean | `true`, `false` | `true` | Когда `false`, эскалация отключена даже если `enabled: true` — каждая попытка использует default tier. Добавлено в v1.40 |
| `dynamic_routing.max_escalations` | integer | `0`, `1`, `2`, … | `1` | Жёсткий cap на retry на вызов агента. За пределами cap'а резолвер возвращает cap-tier модель. Добавлено в v1.40 |
| `project_code` | string | любая короткая строка | (нет) | Префикс для имён phase-директорий (например, `"ABC"` производит `ABC-01-setup/`). Добавлено в v1.31 |
| `response_language` | string | код языка | (нет) | Язык для ответов агентов (например, `"pt"`, `"ko"`, `"ja"`). Распространяется на всех заспавненных агентов для cross-phase языковой консистентности. Добавлено в v1.32 |
| `context_window` | number | любое целое | `200000` | Размер контекстного окна в токенах. Поставь `1000000` для 1M-context моделей (например, `claude-opus-4-7[1m]`). Значения `>= 500000` включают адаптивное обогащение контекста (full-body чтение предыдущих SUMMARY.md, глубокие anti-pattern чтения). Конфигурируется через `/gsd-config --advanced`. |
| `context_profile` | string | `dev`, `research`, `review` | (нет) | Execution context пресет, применяющий пред-сконфигурированный bundle настроек mode, model и workflow под текущий тип работы. Добавлено в v1.34 |
| `claude_md_path` | string | любой путь | `./CLAUDE.md` | Кастомный output-путь для сгенерированного CLAUDE.md. Полезно для монорепо или проектов, которым нужен CLAUDE.md в не-корневой локации. Дефолт — `./CLAUDE.md` в корне проекта. Добавлено в v1.36 |
| `claude_md_assembly.mode` | enum | `embed`, `link` | `embed` | Контролирует, как managed-секции пишутся в CLAUDE.md. `embed` (дефолт) инлайнит контент между GSD-маркерами. `link` пишет `@.planning/<source-path>` вместо этого — Claude Code раскрывает референс в рантайме, уменьшая размер CLAUDE.md на ~65% на типичных проектах. `link` применяется только к секциям с реальным source-файлом; `workflow` и fallback-секции всегда инлайнятся. Per-block override'ы: `claude_md_assembly.blocks.<section>` (например, `claude_md_assembly.blocks.architecture: link`). Добавлено в v1.38 |
| `context` | string | любой текст | (нет) | Кастомная строка контекста, инжектируемая в каждый промпт агента проекта. Используй чтобы давать persistent project-specific гайды (например, code-конвенции, командные практики), о которых должен знать каждый агент |
| `phase_naming` | string | любая строка | (нет) | Кастомный префикс для имён phase-директорий. Когда установлен, override авто-сгенерированный phase slug (например, `"feature"` производит `feature-01-setup/` вместо roadmap-derived slug'а) |
| `brave_search` | boolean | `true`/`false` | auto-detected | Override авто-детекта доступности Brave Search API. Когда не установлен, GSD проверяет env-переменную `BRAVE_API_KEY` или файл `~/.gsd/brave_api_key` |
| `firecrawl` | boolean | `true`/`false` | auto-detected | Override авто-детекта доступности Firecrawl API. Когда не установлен, GSD проверяет env-переменную `FIRECRAWL_API_KEY` или файл `~/.gsd/firecrawl_api_key` |
| `exa_search` | boolean | `true`/`false` | auto-detected | Override авто-детекта доступности Exa Search API. Когда не установлен, GSD проверяет env-переменную `EXA_API_KEY` или файл `~/.gsd/exa_api_key` |
| `search_gitignored` | boolean | `true`/`false` | `false` | Легаси top-level алиас для `planning.search_gitignored`. Предпочитай namespaced-форму; алиас принимается для backward compatibility |

> **Заметка:** `granularity` была переименована из `depth` в v1.22.3. Существующие конфиги авто-мигрируются.

---

## Интеграционные настройки

Конфигурируются интерактивно через [`/gsd-config --integrations`](COMMANDS.ru-RU.md). Это настройки *коннективности* — API-ключи и cross-tool routing — намеренно отделены от `/gsd-settings` (переключатели workflow).

### API-ключи поиска

Поля API-ключей принимают строковое значение (сам ключ). Также можно установить sentinel'ы `true`/`false`/`null` чтобы переопределить авто-детект из env-переменных / `~/.gsd/*_api_key` файлов (легаси, см. строки выше).

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `brave_search` | string \| boolean \| null | `null` | Brave Search API-ключ для web research. Отображается как `****<last-4>` во всём UI / выводе `config-set`; никогда не выводится в plaintext |
| `firecrawl` | string \| boolean \| null | `null` | Firecrawl API-ключ для deep-crawl скрейпинга. Маскирован в отображении |
| `exa_search` | string \| boolean \| null | `null` | Exa Search API-ключ для семантического поиска. Маскирован в отображении |

**Конвенция маскирования (`get-shit-done/bin/lib/secrets.cjs`):** ключи 8+ символов рендерятся как `****<last-4>`; короче — `****`; `null`/empty — `(unset)`. Plaintext пишется как есть в `.planning/config.json` — этот файл является границей безопасности — но CLI, confirmation-таблицы, логи и описания `AskUserQuestion` никогда не отображают plaintext. Это применяется к самому выводу `config-set`: `config-set brave_search <key>` возвращает JSON-payload со значением маскированным.

### Code-review CLI routing

`review.models.<cli>` маппит флейвор ревьюера на shell-команду. Code-review workflow шеллится этой командой когда запрошен соответствующий флейвор.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `review.models.claude` | string | (session model) | Команда для Claude-flavored ревью. Дефолт — session model когда не установлен |
| `review.models.codex` | string | `null` | Команда для Codex-ревью, например `"codex exec --model gpt-5"` |
| `review.models.gemini` | string | `null` | Команда для Gemini-ревью, например `"gemini -m gemini-2.5-pro"` |
| `review.models.opencode` | string | `null` | Команда для OpenCode-ревью, например `"opencode run --model claude-sonnet-4"` |

Slug `<cli>` валидируется против `[a-zA-Z0-9_-]+`. Пустые или содержащие путь slug'и отвергаются `config-set`.

### Дефолтные ревьюеры для `/gsd-review`

Используй `review.default_reviewers` чтобы скоупить no-flag запуск `/gsd-review` на subset детектированных ревьюеров.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `review.default_reviewers` | string[] \| null | `null` (все детектированные ревьюеры) | Опциональный дефолтный subset для no-flag `/gsd-review`, например `["gemini","codex"]`. Приоритет: явные ревьюер-флаги > `--all` > `review.default_reviewers` > все детектированные. Неизвестные slug'и игнорируются с warning'ом; известные-но-недетектированные slug'и игнорируются с info-нотой; пустые массивы отвергаются `config-set`. |

Пример:

```json
{
  "review": {
    "default_reviewers": ["gemini", "codex"]
  }
}
```

### Инъекция скиллов агента (dynamic)

`agent_skills.<agent-type>` расширяет map `agent_skills`, задокументированный ниже. Slug валидируется против `[a-zA-Z0-9_-]+` — без path-сепараторов, whitespace, shell-метасимволов. Конфигурируется интерактивно через `/gsd-config --integrations`.

---

## Переключатели Workflow

Все переключатели workflow следуют паттерну **absent = enabled**. Если ключ отсутствует в конфиге, дефолт — `true`.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `workflow.research` | boolean | `true` | Domain-исследование до планирования каждой фазы |
| `workflow.plan_check` | boolean | `true` | Verify-цикл плана (до 3 итераций) |
| `workflow.verifier` | boolean | `true` | Постфактум-верификация против целей фазы |
| `workflow.auto_advance` | boolean | `false` | Авто-цепочка discuss → plan → execute без остановок |
| `workflow.nyquist_validation` | boolean | `true` | Маппинг тест-покрытия во время research'а plan-фазы |
| `workflow.ui_phase` | boolean | `true` | Генерировать UI-дизайн-контракты для frontend-фаз |
| `workflow.ui_safety_gate` | boolean | `true` | Промпт на запуск /gsd-ui-phase для frontend-фаз во время plan-phase |
| `workflow.ui_review` | boolean | `true` | Запускать визуальный quality-аудит (`/gsd-ui-review`) после исполнения фазы в автономном режиме. Когда `false`, UI-аудит пропускается. |
| `workflow.node_repair` | boolean | `true` | Автономный task-repair на verification failure |
| `workflow.node_repair_budget` | number | `2` | Макс попыток repair на упавшую задачу |
| `workflow.research_before_questions` | boolean | `false` | Запускать research до вопросов обсуждения, а не после |
| `workflow.discuss_mode` | string | `'discuss'` | Контролирует как `/gsd-discuss-phase` собирает контекст. `'discuss'` (дефолт) задаёт вопросы по одному. `'assumptions'` сначала читает кодовую базу, генерит структурированные предположения с уровнями уверенности и просит только корректировки. Добавлено в v1.28 |
| `workflow.max_discuss_passes` | number | `3` | Максимум раундов вопросов в discuss-phase до того как workflow перестанет спрашивать. Полезно в headless/auto режиме чтобы предотвратить бесконечные discussion-циклы. |
| `workflow.skip_discuss` | boolean | `false` | Когда `true`, `/gsd-autonomous` полностью обходит discuss-phase, пишет минимальный CONTEXT.md из ROADMAP phase goal. Полезно для проектов, где developer-предпочтения полностью зафиксированы в PROJECT.md/REQUIREMENTS.md. Добавлено в v1.28 |
| `workflow.text_mode` | boolean | `false` | Заменяет AskUserQuestion TUI-меню plain-text нумерованными списками. Требуется для remote-сессий Claude Code (`/rc` режим), где TUI-меню не рендерятся. Также можно установить per-session флагом `--text` на discuss-phase. Добавлено в v1.28 |
| `workflow.use_worktrees` | boolean | `true` | Когда `false`, отключает изоляцию через git worktree для параллельного исполнения. Пользователи, предпочитающие sequential-исполнение или среды без поддержки worktree, могут отключить. Добавлено в v1.31 |
| `workflow.worktree_skip_hooks` | boolean | `false` | Когда `true`, executor-агенты в worktree-режиме передают `--no-verify` (пропуская pre-commit хуки) и post-wave валидация хуков прогоняется против объединённого результата. Opt-in escape hatch для проектов, чьи хуки не могут работать в agent worktree'ах. Дефолт `false` запускает хуки на каждом коммите (#2924). |
| `workflow.code_review` | boolean | `true` | Включить команды `/gsd-code-review` и `/gsd-code-review --fix`. Когда `false`, команды выходят с configuration-gate сообщением. Добавлено в v1.34 |
| `workflow.code_review_depth` | string | `standard` | Дефолтная глубина ревью для `/gsd-code-review`: `quick` (только pattern-matching), `standard` (per-file анализ) или `deep` (cross-file с графами импортов). Можно переопределить per-run флагом `--depth=`. Добавлено в v1.34 |
| `workflow.plan_bounce` | boolean | `false` | Запускать внешний validation-скрипт против сгенерированных планов. Когда включено, plan-phase оркестратор пайпит каждый PLAN.md через скрипт, указанный в `plan_bounce_script`, и блокирует на non-zero exit. Добавлено в v1.36 |
| `workflow.plan_bounce_script` | string | (нет) | Путь к внешнему скрипту, вызываемому для plan bounce валидации. Получает путь к PLAN.md как первый аргумент. Обязателен когда `plan_bounce` равен `true`. Добавлено в v1.36 |
| `workflow.plan_bounce_passes` | number | `2` | Число последовательных bounce-проходов. Каждый проход кормит вывод предыдущего в валидатор. Высокие значения повышают строгость ценой latency. Добавлено в v1.36 |
| `workflow.post_planning_gaps` | boolean | `true` | Унифицированный post-planning gap-отчёт (#2493). После генерации и коммита всех планов сканирует REQUIREMENTS.md и `<decisions>` из CONTEXT.md против каждого PLAN.md в phase-директории, потом печатает одну таблицу `Source \| Item \| Status`. Word-boundary match (REQ-1 vs REQ-10) и natural sort (REQ-02 до REQ-10). Не блокирует — только информационный отчёт. Поставь в `false` чтобы пропустить Step 13e plan-phase. |
| `workflow.plan_review_convergence` | boolean | `false` | Включить команду `/gsd-plan-review-convergence`. Отключено по умолчанию — команда выходит с инструкцией включения когда этот ключ `false`. Команда автоматизирует ручной цикл plan→review→replan: спавнит сконфигурированных ревьюеров (Codex, Gemini, Claude, OpenCode, Ollama, LM Studio, llama.cpp), считает неразрешённые HIGH-замечания через CYCLE_SUMMARY-контракт, перепланирует с `--reviews` фидбеком и повторяет до конвергенции или достижения max cycles. Включи через `gsd config-set workflow.plan_review_convergence true`. Добавлено в v1.39 |
| `workflow.plan_chunked` | boolean | `false` | Включить chunked-режим планирования. Когда `true` (или когда передан флаг `--chunked` в `/gsd-plan-phase`), оркестратор разбивает один долгоживущий planner-Task на короткий outline-Task плюс N коротких per-plan Task'ов (~3-5 мин каждый). Каждый план коммитится индивидуально для crash-resilience. Если Task зависнет и терминал force-killed, перезапуск с `--chunked` возобновляет с последнего завершённого плана. Особенно полезно на Windows, где долгоживущие Task'и могут зависнуть на stdio. Добавлено в v1.38 |
| `workflow.code_review_command` | string | (нет) | Shell-команда для внешней code-review интеграции в `/gsd-ship`. Получает пути изменённых файлов через stdin. Non-zero exit блокирует ship-workflow. Добавлено в v1.36 |
| `workflow.tdd_mode` | boolean | `false` | Включить TDD-пайплайн как first-class execution-режим. Когда `true`, планировщик агрессивно применяет `type: tdd` к подходящим задачам (бизнес-логика, API, валидации, алгоритмы) и executor форсит последовательность gate'ов RED/GREEN/REFACTOR. End-of-phase collaborative review checkpoint верифицирует соблюдение gate'ов. Добавлено в v1.36 |
| `workflow.human_verify_mode` | string | `'end-of-phase'` | Контролирует human verification чекпоинты. `'end-of-phase'` (дефолт с #3309) подавляет `checkpoint:human-verify` задачи и встраивает проверки в `<verify><human-check>` блоки для end-of-phase ревью. `'mid-flight'` восстанавливает блокирующие checkpoint-задачи. `checkpoint:decision` и `checkpoint:human-action` не затронуты. См. [Checkpoints Reference](../get-shit-done/references/checkpoints.md#checkpoint_types). |
| `workflow.cross_ai_execution` | boolean | `false` | Делегировать исполнение фазы внешнему AI CLI вместо спавна локальных executor-агентов. Полезно для использования сильных сторон другой модели для конкретных фаз. Добавлено в v1.36 |
| `workflow.cross_ai_command` | string | (нет) | Shell-команда-шаблон для cross-AI исполнения. Получает phase-промпт через stdin. Должна производить SUMMARY.md-совместимый output. Обязателен когда `cross_ai_execution` равен `true`. Добавлено в v1.36 |
| `workflow.cross_ai_timeout` | number | `300` | Таймаут в секундах для cross-AI execution команд. Предотвращает runaway-внешние процессы. Добавлено в v1.36 |
| `workflow.ai_integration_phase` | boolean | `true` | Включить команду `/gsd-ai-integration-phase`. Когда `false`, команда выходит с configuration-gate сообщением |
| `workflow.auto_prune_state` | boolean | `false` | Когда `true`, автоматически чистить stale-записи из STATE.md на границах фаз вместо промпта |
| `workflow.pattern_mapper` | boolean | `true` | Запускать агент `gsd-pattern-mapper` между research'ем и планированием чтобы маппить новые файлы на существующие codebase-аналоги |
| `workflow.subagent_timeout` | number | `600` | Таймаут в секундах для индивидуальных subagent-вызовов. Увеличь для долгих research- или execution-фаз |
| `executor.stall_detect_interval_minutes` | number | `5` | Минут между executor stall-проверками пока executor-агент активен. Execute-phase оркестратор использует эту каденцию чтобы инспектировать недавние коммиты и не ждать вечно молчащего агента. |
| `executor.stall_threshold_minutes` | number | `10` | Минут без executor-завершения или коммит-активности на expected-branch до того как execute-phase предлагает recovery-выборы для возможного заглохшего executor'а. |
| `workflow.inline_plan_threshold` | number | `3` | Максимум задач в фазе до того как планировщик генерит отдельный PLAN.md файл вместо инлайна задач в промпте |
| `workflow.drift_threshold` | number | `3` | Минимум новых структурных элементов (новые директории, barrel exports, миграции, route modules), введённых во время фазы до того как post-execute codebase-drift gate действует. См. [#2003](https://github.com/gsd-build/get-shit-done/issues/2003). Добавлено в v1.39 |
| `workflow.drift_action` | string | `warn` | Что делать когда `workflow.drift_threshold` превышен после `/gsd-execute-phase`. `warn` печатает сообщение, предлагая `/gsd-map-codebase --paths …`; `auto-remap` спавнит `gsd-codebase-mapper`, скоупленный на затронутые пути. Добавлено в v1.39 |
| `workflow.build_command` | string | (нет) | Shell-команда для билда проекта в post-merge build gate (Step A of step 5.6 в execute-phase). Когда не установлен, gate авто-детектит: Xcode (`.xcodeproj` есть) → `xcodebuild build`, `Makefile` с `build:` target → `make build`, Justfile → `just build`, `Cargo.toml` → `cargo build`, `go.mod` → `go build ./...`, Python → `python -m py_compile`, `package.json` с `build` script → `npm run build`. Запускается с 5-минутным таймаутом; падение инкрементит `WAVE_FAILURE_COUNT`. Добавлено в v1.39 |
| `workflow.test_command` | string | (нет) | Shell-команда для запуска тест-набора проекта в post-merge test gate (Step B of step 5.6 в execute-phase) и regression gate. Когда не установлен, gate авто-детектит: Xcode (`.xcodeproj` есть) → `xcodebuild test`, `Makefile` с `test:` target → `make test`, Justfile → `just test`, `package.json` → `npm test`, `Cargo.toml` → `cargo test`, `go.mod` → `go test ./...`, Python → `python -m pytest`. Запускается с 5-минутным таймаутом; падение инкрементит `WAVE_FAILURE_COUNT`. Добавлено в v1.39 |

## Настройки Code Quality

Namespace `code_quality.*` гейтит опциональную structural-analysis тулчейн, дополняющую `/gsd-code-review`. Настройки аддитивны: каждый инструмент независимо opt-in и по дефолту off.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `code_quality.fallow.enabled` | boolean | `false` | Включает fallow structural pre-pass для `/gsd-code-review`. Когда `false`, fallow-бинарь не probe'ится и JSON-артефакт не производится. |
| `code_quality.fallow.scope` | string | `phase` | Скоуп fallow-анализа: `phase` (текущий review-file scope) или `repo` (весь репозиторий). |
| `code_quality.fallow.profile` | string | `standard` | Селектор fallow-профиля, передаваемый pre-pass runner'у (`minimal`, `standard`, `strict`). |
| `code_quality.fallow.mcp` | boolean | `false` | **Зарезервировано — пока не реализовано.** Когда `true`, включает MCP-backed режим structural-находок для рантаймов, поддерживающих MCP server routing. Установка в `true` сейчас no-op и эмитит runtime warning. |

## Настройки Ship

`ship.pr_body_sections` добавляет дополнительные секции PR-body для project-specific PRD/PR-body контента в `/gsd-ship` без правки `get-shit-done/workflows/ship.md`.

Гайд пользователя с примерами онбординга и диагностикой — [Custom PR Body Sections](ship-pr-body-sections.md).

Этот список append-only: сконфигурированные записи добавляются после core-секций `Summary`, `Changes`, `Requirements Addressed`, `Verification` и `Key Decisions`. Они не могут заменить, удалить или переупорядочить обязательные секции.

Рекомендованные lean/agile PRD-применения включают user stories, acceptance criteria, Definition of Done или release-критерии, риски и зависимости, метрики успеха, заметки stakeholder-ревью. Держи эти секции короткими и evidence-oriented, чтобы PR-body оставалось живым release-артефактом, а не статичным дампом требований.

Каждая запись поддерживает:

| Поле | Тип | Дефолт | Описание |
|------|-----|--------|----------|
| `heading` | string | обязателен | Markdown section heading, рендерится как `## {heading}`. Должен быть одной строкой. |
| `enabled` | boolean | `true` | Когда `false`, онбординг может держать кандидат-секцию в конфиге не рендеря её в сгенерированных PR-body. |
| `source` | string | (нет) | Опциональная fallback-цепочка заголовков планировочных артефактов, например `PLAN.md ## Risks \|\| VERIFICATION.md ## Manual Checks`. Разрешённые артефакты: `ROADMAP.md`, `PLAN.md`, `SUMMARY.md`, `VERIFICATION.md`, `STATE.md`, `REQUIREMENTS.md`, `CONTEXT.md`. |
| `template` | string | (нет) | Литеральный Markdown с закрытыми токенами: `{phase_number}`, `{phase_name}`, `{phase_dir}`, `{base_branch}`, `{padded_phase}`. |
| `fallback` | string | (нет) | Литеральный Markdown, используемый когда `source` не даёт контента и `template` не предоставлен. |

Хотя бы один из `source`, `template` или `fallback` обязателен для каждой секции. Дефолт — `[]`, существующие проекты сохраняют текущий вывод `/gsd-ship` пока онбординг не добавит enabled-записи.

Пример:

```json
{
  "ship": {
    "pr_body_sections": [
      {
        "heading": "User Stories & Acceptance Criteria",
        "enabled": true,
        "source": "REQUIREMENTS.md ## User Stories || REQUIREMENTS.md ## Acceptance Criteria",
        "fallback": "- Acceptance criteria are covered by the linked requirements and verification evidence."
      },
      {
        "heading": "Risks & Rollback",
        "enabled": true,
        "source": "PLAN.md ## Risks || PLAN.md ## Rollback",
        "fallback": "- Rollback: revert this PR."
      },
      {
        "heading": "Stakeholder Sign-off",
        "enabled": false,
        "template": "- Product owner: pending for {phase_name}"
      }
    ]
  }
}
```

### Рекомендованные пресеты

| Сценарий | mode | granularity | profile | research | plan_check | verifier |
|----------|------|-------------|---------|----------|------------|----------|
| Прототипирование | `yolo` | `coarse` | `budget` | `false` | `false` | `false` |
| Нормальная разработка | `interactive` | `standard` | `balanced` | `true` | `true` | `true` |
| Production-релиз | `interactive` | `fine` | `quality` | `true` | `true` | `true` |

---

## Planning-настройки

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `planning.commit_docs` | boolean | `true` | Коммитятся ли файлы `.planning/` в git |
| `planning.search_gitignored` | boolean | `false` | Добавлять `--no-ignore` к широким search'ам чтобы включить `.planning/` |
| `planning.sub_repos` | array of strings | `[]` | Пути вложенных sub-реп относительно корня проекта. Когда установлен, GSD-aware tooling скоупит phase-lookup, path-resolution и commit-операции per sub-repo вместо трактовки внешней репы как монорепо |

### Резолв project-root в multi-repo workspace'ах

Когда `sub_repos` установлен и `gsd-tools.cjs` или `gsd-sdk query` вызваны изнутри указанной child-репы, оба CLI идут вверх к parent-workspace, владеющему `.planning/`, до диспатча хендлеров. Порядок резолва (проверяется на каждом предке до 10 уровней, никогда выше `$HOME`):

1. Если стартовая директория уже имеет свой `.planning/`, она — project root (без walk-up).
2. Parent имеет `.planning/config.json`, перечисляющий top-level сегмент стартовой директории в `sub_repos` (или легаси-форма `planning.sub_repos`).
3. Parent имеет `.planning/config.json` с легаси `multiRepo: true`, и стартовая директория внутри git-репы.
4. Parent имеет `.planning/`, и предок до кандидат-родителя содержит `.git` (эвристический fallback).

Если ничего не совпадает, стартовая директория возвращается без изменений. Явный `--project-dir /path/to/workspace` идемпотентен под этим резолвом.

### Авто-детект

Если `.planning/` в `.gitignore`, `commit_docs` автоматически `false` независимо от config.json. Это предотвращает git-ошибки.

---

## Hook-настройки

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `hooks.context_warnings` | boolean | `true` | Показывать предупреждения использования контекстного окна через context-monitor хук |
| `hooks.workflow_guard` | boolean | `false` | Предупреждать когда правки файлов происходят вне GSD-воркфлоу (советует использовать `/gsd-quick` или `/gsd-fast`) |
| `statusline.show_last_command` | boolean | `false` | Дописывать суффикс `last: /<cmd>` к statusline, показывая последнюю вызванную slash-команду. Opt-in; читает активный session-transcript чтобы извлечь последний `<command-name>` тег (закрывает #2538) |

Prompt injection guard hook (`gsd-prompt-guard.js`) всегда активен и не может быть отключён — это security-фича, не workflow-переключатель.

### Private planning setup

Чтобы держать планировочные артефакты вне git:

1. Поставь `planning.commit_docs: false` и `planning.search_gitignored: true`
2. Добавь `.planning/` в `.gitignore`
3. Если ранее отслеживалось: `git rm -r --cached .planning/ && git commit -m "chore: stop tracking planning docs"`

---

## Инъекция скиллов агента

Инжектить кастомные skill-файлы в промпты GSD sub-агентов. Скиллы читаются агентами на spawn-time, давая им project-specific инструкции сверх того, что предоставляет CLAUDE.md.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `agent_skills` | object | `{}` | Map типов агентов на пути skill-директорий |

### Конфигурация

Добавь секцию `agent_skills` в `.planning/config.json`, маппя типы агентов на массивы путей skill-директорий (относительно корня проекта):

```json
{
  "agent_skills": {
    "gsd-executor": ["skills/testing-standards", "skills/api-conventions"],
    "gsd-planner": ["skills/architecture-rules"],
    "gsd-verifier": ["skills/acceptance-criteria"]
  }
}
```

Каждый путь должен быть директорией, содержащей файл `SKILL.md`. Пути валидируются на безопасность (без traversal за пределы корня проекта).

### Поддерживаемые типы агентов

Любой GSD-тип агента может получать скиллы. Распространённые типы:

- `gsd-executor` — исполняет implementation-планы
- `gsd-planner` — создаёт phase-планы
- `gsd-checker` — верифицирует качество планов
- `gsd-verifier` — постфактум-верификация
- `gsd-researcher` — research фазы
- `gsd-project-researcher` — research нового проекта
- `gsd-debugger` — диагностические агенты
- `gsd-codebase-mapper` — анализ кодовой базы
- `gsd-advisor` — discuss-phase advisor'ы
- `gsd-ui-researcher` — создание UI дизайн-контракта
- `gsd-ui-checker` — верификация UI-спеки
- `gsd-roadmapper` — создание roadmap'а
- `gsd-synthesizer` — research-синтез

### Как работает

На spawn-time workflow'ы вызывают `gsd-sdk query agent-skills <type>` (или легаси `node gsd-tools.cjs agent-skills <type>`) чтобы загрузить сконфигурированные скиллы. Если скиллы для типа агента существуют, они инжектятся как блок `<agent_skills>` в промпт Task():

```xml
<agent_skills>
Read these user-configured skills:
- @skills/testing-standards/SKILL.md
- @skills/api-conventions/SKILL.md
</agent_skills>
```

Если скиллы не сконфигурированы, блок опускается (zero overhead).

### CLI

Установка скиллов через CLI:

```bash
gsd-sdk query config-set agent_skills.gsd-executor '["skills/my-skill"]'
```

---

## Feature flags

Переключение опциональных способностей через namespace конфига `features.*`. Feature flags по дефолту `false` (отключены) — включение флага opt-in'ит в новое поведение без затрагивания существующих workflow.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `features.thinking_partner` | boolean | `false` | Включить thinking-partner анализ в workflow decision-points |
| `features.global_learnings` | boolean | `false` | Включить cross-project learnings пайплайн (авто-копирование на phase-completion, planner-инъекция) |
| `learnings.max_inject` | number | `10` | Максимум cross-project learnings, инжектируемых в каждый planner-промпт. Низкие значения уменьшают размер промпта; высокие дают более широкий исторический контекст |
| `intel.enabled` | boolean | `false` | Включить queryable codebase intelligence систему. Когда `true`, команды `/gsd-map-codebase --query` строят и query'ят JSON-индекс в `.planning/intel/`. Добавлено в v1.34 |

<a id="graphify-settings"></a>
### Graphify-настройки

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `graphify.enabled` | boolean | `false` | Включить project knowledge graph. Когда `true`, `/gsd-graphify` строит и query'ит граф в `.planning/graphs/`. Добавлено в v1.36 |
| `graphify.build_timeout` | number (секунды) | `300` | Максимум секунд для запуска `/gsd-graphify build` до прерывания. Добавлено в v1.36 |

#### Multi-developer setup

Если несколько разработчиков будут перестраивать граф в одной репе, запусти один раз на клон после включения graphify:

```bash
graphify hook install
```

Это устанавливает git merge driver, который union-merge'ит конкурентные `graph.json` записи (без conflict-маркеров в knowledge graph), плюс post-commit rebuild hook. Пишет `.gitattributes` и регистрирует `graphify merge-driver` в `.git/config`. Соло-проекты могут пропустить шаг; запуск всё равно безвреден. Введено upstream в graphify v0.7.0 вместе с сигналом freshness `built_at_commit`, который всплывает `/gsd-graphify status`.

#### Commit-based staleness

`/gsd-graphify status` репортит два ортогональных сигнала staleness:

- **`stale`** (mtime-based, 24-часовое окно) — когда graph-файл был последний раз записан. Полезно когда graphify не запускается автоматически.
- **`commit_stale`** (commit-based, требует graphify v0.7+) — был ли граф построен против текущего `git HEAD`. Достоверный когда присутствует. Tri-state: `true` / `false` / `null`. `null` означает что сигнал недоступен (pre-v0.7 граф, нет git, или unreachable commit) — fallback на mtime-флаг.

CI-построенный граф, перестроенный минуты назад против старого чекаута, прочитается как fresh на mtime но `commit_stale: true`. Всплывай оба при ответе на архитектурные вопросы.

### Использование

```bash
# Включить фичу
gsd-sdk query config-set features.global_learnings true

# Отключить фичу
gsd-sdk query config-set features.thinking_partner false
```

Namespace `features.*` — это dynamic key-паттерн — новые feature flags могут быть добавлены без модификации `VALID_CONFIG_KEYS`. Любой ключ, совпадающий с `features.<name>`, принимается системой конфига.

---

## Настройки параллелизации

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `parallelization` | boolean | `true` | Сокращение для `parallelization.enabled`. Установка `parallelization false` отключает параллельное исполнение, не меняя других sub-ключей |
| `parallelization.enabled` | boolean | `true` | Запускать независимые планы одновременно |
| `parallelization.plan_level` | boolean | `true` | Параллелизация на уровне плана |
| `parallelization.task_level` | boolean | `false` | Параллелизация задач в плане |
| `parallelization.skip_checkpoints` | boolean | `true` | Пропускать чекпоинты во время параллельного исполнения |
| `parallelization.max_concurrent_agents` | number | `3` | Максимум одновременных агентов |
| `parallelization.min_plans_for_parallel` | number | `2` | Минимум планов для триггера параллельного исполнения |

> **Pre-commit хуки и параллельное исполнение**: Когда параллелизация включена, executor-агенты коммитят с `--no-verify` чтобы избежать build lock contention (например, cargo lock fights в Rust-проектах). Оркестратор валидирует хуки один раз после завершения каждой волны. Записи в STATE.md защищены file-level локами чтобы предотвратить порчу при конкурентной записи. Если нужно чтобы хуки прогонялись per-commit, поставь `parallelization.enabled: false`.

---

## STATE.md frontmatter (Phase Lifecycle)

`STATE.md` несёт YAML frontmatter, который status-line хук читает на каждом рендере. v1.40 добавляет четыре опциональных phase-lifecycle поля, читаемых `parseStateMd()` и рендерящихся `formatGsdState()`:

| Поле | Тип | Назначение |
|------|-----|------------|
| `active_phase` | string (например, `"4.5"`) | Номер фазы когда orchestrator-команда в полёте |
| `next_action` | string | Рекомендованная следующая команда в idle-состоянии (`discuss-phase` / `plan-phase` / `execute-phase` / `verify-phase`) |
| `next_phases` | YAML flow array | Фазы, к которым применяется `next_action` (например, `["4.5"]`) |
| `progress` | block | Вложенные `total_phases` / `completed_phases` / `percent` для progress-бара майлстоуна |

Все четыре поля **опциональны и аддитивны** — STATE.md без них продолжают рендериться точно как в v1.38.x. См. [`STATE-MD-LIFECYCLE.md`](STATE-MD-LIFECYCLE.md) для полного field-справочника, parser-constraints и rendering-сцен.

---

## Git-ветвление

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `git.branching_strategy` | enum | `none` | `none`, `phase` или `milestone` |
| `git.base_branch` | string | `main` | Интеграционная ветка, от которой создаются и в которую мерджатся phase/milestone-ветки. Override когда репа использует `master` или release-ветку |
| `git.create_tag` | boolean | `true` | Создавать git-тег (`v[X.Y]`) на завершении майлстоуна. Поставь `false` для проектов со своим release-flow |
| `git.phase_branch_template` | string | `gsd/phase-{phase}-{slug}` | Шаблон имени ветки для phase-стратегии |
| `git.milestone_branch_template` | string | `gsd/{milestone}-{slug}` | Шаблон имени ветки для milestone-стратегии |
| `git.quick_branch_template` | string or null | `null` | Опциональный шаблон имени ветки для `/gsd-quick` задач |

### Сравнение стратегий

| Стратегия | Создаёт ветку | Scope | Merge-point | Лучшее для |
|-----------|---------------|-------|-------------|------------|
| `none` | Никогда | N/A | N/A | Соло-разработка, простые проекты |
| `phase` | На старте `execute-phase` | Одна фаза | Пользователь мерджит после фазы | Code-review per фазе, granular rollback |
| `milestone` | На первом `execute-phase` | Все фазы майлстоуна | На `complete-milestone` | Release-ветки, PR per версии |

### Template-переменные

| Переменная | Доступна в | Пример |
|------------|------------|--------|
| `{phase}` | `phase_branch_template` | `03` (zero-padded) |
| `{slug}` | Обоих шаблонах | `user-authentication` (lowercase, hyphenated) |
| `{milestone}` | `milestone_branch_template` | `v1.0` |
| `{num}` / `{quick}` | `quick_branch_template` | `260317-abc` (quick task ID) |

Пример quick-task ветвления:

```json
"git": {
  "quick_branch_template": "gsd/quick-{num}-{slug}"
}
```

### Merge-опции на завершении майлстоуна

| Опция | Git-команда | Результат |
|-------|-------------|-----------|
| Squash merge (рекомендуется) | `git merge --squash` | Один чистый коммит на ветку |
| Merge с историей | `git merge --no-ff` | Сохраняет все индивидуальные коммиты |
| Удалить без мерджа | `git branch -D` | Отбросить работу ветки |
| Сохранить ветки | (нет) | Ручная обработка позже |

---

## Настройки Gate'ов

Контроль confirmation-промптов во время workflow.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `gates.confirm_project` | boolean | `true` | Подтвердить детали проекта до финализации |
| `gates.confirm_phases` | boolean | `true` | Подтвердить разбивку по фазам |
| `gates.confirm_roadmap` | boolean | `true` | Подтвердить roadmap до продолжения |
| `gates.confirm_breakdown` | boolean | `true` | Подтвердить разбивку задач |
| `gates.confirm_plan` | boolean | `true` | Подтвердить каждый план до исполнения |
| `gates.execute_next_plan` | boolean | `true` | Подтвердить до исполнения следующего плана |
| `gates.issues_review` | boolean | `true` | Просмотреть проблемы до создания fix-планов |
| `gates.confirm_transition` | boolean | `true` | Подтвердить phase-transition |

---

## Safety-настройки

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `safety.always_confirm_destructive` | boolean | `true` | Подтверждать destructive-операции (удаления, перезаписи) |
| `safety.always_confirm_external_services` | boolean | `true` | Подтверждать взаимодействия с внешними сервисами |

---

## Security-настройки

Настройки для security enforcement (v1.31). Все следуют паттерну **absent = enabled**. Эти ключи живут под `workflow.*` в `.planning/config.json` — совпадая с зашиппленным шаблоном и runtime-чтениями в `workflows/plan-phase.md`, `workflows/execute-phase.md`, `workflows/secure-phase.md`, `workflows/verify-work.md`.

Эти ключи живут под `workflow.*` — туда workflow'ы и установщик пишут и читают их. Установка на верхнем уровне `config.json` молча игнорируется.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `workflow.security_enforcement` | boolean | `true` | Включить threat-model-anchored security-верификацию через `/gsd-secure-phase`. Когда `false`, security-проверки полностью пропускаются |
| `workflow.security_asvs_level` | number (1-3) | `1` | OWASP ASVS verification level. Level 1 = opportunistic, Level 2 = standard, Level 3 = comprehensive |
| `workflow.security_block_on` | string | `"high"` | Минимальная severity, блокирующая phase-advancement. Опции: `"high"`, `"medium"`, `"low"` |

---

## Decision Coverage Gates (`workflow.context_coverage_gate`)

Когда `discuss-phase` пишет implementation-решения в `<decisions>` в CONTEXT.md, два gate'а гарантируют что эти решения переживают переход в планы и зашиппленный код (issue #2492).

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `workflow.context_coverage_gate` | boolean | `true` | Переключатель для обоих decision-coverage gate'ов. Когда `false`, и plan-phase translation gate, и verify-phase validation gate пропускаются молча. |

### Что gate'ы делают

**Plan-phase translation gate (БЛОКИРУЮЩИЙ).** Запускается сразу после существующего requirements coverage gate, до коммита планов. Для каждого отслеживаемого решения в `<decisions>` он проверяет, что id решения (`D-NN`) или его текст появляется в `must_haves`, `truths` или теле хотя бы одного плана. Промах всплывает с id и отказывается пометить фазу planned.

**Verify-phase validation gate (НЕ-БЛОКИРУЮЩИЙ).** Запускается рядом с другими verify-шагами. Ищет в каждом зашиппленном артефакте (PLAN.md, SUMMARY.md, изменённых файлах, недавних commit-subject'ах) каждое отслеживаемое решение. Промахи пишутся в VERIFICATION.md как warning-секция, но **не** флипают общий статус верификации. Асимметрия намеренная — к verify-time работа сделана, нечёткое substring-промахи не должны фейлить иначе зелёную фазу.

### Как писать решения, чтобы gate'ы их приняли

Шаблон discuss-phase уже производит `D-NN`-нумерованные решения. Gate доволен когда:

1. Каждый план, реализующий решение, **цитирует id** где-нибудь — `must_haves.truths: ["D-12: bit offsets exposed"]` или упоминание `D-12:` в теле плана. Strict id match — самый дешёвый, детерминированный путь.
2. Soft phrase matching — fallback для парафраз — если фраза из 6+ слов из текста решения появляется дословно в плане/summary, засчитывается.

### Opt-out

Решение **не** подлежит gate'ам когда применяется одно из:

- Оно живёт под заголовком `### Claude's Discretion` внутри `<decisions>`.
- Оно тегнуто `[informational]`, `[folded]` или `[deferred]` в своём пункте (например, `- **D-08 [informational]:** Naming style for internal helpers`).

Используй эти escape hatches когда решение действительно не нуждается в plan-покрытии — implementation discretion, идеи на будущее, зафиксированные для записи, или айтемы уже отложенные к следующей фазе.

---

## Review-настройки

Конфигурировать per-CLI выбор модели для `/gsd-review`. Когда установлен, override default-модели CLI для этого ревьюера.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `review.models.gemini` | string | (CLI default) | Модель используемая когда вызывается `--gemini`-ревьюер |
| `review.models.claude` | string | (CLI default) | Модель используемая когда вызывается `--claude`-ревьюер |
| `review.models.codex` | string | (CLI default) | Модель используемая когда вызывается `--codex`-ревьюер |
| `review.models.opencode` | string | (CLI default) | Модель используемая когда вызывается `--opencode`-ревьюер |
| `review.models.qwen` | string | (CLI default) | Модель используемая когда вызывается `--qwen`-ревьюер |
| `review.models.cursor` | string | (CLI default) | Модель используемая когда вызывается `--cursor`-ревьюер |
| `review.models.ollama` | string | (server default) | Имя модели передаваемое Ollama когда вызывается `--ollama`-ревьюер. Если не установлено, используется первая доступная модель, репортируемая сервером (например, `llama3`). Поставь конкретный тег: `gsd config-set review.models.ollama codellama` |
| `review.models.lm_studio` | string | (server default) | Имя модели для LM Studio когда вызывается `--lm-studio`-ревьюер. Если не установлено, используется первая доступная. |
| `review.models.llama_cpp` | string | (server default) | Имя модели для llama.cpp когда вызывается `--llama-cpp`-ревьюер. Если не установлено, используется первая модель, репортируемая `/v1/models`. |
| `review.default_reviewers` | string[] \| null | (все детектированные) | Дефолтный subset ревьюеров для no-flag `/gsd-review`. Пример: `["gemini","codex"]`. Явные флаги и `--all` переопределяют. |
| `review.ollama_host` | string | `http://localhost:11434` | Base URL Ollama-сервера. Override когда Ollama на нестандартном порту или удалённом хосте: `gsd config-set review.ollama_host http://192.168.1.10:11434` |
| `review.lm_studio_host` | string | `http://localhost:1234` | Base URL LM Studio local-сервера. Override при использовании нестандартного порта. |
| `review.llama_cpp_host` | string | `http://localhost:8080` | Base URL llama.cpp-сервера (`llama-server`). Override при использовании нестандартного порта. |

### Пример

```json
{
  "review": {
    "models": {
      "gemini": "gemini-2.5-pro",
      "qwen": "qwen-max"
    }
  }
}
```

Fallback на сконфигурированный default каждого CLI когда ключ отсутствует. Добавлено в v1.35.0 (#1849).

---

## Manager Passthrough-флаги

Конфигурировать per-step флаги, которые `/gsd-manager` дописывает к каждой диспатчуемой команде. Позволяет настраивать как менеджер запускает discuss-, plan- и execute-шаги без ручного ввода флагов.

| Настройка | Тип | Дефолт | Описание |
|-----------|-----|--------|----------|
| `manager.flags.discuss` | string | (нет) | Флаги дописываются к discuss-phase командам (например, `"--auto"`) |
| `manager.flags.plan` | string | (нет) | Флаги дописываются к plan-phase командам (например, `"--skip-research"`) |
| `manager.flags.execute` | string | (нет) | Флаги дописываются к execute-phase командам (например, `"--validate"`) |

**Пример:**

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

Невалидные flag-токены санитизируются и логируются как warning'и. Только распознанные GSD-флаги передаются.

---

## Профили моделей

### Определения профилей

| Агент | `quality` | `balanced` | `budget` | `inherit` |
|-------|-----------|------------|----------|-----------|
| gsd-planner | Opus | Opus | Sonnet | Inherit |
| gsd-roadmapper | Opus | Sonnet | Sonnet | Inherit |
| gsd-executor | Opus | Sonnet | Sonnet | Inherit |
| gsd-phase-researcher | Opus | Sonnet | Haiku | Inherit |
| gsd-project-researcher | Opus | Sonnet | Haiku | Inherit |
| gsd-research-synthesizer | Sonnet | Sonnet | Haiku | Inherit |
| gsd-debugger | Opus | Sonnet | Sonnet | Inherit |
| gsd-codebase-mapper | Sonnet | Haiku | Haiku | Inherit |
| gsd-verifier | Sonnet | Sonnet | Haiku | Inherit |
| gsd-plan-checker | Sonnet | Sonnet | Haiku | Inherit |
| gsd-integration-checker | Sonnet | Sonnet | Haiku | Inherit |
| gsd-nyquist-auditor | Sonnet | Sonnet | Haiku | Inherit |
| gsd-pattern-mapper | Sonnet | Sonnet | Haiku | Inherit |
| gsd-ui-researcher | Opus | Sonnet | Haiku | Inherit |
| gsd-ui-checker | Sonnet | Sonnet | Haiku | Inherit |
| gsd-ui-auditor | Sonnet | Sonnet | Haiku | Inherit |
| gsd-doc-writer | Opus | Sonnet | Haiku | Inherit |
| gsd-doc-verifier | Sonnet | Sonnet | Haiku | Inherit |

> **Все 33 поставочных агента имеют явные per-profile tier-назначения** в каталоге (`sdk/shared/model-catalog.json`). Таблица выше показывает репрезентативный subset наиболее используемых агентов. Для агентов, не указанных здесь, `model_overrides` принимает любое имя поставочного агента. Авторитетные данные профилей выводятся из `sdk/shared/model-catalog.json` через `get-shit-done/bin/lib/model-catalog.cjs` и `sdk/src/model-catalog.ts`.

### Per-agent override'ы

Override конкретных агентов без смены всего профиля:

```json
{
  "model_profile": "balanced",
  "model_overrides": {
    "gsd-executor": "opus",
    "gsd-planner": "haiku"
  }
}
```

Валидные override-значения: `opus`, `sonnet`, `haiku`, `inherit` или любой полностью квалифицированный model ID (например, `"openai/o3"`, `"google/gemini-2.5-pro"`).

`model_overrides` можно установить либо в `.planning/config.json` (per-project), либо в `~/.gsd/defaults.json` (глобально). Per-project записи выигрывают при конфликте, и непротиворечащие глобальные записи сохраняются, так что можно тюнить модель одного агента в одной репе без re-установки глобальных дефолтов. Это применяется единообразно к Claude Code, Codex, OpenCode, Kilo и другим поддерживаемым рантаймам. На Codex и OpenCode резолвенная модель встраивается в static config каждого агента на install-time — `spawn_agent` и `task` интерфейс OpenCode не принимают inline `model` параметр, поэтому запуск `gsd install <runtime>` после правки `model_overrides` обязателен чтобы изменение применилось. См. issue #2256.

### Per-Phase-Type Models (`models`) — добавлено в v1.41

> Express-тюнинг на **уровне фазы** (planning, research, execution, verification) без изучения таксономии агентов. Добавлено в [#3023](https://github.com/gsd-build/get-shit-done/pull/3030).

`model_overrides` — per-**agent** (точно, но verbose; нужно знать что `gsd-codebase-mapper` — research, а `gsd-doc-writer` — execution). Блок `models` позволяет сказать «Opus для planning и execution, Sonnet для остального» в две строки:

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
  },
  "model_overrides": {
    "gsd-codebase-mapper": "haiku"
  }
}
```

#### Маппинг phase-type → агенты

| Phase type | Агенты |
|---|---|
| `planning` | `gsd-planner`, `gsd-roadmapper`, `gsd-pattern-mapper` |
| `discuss` | (зарезервировано — sub-агента сегодня нет) |
| `research` | `gsd-phase-researcher`, `gsd-project-researcher`, `gsd-research-synthesizer`, `gsd-codebase-mapper`, `gsd-ui-researcher` |
| `execution` | `gsd-executor`, `gsd-debugger`, `gsd-doc-writer` |
| `verification` | `gsd-verifier`, `gsd-plan-checker`, `gsd-integration-checker`, `gsd-nyquist-auditor`, `gsd-ui-checker`, `gsd-ui-auditor`, `gsd-doc-verifier` |
| `completion` | (зарезервировано — sub-агента сегодня нет) |

`discuss` и `completion` принимаются схемой для forward compatibility; установка их сегодня no-op пока sub-агент к ним не маппится.

#### Приоритет резолва (высокий → низкий)

```text
1. model_overrides[<agent>]              ← per-agent; full IDs; таргетное исключение
2. dynamic_routing.tier_models[<tier>]   ← когда включено (см. §Dynamic Routing)
3. models[<phase_type>]                  ← coarse phase-level tier (эта секция)
4. model_profile (per-agent колонка)     ← глобальная tier-стратегия
5. Runtime default                       ← когда ничего другого не применимо
```

Пять слоёв компонуются top-down: `model_profile` — базовый tier, `models[<phase_type>]` override на phase-level, `dynamic_routing` (когда включён) эскалирует per-attempt на soft failure, `model_overrides[<agent>]` вырезает per-agent исключения наверху, runtime default применяется когда ничего другого не подходит. В примере выше все пять research-агентов резолвятся в `sonnet` *кроме* `gsd-codebase-mapper`, которого per-agent override пиннит к `haiku`. `dynamic_routing` отключён по дефолту — когда off (`enabled: false` или блок отсутствует), поведение этой секции не меняется со старого.

#### Принимаемые значения

`models.<phase_type>` принимает только tier-алиасы:

| Значение | Эффект |
|---|---|
| `"opus"` / `"sonnet"` / `"haiku"` | Стандартный tier — runtime-резолв маппит на модель активного рантайма для этого tier'а |
| `"inherit"` | Агенты в этой фазе следуют session-модели (та же семантика что `model_profile: "inherit"`) |

Если нужен полностью квалифицированный model ID (`"openai/gpt-5"`, `"google/gemini-2.5-pro"`), используй `model_overrides` per агента. `models.*` намеренно tier-only чтобы runtime-aware маппинг оставался корректным на Codex / OpenCode / Gemini CLI установках.

#### Когда что использовать

| Ты хочешь | Используй |
|---|---|
| Одну глобальную tier-стратегию («balanced everywhere») | `model_profile` |
| Coarse phase-level тюнинг («Opus для planning») | `models.<phase_type>` |
| Per-agent точность («force haiku на codebase mapper») | `model_overrides[<agent>]` |
| Полный model ID для конкретного агента | `model_overrides[<agent>]: "openai/gpt-5"` |

Микшируй свободно — правило приоритета выше резолвит любое overlap детерминированно.

#### Валидация

`config-set` отвергает неизвестные phase-type'ы:

```bash
$ gsd config-set models.deployment opus
Error: 'models.deployment' is not a valid config key

# Валидно:
$ gsd config-set models.research sonnet
```

Прямые правки `.planning/config.json` мягче — резолвер просто игнорирует значения, которые не распознаёт, и проваливается к profile tier'у — так что опечатка не сломает tier-резолв молча.

### Dynamic Routing с failure-tier эскалацией (`dynamic_routing`) — добавлено в v1.41

> Стартуй дёшево, эскалируйся только когда агент фейлит gate. Добавлено в [#3024](https://github.com/gsd-build/get-shit-done/pull/3031).

`dynamic_routing` позволяет платить за дешёвый tier по дефолту и эскалироваться к более дорогому tier'у только когда оркестратор детектит soft failure (верификация неоднозначна, plan-check FLAG и т.д.).

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

#### Дефолтные tier'ы агентов

Каждый агент в `MODEL_PROFILES` объявляет один из трёх дефолтных tier'ов. Резолвер выбирает `tier_models[default_tier]` для первой попытки.

| Tier | Агенты | Use case |
|---|---|---|
| `light` | gsd-codebase-mapper, gsd-doc-classifier, gsd-doc-verifier, gsd-integration-checker, gsd-intel-updater, gsd-nyquist-auditor, gsd-pattern-mapper, gsd-plan-checker, gsd-research-synthesizer, gsd-ui-auditor, gsd-ui-checker | Дёшево/быстро — pure mappers, scanners, low-stakes аудиты |
| `standard` | gsd-advisor-researcher, gsd-ai-researcher, gsd-code-fixer, gsd-code-reviewer, gsd-doc-synthesizer, gsd-doc-writer, gsd-domain-researcher, gsd-eval-auditor, gsd-executor, gsd-phase-researcher, gsd-project-researcher, gsd-ui-researcher, gsd-verifier | Дефолтная рабочая лошадь — research, writing, основная верификация |
| `heavy` | gsd-assumptions-analyzer, gsd-debug-session-manager, gsd-debugger, gsd-eval-planner, gsd-framework-selector, gsd-planner, gsd-roadmapper, gsd-security-auditor, gsd-user-profiler | Глубокое рассуждение — уже наверху, эскалироваться дальше некуда |

#### Поток эскалации

```text
1. Оркестратор спавнит агента → резолвер возвращает tier_models[default_tier]
2. Soft failure?
   ├─ нет → ✓ готово (дешёвый путь)
   └─ да → оркестратор re-спавнит на attempt+1
            → резолвер возвращает tier_models[next_tier_up]
            → cap на max_escalations
3. Hard failure (exception/crash) → bypass эскалации, всплыть немедленно
```

Если `dynamic_routing.escalate_on_failure: false`, soft failures **не** продвигают tier — каждый re-спавн продолжает использовать `tier_models[default_tier]` независимо от attempt-counter'а. Kill-switch override'ит soft-failure ветку выше.

`light → standard → heavy → heavy` (heavy остаётся на heavy; дальше идти некуда).

#### Приоритет резолва (высокий → низкий)

1. **`model_overrides[<agent>]`** — full ID принимаются; таргетное исключение
2. **`dynamic_routing.tier_models[<tier>]`** (когда `enabled: true`)
3. **`models[<phase_type>]`** — coarse phase-level (#3023)
4. **`model_profile`** — per-agent колонка из активного профиля
5. **Runtime default**

Блок `dynamic_routing` **отключён по дефолту** — `enabled: false` (или опускание блока) сохраняет сегодняшний static-резолв точно.

#### Настройки

| Ключ | Тип | Дефолт | Описание |
|---|---|---|---|
| `dynamic_routing.enabled` | boolean | `false` | Главный переключатель. Когда `true`, dynamic-routing резолвер используется для tier-выбора. |
| `dynamic_routing.tier_models.light` | enum | (нет) | Tier-alias для light. Типично `haiku`. |
| `dynamic_routing.tier_models.standard` | enum | (нет) | Tier-alias для standard. Типично `sonnet`. |
| `dynamic_routing.tier_models.heavy` | enum | (нет) | Tier-alias для heavy. Типично `opus`. |
| `dynamic_routing.escalate_on_failure` | boolean | `true` | Когда false, эскалация отключена (каждая попытка использует default tier). |
| `dynamic_routing.max_escalations` | integer | `1` | Жёсткий cap на retry на вызов агента. Предотвращает runaway-циклы. |

#### Когда что использовать

| Ты хочешь | Используй |
|---|---|
| Одну tier-стратегию на всех агентов | `model_profile` |
| Coarse phase-level тюнинг | `models.<phase_type>` |
| Per-agent точность (full ID) | `model_overrides` |
| **Cheap-by-default, эскалируйся только на failure** | **`dynamic_routing`** |

`dynamic_routing` структурно — *cost lever*: ты платишь Opus-rate только за сложные кейсы, заслуживающие Opus. Компонуй с `model_overrides` для per-agent исключений (override всегда выигрывает).

### Не-Claude рантаймы (Codex, OpenCode, Gemini CLI, Kilo)

Когда GSD установлен под не-Claude рантайм, установщик автоматически ставит `resolve_model_ids: "omit"` в `~/.gsd/defaults.json`. Это заставляет GSD возвращать пустой model-параметр для всех агентов, так что каждый агент использует ту модель, под которую сконфигурирован рантайм. Дополнительной настройки для дефолтного случая не нужно.

Если хочешь разным агентам разные модели, используй `model_overrides` с полностью квалифицированными model ID, которые твой рантайм распознаёт:

```json
{
  "resolve_model_ids": "omit",
  "model_overrides": {
    "gsd-planner": "o3",
    "gsd-executor": "o4-mini",
    "gsd-debugger": "o3",
    "gsd-codebase-mapper": "o4-mini"
  }
}
```

Намерение то же что и tier'ов Claude-профилей — использовать более сильную модель для планирования и дебага (где качество рассуждения важнее всего) и более дешёвую для исполнения и маппинга (где план уже содержит рассуждение).

**Когда какой подход:**

| Сценарий | Настройка | Эффект |
|----------|-----------|--------|
| Не-Claude рантайм, одна модель | `resolve_model_ids: "omit"` (дефолт установщика) | Все агенты используют дефолтную модель рантайма |
| Не-Claude рантайм, tier'ы | `resolve_model_ids: "omit"` + `model_overrides` | Именованные агенты используют конкретные модели, остальные — дефолт рантайма |
| Claude Code с OpenRouter/local провайдером | `model_profile: "inherit"` | Все агенты следуют session-модели |
| Claude Code с OpenRouter, tier'ы | `model_profile: "inherit"` + `model_overrides` | Именованные агенты используют конкретные модели, остальные наследуют |

**Значения `resolve_model_ids`:**

| Значение | Поведение | Используй когда |
|----------|-----------|-----------------|
| `false` (дефолт) | Возвращает Claude-aliases (`opus`, `sonnet`, `haiku`) | Claude Code с нативным Anthropic API |
| `true` | Маппит aliases в полные Claude model ID (`claude-opus-4-7`) | Claude Code с API, требующим полные ID |
| `"omit"` | Возвращает пустую строку (рантайм выбирает свой default) | Не-Claude рантаймы (Codex, OpenCode, Gemini CLI, Kilo) |

### Runtime-Aware профили (#2517)

Когда `runtime` установлен, profile tier'ы (`opus`/`sonnet`/`haiku`) резолвятся в runtime-native model ID вместо Claude-aliases. Это даёт одному shared `.planning/config.json` чисто работать через Claude и Codex.

**Built-in tier-маппы:**

| Рантайм | `opus` | `sonnet` | `haiku` | reasoning_effort |
|---------|--------|----------|---------|------------------|
| `claude` | `claude-opus-4-7` | `claude-sonnet-4-6` | `claude-haiku-4-5` | (не используется) |
| `codex` | `gpt-5.4` | `gpt-5.3-codex` | `gpt-5.4-mini` | `xhigh` / `medium` / `medium` |
| `gemini` | `gemini-3-pro` | `gemini-3-flash` | `gemini-2.5-flash-lite` | (не используется) |
| `qwen` | `qwen3-max-2026-01-23` | `qwen3-coder-plus` | `qwen3-coder-next` | (не используется) |
| `opencode` | `anthropic/claude-opus-4-7` | `anthropic/claude-sonnet-4-6` | `anthropic/claude-haiku-4-5` | (не используется) |
| `copilot` | `claude-opus-4-7` | `claude-sonnet-4-6` | `claude-haiku-4-5` | (не используется) |
| `hermes` | `anthropic/claude-opus-4-7` | `anthropic/claude-sonnet-4-6` | `anthropic/claude-haiku-4-5` | (не используется) |
| Группа B (`kilo`, `cline`, `cursor`, `windsurf`, `augment`, `trae`, `codebuddy`, `antigravity`) | (built-in default нет — твой рантайм обрабатывает выбор модели) | | | |

**Пример Codex** — один конфиг, tier'ы моделей, без большого блока `model_overrides`:

```json
{
  "runtime": "codex",
  "model_profile": "balanced"
}
```

Это резолвит `gsd-planner` → `gpt-5.4` (xhigh), `gsd-executor` → `gpt-5.3-codex` (medium), `gsd-codebase-mapper` → `gpt-5.4-mini` (medium). Codex-установщик встраивает `model = "..."` и `model_reasoning_effort = "..."` в каждый сгенерированный agent-TOML.

**Пример Claude** — явный opt-in резолвит в полные Claude ID (`resolve_model_ids: true` не нужен):

```json
{
  "runtime": "claude",
  "model_profile": "quality"
}
```

**Per-runtime override'ы** — заменить один или несколько tier-defaults:

```json
{
  "runtime": "codex",
  "model_profile": "quality",
  "model_profile_overrides": {
    "codex": {
      "opus": "gpt-5-pro",
      "haiku": { "model": "gpt-5-nano", "reasoning_effort": "low" }
    }
  }
}
```

**Приоритет (высокий → низкий):**

1. `model_overrides[<agent>]` — явный per-agent ID всегда выигрывает.
2. **Runtime-aware tier resolution** (эта секция) — когда `runtime` установлен и профиль не `inherit`.
3. `resolve_model_ids: "omit"` — возвращает пустую строку когда `runtime` не установлен.
4. Claude-native default — `model_profile` tier как alias (текущий дефолт).
5. `inherit` — пробрасывает литеральный `inherit` для семантики `Task(model="inherit")`.

**Backward compatibility.** Setup'ы без `runtime` не видят изменений поведения — каждый существующий конфиг продолжает работать идентично. Codex-установки, авто-ставящие `resolve_model_ids: "omit"`, продолжают опускать model-поле пока пользователь не opt-in'ит установкой `runtime: "codex"`.

**Неизвестные рантаймы.** Если `runtime` установлен в значение без built-in tier-маппа и без `model_profile_overrides[<runtime>]`, GSD fallback'ится на Claude-alias safe-default вместо эмита model ID, который рантайм не примет. Чтобы поддержать новый рантайм, заполни `model_profile_overrides.<runtime>.{opus,sonnet,haiku}` валидными ID.

### Философия профилей

| Профиль | Философия | Когда использовать |
|---------|-----------|--------------------|
| `quality` | Opus для всего decision-making, Sonnet для верификации | Квота доступна, критическая архитектурная работа |
| `balanced` | Opus только для планирования, Sonnet для всего остального | Нормальная разработка (дефолт) |
| `budget` | Sonnet для написания кода, Haiku для research/верификации | High-volume работа, менее критичные фазы |
| `inherit` | Все агенты используют текущую session-модель | Динамическое переключение моделей, **не-Anthropic провайдеры** (OpenRouter, локальные модели) |

---

## Переменные окружения

| Переменная | Назначение |
|------------|------------|
| `CLAUDE_CONFIG_DIR` | Override дефолтной конфиг-директории (`~/.claude/`) |
| `GEMINI_API_KEY` | Детектится context monitor'ом для переключения имени hook-события |
| `WSL_DISTRO_NAME` | Детектится установщиком для обработки WSL-путей |
| `GSD_SKIP_SCHEMA_CHECK` | Пропустить schema drift детекцию во время execute-phase (v1.31) |
| `GSD_PROJECT` | Override project-root для multi-project workspace поддержки (v1.32) |

---

## Глобальные дефолты

Сохрани настройки как глобальные дефолты для будущих проектов:

**Расположение:** `~/.gsd/defaults.json`

Когда `/gsd-new-project` создаёт новый `config.json`, он читает глобальные дефолты и мерджит их как стартовую конфигурацию. Per-project настройки всегда override'ят глобальные.
