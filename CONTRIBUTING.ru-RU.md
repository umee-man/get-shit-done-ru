# Контрибьютинг в GSD

> Это перевод. Поскольку PR подаются в upstream-репозиторий [`gsd-build/get-shit-done`](https://github.com/gsd-build/get-shit-done), официальный английский [CONTRIBUTING.md](CONTRIBUTING.md) — это рабочий документ. Этот перевод — справочное чтение для русскоязычных контрибьюторов.

## Начало работы

```bash
# Клонировать репу
git clone https://github.com/gsd-build/get-shit-done.git
cd get-shit-done

# Установить зависимости
npm install

# Запустить тесты
npm test
```

---

## Типы контрибьюций

GSD принимает три типа контрибьюций. У каждого свой процесс и свой барьер для принятия. **Прочитай эту секцию до того как что-либо открывать.**

### 🐛 Fix (баг-репорт)

Fix исправляет что-то сломанное, что падает, производит неправильный вывод или ведёт себя вопреки задокументированному поведению.

**Процесс:**
1. Открой [Bug Report issue](https://github.com/gsd-build/get-shit-done/issues/new?template=bug_report.yml) — заполни полностью.
2. Подожди пока мейнтейнер подтвердит, что это баг (лейбл: `confirmed-bug`). Для очевидных, воспроизводимых багов это обычно быстро.
3. Почини. Напиши тест, который бы поймал этот баг.
4. Открой PR используя [Fix PR-шаблон](.github/PULL_REQUEST_TEMPLATE/fix.md) — линкни подтверждённый issue.

**Причины отказа:** Невоспроизводимо, works-as-designed, дубликат существующего issue.

---

### ⚡ Enhancement

Enhancement улучшает существующую фичу — лучший output, более быстрое исполнение, более чистый UX, расширенная обработка edge-case'ов. Он **не** добавляет новые команды, новые workflow или новые концепции.

**Барьер:** Enhancement требует скоупленного письменного предложения, одобренного мейнтейнером, **до** написания любого кода. PR на enhancement будет закрыт без ревью если связанный issue не несёт лейбл `approved-enhancement`.

**Процесс:**
1. Открой [Enhancement issue](https://github.com/gsd-build/get-shit-done/issues/new?template=enhancement.yml) с полным предложением. Шаблон issue требует: решаемую проблему, конкретный benefit, scope изменений, рассмотренные альтернативы.
2. **Жди одобрения мейнтейнера.** Мейнтейнер должен пометить issue `approved-enhancement` до того как ты напишешь хоть строчку кода. Не открывай PR против неодобренного enhancement issue — он будет закрыт.
3. Пиши код. Держи scope ровно таким, как одобрен. Если случается scope creep — комментируй в issue и получай re-approval до продолжения.
4. Открой PR используя [Enhancement PR-шаблон](.github/PULL_REQUEST_TEMPLATE/enhancement.md) — линкни одобренный issue.

**Причины отказа:** Issue без лейбла `approved-enhancement`, scope превышает одобренный, нет письменного предложения, дубликат существующего поведения.

---

### ✨ Feature

Feature добавляет что-то новое — новую команду, новый workflow, новую концепцию, новую интеграцию. У фич самый высокий барьер, потому что они добавляют постоянную нагрузку поддержки к solo-developer инструменту, поддерживаемому маленькой командой.

**Барьер:** Фичи требуют полной письменной спецификации, одобренной мейнтейнером, до написания любого кода. PR на фичу будет закрыт без ревью если связанный issue не несёт лейбл `approved-feature`. Неполные спеки закрываются, а не правятся мейнтейнерами.

**Процесс:**
1. **Сначала обсуди** — проверь [Discussions](https://github.com/gsd-build/get-shit-done/discussions) — может, идею уже поднимали. Если поднимали и отклонили — не открывай новый issue.
2. Открой [Feature Request issue](https://github.com/gsd-build/get-shit-done/issues/new?template=feature_request.yml) с полной спекой. Шаблон требует: решаемую solo-developer проблему, что добавляется, полный scope затронутых файлов и систем, user stories, acceptance criteria, оценку нагрузки поддержки.
3. **Жди одобрения мейнтейнера.** Мейнтейнер должен пометить issue `approved-feature` до того как ты напишешь хоть строчку кода. Одобрение не гарантировано — GSD намеренно lean, и многие валидные идеи отклоняются как конфликтующие с дизайн-философией проекта.
4. Пиши код. Реализуй ровно одобренную спеку. Изменения scope требуют re-approval.
5. Открой PR используя [Feature PR-шаблон](.github/PULL_REQUEST_TEMPLATE/feature.md) — линкни одобренный issue.

**Причины отказа:** Issue без лейбла `approved-feature`, спека неполная, scope превышает одобренный, фича конфликтует с solo-developer фокусом GSD, нагрузка поддержки слишком высока.

---

### 📐 Предложение ADR или PRD

ADR (Architecture Decision Record) документирует значимое архитектурное решение. PRD (Product Requirements Document) фиксирует что и зачем фичи до её реализации. Оба управляются тем же issue-first правилом, что и всё остальное.

**Процесс:**

1. Открой issue подходящего типа (enhancement для ADR, пересматривающего существующую область; feature для новой архитектурной поверхности; chore для policy/docs-решений). Заполни полностью.
2. **Жди одобрения мейнтейнера.** Мейнтейнер должен пометить issue `approved-enhancement`, `approved-feature` или подтвердить chore до создания любого файла.
3. GitHub-номер issue становится префиксом имени файла. Создавай файл на ветке, названной по issue:
   - `docs/adr/<issue#>-<slug>.md` для ADR
   - `docs/prd/<issue#>-<slug>.md` для PRD
   - Ветка: `docs/<issue#>-<slug>`
4. Открой PR используя соответствующий шаблон и закрой issue через `Closes #<issue#>` в теле PR.

**Один issue = один ADR-или-PRD = один PR.** Не батчь несколько решений в один файл или один PR.

**Не вычисляй «следующий номер» локально.** Любой PR, использующий легаси `NNNN-*` секвенциальный паттерн для *нового* ADR или PRD, будет попрошен переименовать файл в формат `<issue#>-<slug>.md` до мерджа.

**Пример:** Issue #3485 был открыт, одобрен, и его номер стал префиксом: `docs/adr/3485-adr-prd-naming-convention.md` на ветке `docs/3485-adr-prd-naming-convention`.

**Причины отказа:** Issue не одобрен до создания файла, имя файла использует local-compute секвенциальный номер вместо issue#, несколько решений упакованы в один PR, файл размещён в неправильной директории (`docs/adr/` vs `docs/prd/`).

---

## Правило issue-first — без исключений

> **Никакого кода до одобрения.**

Для **fix**: открой issue, подтверди что это баг, потом чини.
Для **enhancement**: открой issue, получи `approved-enhancement`, потом кодь.
Для **feature**: открой issue, получи `approved-feature`, потом кодь.

PR, приходящие без правильно помеченного связанного issue, закрываются автоматически. Это не бюрократический барьер — он защищает тебя от траты времени на работу, которая будет отклонена, и защищает мейнтейнеров от ревью кода для изменений, на которые никогда не соглашались.

---

## Гайдлайны Pull Request

### Architecture & Domain Standards (определяются мейнтейнерами)

Следующие файлы — maintainer-owned coding-стандарты и должны рассматриваться как канонические при контрибьютинге:

- `CONTEXT.md` — domain language и стандарты именования модулей
- `docs/adr/` — Architecture Decision Records (ADR) для принятых архитектурных решений

Полные требования к контрибьюторам — включая формат CONTEXT.md, ADR governance и стандарты AI-assisted работы — в **[`docs/contributor-standards.md`](docs/contributor-standards.md)**.

Требования к контрибьюторам (краткая сводка):
- Прочитай `CONTEXT.md` до именования или рефакторинга модулей/интерфейсов/seam'ов.
- Используй словарь `CONTEXT.md` последовательно в комментариях кода, тестах, issue/PR-текстах и доках затронутой области.
- Проверяй релевантные ADR в `docs/adr/` перед предложением или реализацией архитектурных изменений.
- Если изменение намеренно пересматривает ADR-решение — явно упомяни это в связанном issue и PR-обосновании.
- Не переписывай maintainer intent в `CONTEXT.md`/ADR как drive-by cleanup; предлагай фокусные обновления, привязанные к одобренному scope.
- Если используешь AI-ассистент, попроси его прочитать `CONTEXT.md` и релевантные ADR до написания любого кода или доков, и убедись что он использовал правильный словарь до открытия PR.

**Каждый PR должен линковаться на одобренный issue.** PR без связанного issue закрываются без ревью, без исключений.

- **Никаких draft PR** — draft PR автоматически закрываются. Открывай PR только когда он завершён, протестирован и готов к ревью. Если работа не закончена — держи на локальной ветке.
- **Используй правильный PR-шаблон** — есть отдельные шаблоны для [Fix](.github/PULL_REQUEST_TEMPLATE/fix.md), [Enhancement](.github/PULL_REQUEST_TEMPLATE/enhancement.md) и [Feature](.github/PULL_REQUEST_TEMPLATE/feature.md). Использование не того шаблона или дефолтного для фичи — причина отказа.
- **Линкуй с closing-keyword** — используй `Closes #123`, `Fixes #123` или `Resolves #123` в теле PR. CI-проверка упадёт и PR авто-закроется если валидной issue-ссылки нет.
- **Один concern на PR** — багфиксы, enhancement'ы и фичи должны быть отдельными PR
- **Никаких drive-by форматирований** — не переформатируй код, не связанный с твоим изменением
- **CI должен пройти** — все matrix-job'ы (Ubuntu × Node 22, 24; macOS × Node 24) должны быть зелёными
- **Scope совпадает с одобренным issue** — если PR делает больше чем описано в issue, лишние изменения попросят убрать или вынести в новый issue

## CHANGELOG-записи — дроп фрагмента

**Не редактируй `CHANGELOG.md` напрямую.** Два PR, оба append'ящие в блок `### Fixed`, всегда конфликтуют при мердже — git не может выбрать serialization order без человека. Вместо этого каждый PR с user-facing изменениями дропает fragment-файл в `.changeset/`.

```bash
npm run changeset -- --type Fixed --pr <YOUR_PR_NUMBER> \
  --body "**\`/gsd-foo\` no longer drops trailing slashes** — объясни user-visible изменение."
```

Это пишет `.changeset/<adjective>-<noun>-<noun>.md`. Три случайных слова → конкурентные PR никогда не сталкиваются. Разрешённые значения `type:` следуют [Keep a Changelog](https://keepachangelog.com/): `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.

Фрагменты консолидируются в `CHANGELOG.md` на release-time release-воркфлоу. См. [`.changeset/README.md`](.changeset/README.md) для спеки формата и [#2975](https://github.com/gsd-build/get-shit-done/issues/2975) для обоснования.

**CI-форсинг:** Workflow `Changeset Required` (`scripts/changeset/lint.cjs`) фейлит любой PR, трогающий `bin/`, `get-shit-done/`, `agents/`, `commands/`, `hooks/` или `sdk/src/` без `.changeset/*.md` фрагмента.

**Opt-out:** PR без user-facing impact (test-рефакторинги, изменения lint-конфига, CI-tweaks, formatting-only изменения) могут добавить лейбл `no-changelog`. Lint его уважает. Если не уверен — **добавь фрагмент**.

## Стандарты тестирования

Все тесты используют встроенный test runner Node.js (`node:test`) и assertion-библиотеку (`node:assert`). **Не используй Jest, Mocha, Chai или любой внешний test framework.**

### Обязательные импорты

```javascript
const { describe, it, test, beforeEach, afterEach, before, after } = require('node:test');
const assert = require('node:assert/strict');
```

### Setup и cleanup

Есть два одобренных паттерна cleanup. Выбирай тот, что подходит ситуации.

**Паттерн 1 — Shared fixtures (`beforeEach`/`afterEach`):** Используй когда все тесты в `describe`-блоке шарят идентичный setup и teardown. Самый частый случай.

```javascript
// GOOD — shared setup/teardown с hook'ами
describe('my feature', () => {
  let tmpDir;

  beforeEach(() => {
    tmpDir = createTempProject();
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('does the thing', () => {
    assert.strictEqual(result, expected);
  });
});
```

**Паттерн 2 — Per-test cleanup (`t.after()`):** Используй когда индивидуальные тесты требуют уникальный teardown, отличающийся от других в том же блоке.

```javascript
// GOOD — per-test cleanup когда каждый тест нуждается в своём teardown
test('does the thing with a custom setup', (t) => {
  const tmpDir = createTempProject('custom-prefix');
  t.after(() => cleanup(tmpDir));

  assert.strictEqual(result, expected);
});
```

**Никогда не используй `try/finally` внутри test body.** Это многословно, маскирует test-падения и не одобренный паттерн в этом проекте.

```javascript
// BAD — try/finally внутри test body
test('does the thing', () => {
  const tmpDir = createTempProject();
  try {
    assert.strictEqual(result, expected);
  } finally {
    cleanup(tmpDir); // маскирует падения — не делай так
  }
});
```

> `try/finally` разрешён только внутри standalone утилит или helper-функций без доступа к test-context.

### Используй централизованные test-хелперы

Импортируй helpers из `tests/helpers.cjs` вместо inline-создания temp-директорий:

```javascript
const { createTempProject, createTempGitProject, createTempDir, cleanup, runGsdTools } = require('./helpers.cjs');
```

| Helper | Создаёт | Используй когда |
|--------|---------|------------------|
| `createTempProject(prefix?)` | tmpDir с `.planning/phases/` | Тестируешь GSD-инструменты, нуждающиеся в planning-структуре |
| `createTempGitProject(prefix?)` | То же + git init + initial commit | Тестируешь git-зависимые фичи |
| `createTempDir(prefix?)` | Голая temp-директория | Тестируешь фичи без `.planning/` |
| `cleanup(tmpDir)` | Удаляет директорию рекурсивно | Всегда в `afterEach` |
| `runGsdTools(args, cwd, env?)` | Исполняет gsd-tools.cjs | Тестируешь CLI-команды |

### Структура теста

```javascript
describe('featureName', () => {
  let tmpDir;

  beforeEach(() => {
    tmpDir = createTempProject();
    // Дополнительный setup специфичный для этого suite
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('handles normal case', () => {
    // Arrange
    // Act
    // Assert
  });

  test('handles edge case', () => {
    // ...
  });

  describe('sub-feature', () => {
    // Nested describe'ы могут иметь свои хуки
    beforeEach(() => {
      // Дополнительный setup для sub-фичи
    });

    test('sub-feature works', () => {
      // ...
    });
  });
});
```

### Форматирование fixture-данных

Template literals внутри test-блоков наследуют отступы от окружающего кода. Это может ввести неожиданный leading whitespace, ломающий regex-anchors и string-matching. Конструируй multi-line fixture-строки через `join()` массива:

```javascript
// GOOD — без bleed'а отступов
const content = [
  'line one',
  'line two',
  'line three',
].join('\n');

// BAD — template literal наследует окружающие отступы
const content = `
  line one
  line two
  line three
`;
```

### Запрещено: source-grep тесты

**Никогда не читай source-code `.cjs` файлы через `readFileSync`, чтобы проверить, есть ли в них строки.** Это source-grep theater: доказывает что литерал присутствует в файле, а не что фича работает в рантайме.

```javascript
// BAD — source-grep theater
const configSrc = fs.readFileSync(
  path.join(GSD_ROOT, 'bin', 'lib', 'config-schema.cjs'), 'utf-8'
);
assert.ok(
  configSrc.includes("'workflow.plan_bounce'"),
  'VALID_CONFIG_KEYS should contain workflow.plan_bounce'
);
```

Этот тест проходит, даже если `workflow.plan_bounce` присутствует, но опечатан в схеме, удалён из validation-пути или перемещён в другой файл под другим именем. Он выживает каждую поведенческую регрессию и фейлится только на тривиальных переименованиях.

Правильный паттерн для тестов config-ключей — через CLI:

```javascript
// GOOD — поведенческий тест через CLI
test('config-set accepts workflow.plan_bounce', (t) => {
  const tmpDir = createTempProject();
  t.after(() => cleanup(tmpDir));

  const result = runGsdTools('config-set workflow.plan_bounce true', tmpDir);
  assert.ok(result.success, `config-set should accept workflow.plan_bounce: ${result.error}`);

  const configPath = path.join(tmpDir, '.planning', 'config.json');
  const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
  assert.strictEqual(config.workflow?.plan_bounce, true, 'value must be persisted');
});
```

Этот один тест покрывает регистрацию ключа в `VALID_CONFIG_KEYS`, namespace-резолв ключа в `KNOWN_TOP_LEVEL` и персистенцию значения — все поведения, до которых source-grep тест не может дотянуться.

**Почему этот паттерн ломался в масштабе:** Коммит `990c3e64` в этой репе обновил 5 source-grep тестов за один проход, когда `VALID_CONFIG_KEYS` перемещался между файлами. Ноль из этих тестов тестировали поведение. Будь они поведенческими — миграция была бы невидима.

**CI-форсинг:** Линтер (`scripts/lint-no-source-grep.cjs`, запускается как `npm run lint:tests`) детектит нарушения. Любой test-файл, вызывающий `readFileSync` на `.cjs` пути в source-директории без аннотации-исключения ниже, упадёт в job `lint-tests` CI.

### Исключение: `allow-test-rule: <reason>`

Некоторые тесты легитимно читают source-файлы. Шесть распознаваемых категорий:

| Reason | Когда использовать |
|--------|---------------------|
| `source-text-is-the-product` | Agent `.md`, workflow `.md`, command `.md` — их текст ЕСТЬ то что рантайм загружает. Тестирование текстового контента тестирует deployed-контракт. |
| `architectural-invariant` | Реализация должна использовать конкретный примитив (например, `Atomics.wait`, atomic-file-writes), который нельзя протестировать наблюдая outputs. |
| `structural-regression-guard` | Конкретный code-pattern должен (или не должен) существовать чтобы предотвратить класс багов (например, regex global-state misuse). Поведенческие тесты не могут различить, какой паттерн использован. |
| `docs-parity` | Reference-док должен оставаться в синке с source-defined константами (например, `CONFIG_DEFAULTS`). Source — канонический список; runtime API для перечисления нет. |
| `integration-test-input` | Source-файл используется как реальный fixture-input для transformation-функции под тестом — файл не инспектится на строки, а передаётся как данные. |
| `structural-implementation-guard` | Точка интерсепции/проводки фичи недостижима end-to-end через `runGsdTools`. Используется временно пока не появится behavioral path. |
| `pending-migration-to-typed-ir` | **Отслеживается для коррекции, не освобождается.** Тест идентифицирован lint'ом как несущий raw-text-matching паттерн, противоречащий правилу выше. Каждый аннотированный файл ДОЛЖЕН цитировать открытый migration issue (например, `// allow-test-rule: pending-migration-to-typed-ir [#NNNN]`) чтобы отслеживание было аудитируемым. Новые тесты не могут использовать эту категорию — они должны рефакторить production-код чтобы выставить typed IR. Аннотация удаляется когда тест исправлен. |

Аннотируй standalone `//` комментарием перед opening block-комментарием файла:

```javascript
// allow-test-rule: architectural-invariant
// state.cjs locking must use Atomics.wait(), not a spin-loop. Behavioral tests
// cannot observe which sleep primitive was chosen — only source inspection can.

/**
 * Regression tests for locking bugs #1909...
 */
```

Аннотация **должна** быть standalone строкой `// allow-test-rule:`, не внутри `/** */` block-комментария — CI-линтер сканит на паттерн `// allow-test-rule:`.

### Запрещено: raw text matching на test-outputs (file content, stdout, stderr)

**Source-grep — это не только `readFileSync` на `.cjs` файле.** Тот же анти-паттерн всплывает везде, где test-паттерн matchится против текста, произведённого system-under-test, независимо от того, пришёл ли этот текст из source-файла, рендеренного shim'а, child-process stdout или free-form `reason`-строки. **Все формы запрещены.**

Следующее — все нарушения одного правила:

```javascript
// BAD — substring match на тексте, написанном code under test
const cmdContent = fs.readFileSync(path.join(tmpDir, 'gsd-sdk.cmd'), 'utf8');
assert.ok(cmdContent.includes(`@node ${jsonQuoted} %*`), '.cmd embeds shim path');

// BAD — regex match на human-readable stdout форматтере child-процесса
const r = cp.spawnSync(SCRIPT, ['--patches-dir', dir]);
assert.match(r.stdout, /Failures: 1/);
assert.match(r.stdout, /not a regular file/);

// BAD — "structured parser", прячущий string-операции за обёрткой-функцией
function parseCmdShim(content) {
  const lines = content.split('\r\n').filter((l) => l.length > 0);
  return { header: lines[0], usesCRLF: content.includes('\r\n') };
}

// BAD — assert.match на free-form `reason`-строке из JSON-репорта
assert.ok(/not a regular file/.test(report.results[0].reason));
```

Каждый из этих проходит на случайных near-match'ах (комментарий с `@node` где-то, stack trace, случайно говорящий `Failures: 1`, опечатанный reason, всё ещё содержащий matchимый substring) и фейлится на безобидном переформатировании (смена `Failures: 1` на `1 failure`, смена CRLF стиля рендеринга, переформулирование error-prose).

#### Правило

> **Тесты ассертят на типизированные структурированные значения. Если код под тестом производит текст, код под тестом обязан также выставить структурированное промежуточное представление (IR), и тест должен ассертить на это IR — никогда на рендеренный текст.**

Конкретно: для любой system-under-test, производящей текстовый output (file renderer, CLI formatter, error-message builder), production-код ОБЯЗАН выставить типизированную альтернативу, которую потребляет тест:

| Output kind | Required структурированная поверхность | На что тест ассертит |
|---|---|---|
| Рендеренный файл (shim, template, сгенерированный код) | Чистая builder-функция, возвращающая IR (`{ invocation, eol, fileNames, render }`) | `triple.invocation.target === expected`, `triple.eol.cmd === '\r\n'` |
| CLI human-formatter output | `--json` режим, эмитящий те же данные структурно | `report.results[0].reason === REASON.FAIL_INSTALLED_NOT_REGULAR_FILE` |
| Error / status / reason | Замороженный enum (`Object.freeze({ FAIL_X: 'fail_x', ... })`) | `assert.equal(result.reason, REASON.FAIL_X)` |
| Наличие файла после записи | `fs.statSync().isFile()`, `.size > 0`, `.mtimeMs` продвигается | Filesystem-факты; никогда не читать file-content обратно |

#### Конкретные примеры из этой репы

`buildWindowsShimTriple(shimSrc)` в `bin/install.js` — канонический IR-паттерн: чистая функция, без I/O, возвращает `{ invocation, eol, fileNames, render }`. `trySelfLinkGsdSdkWindows` её вызывает и пишет `triple.render[kind]()` на диск. Тесты ассертят на `triple.invocation.target`, `triple.eol.cmd`, `Object.keys(triple).sort()` — никогда на рендеренный текст. Filesystem-level тесты ассертят `fs.statSync(target).size === Buffer.byteLength(triple.render.cmd())` чтобы доказать что writer пишет то что производит renderer, **без сравнения контента**.

`scripts/verify-reapply-patches.cjs` выставляет замороженный enum `REASON` и эмитит его через `--json`. Тесты ассертят `report.results[0].reason === REASON.FAIL_USER_LINES_MISSING`. Human formatter существует только для operator console output — тесты не должны зависеть от его prose. Добавление нового reason-code требует обновления enum'а `REASON`, `--json` output и теста, лочащего `Object.keys(REASON).sort()` — три скоординированных изменения, не дающих code-surface дрейфовать от test-surface.

#### Прятать grep за функцией — это всё ещё grep

`parseCmdShim`, `parsePs1Invocation` и т.д., которые внутри делают `content.split(...)`, `lines[1].trim()`, `content.includes(...)` — это всё ещё string manipulation. Тот факт, что entry point выглядит как parser, не меняет что происходит под капотом — тест всё ещё ассертит на lexical shape рендеренного текста. Фикс не «обернуть grep в функцию с типизированно-выглядящим return value». Фикс — **полностью исключить рендеренный текст из test-пути**, выставив IR.

#### Когда нельзя исключить text matching

Есть ровно два случая, где текстовый контент — легитимный объект теста, оба уже покрыты существующей exemption-матрицей:

1. `source-text-is-the-product` — workflow `.md` / agent `.md` / command `.md` файлы, где deployed-текст ЕСТЬ то что рантайм загружает.
2. `docs-parity` — reference-док должен зеркалить source-defined константы, и нет runtime enumeration API.

Для всего остального, если тест тянется к `.includes()` / `.startsWith()` / `assert.match(text, /…/)` — production-коду не хватает типизированной поверхности. **Добавь типизированную поверхность; не обходи её.**

**CI-форсинг:** `scripts/lint-no-source-grep.cjs` расширяется (см. issue-трекер для актуального scope) чтобы флагать `String#includes`/`String#startsWith`/`String#endsWith`/`assert.match` на результатах `readFileSync` и на stdout/stderr `cp.spawnSync` в test-файлах, с тем же механизмом исключения `// allow-test-rule:`.

### Совместимость версий Node.js

**Node 22 — минимальная поддерживаемая версия.** Node 24 — основная CI-цель. Все тесты должны проходить на обоих.

| Версия | Статус |
|--------|--------|
| **Node 22** | Минимально требуется — Active LTS до октября 2026, Maintenance LTS до апреля 2027 |
| **Node 24** | Основная CI-цель — текущий Active LTS, все тесты должны проходить |
| Node 26 | Forward-compatible цель — избегай deprecated API |

Не используй:
- Deprecated API
- API недоступные в Node 22

Безопасно использовать:
- `node:test` — стабильно с Node 18, полнофункциональный в 24
- `describe`/`it`/`test` — все поддерживаются
- `beforeEach`/`afterEach`/`before`/`after` — все поддерживаются
- `t.after()` — per-test cleanup
- `t.plan()` — полностью поддерживается
- Snapshot-тестирование — полностью поддерживается

### Assertion'ы

Используй `node:assert/strict` для строгого равенства по дефолту:

```javascript
const assert = require('node:assert/strict');

assert.strictEqual(actual, expected);      // ===
assert.deepStrictEqual(actual, expected);  // deep ===
assert.ok(value);                          // truthy
assert.throws(() => { ... }, /pattern/);   // throws
assert.rejects(async () => { ... });       // async throws
```

### Запуск тестов

```bash
# Запустить все тесты
npm test

# Запустить один test-файл
node --test tests/core.test.cjs

# Запуск с coverage
npm run test:coverage
```

### Pre-PR Seam Checks (Manifest/Alias Routing)

Если трогал любые command-manifest или сгенерированные alias-файлы — запусти:

```bash
npm run check:alias-drift
```

Это проверяет что сгенерированные alias-артефакты в синке с manifest-source-of-truth.

Опциональный локальный pre-commit hook (Git-native):

```bash
# one-time setup
mkdir -p .githooks
cat > .githooks/pre-commit <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if git diff --cached --name-only | grep -Eq "^sdk/src/query/command-manifest\.|^sdk/src/query/command-aliases\.generated\.ts$|^get-shit-done/bin/lib/command-aliases\.generated\.cjs$|^sdk/scripts/gen-command-aliases\.ts$"; then
  npm run check:alias-drift
fi
EOF
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks
```

Опциональный локальный pre-push hook для блокирования паттерна email автора:

```bash
# установи локально в shell-профиле (пример)
export GSD_BLOCKED_AUTHOR_REGEX='@example-corp\\.com$'

cat > .githooks/pre-push <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

zero_sha='0000000000000000000000000000000000000000'
blocked_regex="${GSD_BLOCKED_AUTHOR_REGEX:-}"
[[ -z "$blocked_regex" ]] && exit 0
violations=()

while read -r local_ref local_sha remote_ref remote_sha; do
  [[ "$local_sha" == "$zero_sha" ]] && continue
  if [[ "$remote_sha" == "$zero_sha" ]]; then
    commits=$(git rev-list "$local_sha" --not --remotes)
  else
    commits=$(git rev-list "$remote_sha..$local_sha")
  fi
  while read -r commit; do
    [[ -z "$commit" ]] && continue
    email=$(git show -s --format='%ae' "$commit" | tr '[:upper:]' '[:lower:]')
    if printf '%s' "$email" | grep -Eq "$blocked_regex"; then
      violations+=("$commit <$email>")
    fi
  done <<< "$commits"
done

if [[ ${#violations[@]} -gt 0 ]]; then
  echo "Push blocked: commit author email matched local blocked regex ($blocked_regex)." >&2
  printf '  - %s\n' "${violations[@]}" >&2
  exit 1
fi
EOF
chmod +x .githooks/pre-push
```

### CI Test Quality Checks

Следующие проверки прогоняются на каждом PR в дополнение к test-набору:

| Job | Что проверяет | Как пройти |
|-----|---------------|------------|
| `lint-tests` | Нет source-grep тестов (см. выше) | Замени на `runGsdTools()` поведенческие тесты, или добавь `// allow-test-rule: <reason>` |

Запуск локально до push'а: `npm run lint:tests`

### Test-требования по типу контрибьюции

### Architecture-Aware Testing Requirements

Когда работа трогает архитектуру, routing, политику, registry-assembly или семантику команд:
- Пиши тесты против **интерфейсов** модулей и поведения seam, а не тривии реализации.
- Предпочитай invariant/contract тесты, защищающие ADR-backed поведение и терминологию `CONTEXT.md`.
- Убедись что тесты валидируют каноническое поведение через определённый seam (например: structured result-контракты, каноническую command-метадату и adapter-паритет), не source-text связанность.
- Если ADR определяют ожидаемое поведение, тесты должны ассертить эти ожидания напрямую.

Требуемые тесты различаются в зависимости от типа контрибьюции:

**Bug Fix:** Regression-тест обязателен. Пиши тест первым — он должен демонстрировать оригинальное падение до применения фикса, потом проходить после. PR, чинящий баг без regression-теста, попросят добавить его. «Тесты проходят» не доказывает корректность; это доказывает что баг отсутствует в существующих тестах.

**Enhancement:** Тесты, покрывающие enhanced-поведение, обязательны. Обнови любые существующие тесты, тестирующие область, которую ты менял. Не оставляй тесты, которые проходят, но больше не описывают поведение точно.

**Feature:** Тесты обязательны для primary success path и минимум одного failure-сценария. Оставление пробелов в test-покрытии новой фичи — причина отказа.

**Behavior Change:** Если твоё изменение модифицирует существующее поведение — существующие тесты, покрывающие это поведение, должны быть обновлены или заменены. Оставление passing-but-incorrect тестов в suite неприемлемо — тест, проходящий, но ассертящий старое (теперь неправильное) поведение, делает suite менее полезным чем отсутствие теста.

### Стандарты ревьюера

Ревьюеры не полагаются исключительно на CI для верификации корректности. До одобрения PR ревьюеры:

- Билдят локально (`npm run build` если применимо)
- Запускают полный test-набор локально (`npm test`)
- Подтверждают что regression-тесты существуют для багфиксов и что они бы упали без фикса
- Валидируют что реализация совпадает с тем что описано в связанном issue — зелёный CI на неправильной реализации — не сигнал к approve

**«Тесты проходят в CI» недостаточно для мерджа.** Реализация должна корректно решать проблему, описанную в связанном issue.

## Стиль кода

- **CommonJS** (`.cjs`) — проект использует `require()`, не ESM `import`
- **Никаких внешних зависимостей в core** — `gsd-tools.cjs` и все lib-файлы используют только Node.js built-ins
- **Conventional commits** — `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `ci:`

## Структура файлов

```
bin/install.js          — Установщик (multi-runtime)
get-shit-done/
  bin/lib/              — Core library модули (.cjs)
  workflows/            — Определения workflow (.md)
                          Большие workflow разбиты по progressive-disclosure
                          паттерну: workflows/<name>/modes/*.md +
                          workflows/<name>/templates/*. Родитель диспатчит
                          к mode-файлам. См. workflows/discuss-phase/
                          как канонический пример (#2551). Новые modes для
                          discuss-phase ложатся в
                          workflows/discuss-phase/modes/<mode>.md.
                          Per-file бюджеты форсятся
                          tests/workflow-size-budget.test.cjs.
  references/           — Reference-документация (.md)
  templates/            — Шаблоны файлов
agents/                 — Определения агентов (.md) — КАНОНИЧЕСКИЙ ИСТОЧНИК
commands/gsd/           — Определения slash-команд (.md)
tests/                  — Test-файлы (.test.cjs)
  helpers.cjs           — Общие test-утилиты
docs/                   — Пользовательская документация
```

### Source of truth для агентов

Только `agents/` в корне репы отслеживается git'ом. Следующие директории могут существовать на машине разработчика с установленным GSD и **не должны редактироваться** — они install-sync outputs и будут перезаписаны:

| Путь | Gitignored | Что это |
|------|-----------|---------|
| `.claude/agents/` | Да (`.gitignore:9`) | Локальный Claude Code runtime-синк |
| `.cursor/agents/` | Да (`.gitignore:12`) | Локальный Cursor IDE-бандл |
| `.github/agents/gsd-*` | Да (`.gitignore:37`) | Локальный CI-surface бандл |

Если обнаружишь что `.claude/agents/` дрейфует от `agents/` (например, после смены ветки) — перезапусти `bin/install.js` чтобы re-синкнуть из канонического источника. Всегда правь `agents/` — никогда производные директории.

## Безопасность

- **Валидация путей** — используй `validatePath()` из `security.cjs` для любых пользовательских путей
- **Никаких shell injection** — используй `execFileSync` (array args) вместо `execSync` (string interpolation)
- **Никаких `${{ }}` в GitHub Actions `run:` блоках** — связывай через `env:` маппинги
