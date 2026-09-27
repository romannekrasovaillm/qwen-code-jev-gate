# Отчёт-эвиденс: Jev-стадия (stage-0) на базе upstream/main (гейт A4)

- Проект: `qwen-code`, ветка `arch/jev-gate`, worktree `jev-gate`.
- **База дельты: `224c974a21`** (перенос пакета и артефактов из `arch/jev-stage1`) поверх
  `upstream/main` = `61e7b92b09` (ADR-005). Точка сравнения дельты — `224c974a21`.
- Спека: `docs/specs/jev-gate-spec.md` Rev 2 (§1–§7, §12 «Решения архитектора по расхождениям»).
  Инварианты: `ARCHITECTURE-SPINE.md` AD-1 … AD-11. Решения: ADR-001…ADR-005.
- **Хеш замороженного спека вопросов v1: `a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d`**
  (не изменился: `questions.ts` и `pilot/questions.spec.json` в этой задаче не правились).
- Прогон, породивший эту задачу: `hr-20260926-200537-00` (`status=partial`, ADR-005).
- **Ревизия пакета 2** (2026-09-27): проводка `settings.jev` из `settings.json` в ядро (спека
  §12-бис C8) и тест совместимости хуков со стадией (§10.3) — раздел «Ревизия пакета 2» в конце
  документа, §§9–12.

## 1. Что реализовано

| ID      | Компонент                  | Файлы                                                                                       | Состояние   |
| ------- | -------------------------- | ------------------------------------------------------------------------------------------- | ----------- |
| CMP-001 | Врезка стадии-0            | `permissions/classifier.ts` (врезка в настоящий `classifyAction()`), `jev/stage.ts`         | реализовано |
| CMP-001 | Контекст вызова стадии     | `jev/call-context.ts` (путь/класс каталога из `toolParams`+`config`)                        | реализовано |
| CMP-002 | Адаптер бэкенда            | `jev/backends/{index,dryrun,systemone}.ts` (было `typesafe.ts`)                             | реализовано |
| CMP-003 | Гейт-логика                | `jev/gate.ts`                                                                               | реализовано |
| CMP-004 | Сборщик `state` и редакция | `jev/state.ts`                                                                              | реализовано |
| CMP-005 | Журнал решений             | `jev/decision-log.ts` (было `decisionLog.ts`)                                               | реализовано |
| CMP-006 | Настройки                  | `jev/settings.ts`, блок `jev` в `config/config.ts`                                          | реализовано |
| INT-001 | Носитель решения           | локальный System One-совместимый сервер, loopback (ADR-004); внешний вендор выведен (AD-10) | конфигом    |

Сеть в тестах не используется ни в одном сценарии: транспорты фиктивные, `fetch` подменён
шпионом там, где его можно случайно задеть.

### 1.1 Что изменилось относительно `arch/jev-stage1`

1. **Seam снят.** В прежней базе `classifyAction(input, downstream)` был собран харнессом как
   временная точка врезки (ADR-005). Теперь врезка — в настоящий `classifyAction()`: вызов
   `runJevStage` стоит после построения входа и до `sideQuery` stage-1; при `decided=true`
   возвращается `ClassifierResult` с `stage: 'jev'` (ADR-002 §3).
2. **Стадия достраивает контекст вызова сама.** Точка врезки не получает `PermissionCheckContext`,
   поэтому `jev/call-context.ts` выводит путь/класс каталога теми же правилами, что и остальной
   AUTO-контур (`buildPermissionCheckContext`, `WorkspaceContext.isPathWithinWorkspace`), и
   никогда не бросает: неизвестный путь даёт `path_class: 'none'`, а не выдуманный класс (AD-4).
3. **Бэкенд `typesafe-api` → `system-one-http`** (спека §12 C3, ADR-004 Rev 2): файл
   `backends/typesafe.ts` переименован в `backends/systemone.ts`, внешний вендорский адрес и
   ключ из контура убраны полностью. Ключей доступа нет вовсе — локальный носитель их не
   требует (спека §12 C2/C3, AD-8, AD-10).
4. **Endpoint и периметр в настройках** (спека §12 C2): `JevSettings.endpoint`
   (дефолт `http://127.0.0.1:11435/v1/systemone`), `JevSettings.internalHosts`
   (дефолт `['127.0.0.1','::1','localhost']`), `timeoutMs` 3000 вместо 1200 (спека §12 C4).
5. **Детерминированные слои старше стадии** остались инвариантом, но теперь проверяются на
   реальном AUTO-пути: `evaluateAutoMode` завершает вызов в L5.1/L5.2/L5.2.5/L5.2.6 до
   классификатора, поэтому врезка передаёт `determinsticVerdict: 'allow-eligible'` — это
   следствие порядка слоёв, а не допущение (AD-2).
6. **Ленивый хеш спека.** `JEV_QUESTION_SPEC_SHA256` (значение, считавшееся при загрузке
   модуля) заменён на `jevQuestionSpecSha256()`. Причина не косметическая: `questions.ts`
   теперь достижим из `permissions/classifier.ts`, и вычисление хеша на уровне модуля ломало
   upstream-тест `src/tools/shell.test.ts` (`vi.mock('crypto')` → `createHash` undefined).
   Хеш-значение не изменилось.

## 2. Команды и вывод

### 2.1 Установка зависимостей

```console
$ pnpm install --frozen-lockfile --prefer-offline
… packages/core prebuild / build …
[ELIFECYCLE] Command failed with exit code 1.   # шаг `prepare` (полная сборка), гонялся одновременно с правками
```

Зависимости встали; падение — у lifecycle-скрипта `prepare`, запускающего полную сборку в тот
момент, когда дерево было в середине правок. Отдельная сборка (2.5) проходит.

### 2.2 Тесты Jev-стадии (T1–T15)

```console
$ npx vitest run src/jev
 ✓ src/jev/__tests__/questions.test.ts (7 tests)
 ✓ src/jev/__tests__/gate.test.ts (17 tests)
 ✓ src/jev/__tests__/settings.test.ts (23 tests)
 ✓ src/jev/__tests__/decision-log.test.ts (10 tests)
 ✓ src/jev/__tests__/backends.test.ts (16 tests)
 ✓ src/jev/__tests__/state.test.ts (12 tests)
 ✓ src/jev/__tests__/classifier-stage0-translation.test.ts (6 tests)
 ✓ src/jev/__tests__/stage.test.ts (18 tests)
 ✓ src/jev/__tests__/classifier-stage0.test.ts (9 tests)

 Test Files  9 passed (9)
      Tests  118 passed (118)
```

Что именно доказывается на новой базе:

| #   | Тест                                                                                                                                   | Где                                     | Инвариант  |
| --- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- | ---------- |
| T1  | `enabled=false` → вердикт совпадает с конфигом без блока `jev`; журнал не пишется, `fetch` не тронут                                   | `classifier-stage0.test.ts`             | AD-3       |
| T1  | то же на реальном `evaluateAutoMode` (`via`, `stage`, `shouldBlock`, `reason`, `unavailable`)                                          | `classifier-stage0.test.ts`             | AD-3       |
| T2  | `git reset --hard` → `blocked:destructive-command` до стадии; `ask`-правило и внешняя запись → `fallback`; сеть не тронута             | `classifier-stage0.test.ts`             | AD-2       |
| T3  | `decided=true` → `ClassifierResult.stage='jev'`, stage-1 не вызван; `decided=false` → каскад                                           | `classifier-stage0-translation.test.ts` | ADR-002 §3 |
| T4  | таймаут бюджета → `unavailable='timeout'`, `decided=false`, каскад работает                                                            | `stage.test.ts`                         | AD-3       |
| T5  | HTTP 5xx / битый JSON / вероятности вне [0,1] или не к 1 → `unavailable`, `decided=false`                                              | `backends.test.ts`, `stage.test.ts`     | AD-3       |
| T6  | 3 отказа подряд → circuit breaker разомкнут, 4-й вызов не идёт в бэкенд                                                                | `stage.test.ts`                         | AD-3       |
| T7  | `read_only` ∧ `is_destructive=0.95` → `no-decision` + `read_only_vs_destructive`                                                       | `gate.test.ts`                          | AD-5       |
| T8  | `touches_secrets=0.9` ∧ `exfiltrates_data=0.9` → `block` вне зависимости от класса                                                     | `gate.test.ts`                          | AD-5       |
| T9  | секреты в аргументах вырезаны до запроса; в журнале только счётчик                                                                     | `state.test.ts`, демо                   | AD-4, AD-8 |
| T10 | запрещённые ключи не проходят ни при каких случайных параметрах                                                                        | `state.test.ts`                         | AD-4       |
| T11 | запись журнала валидируется схемой, аргументов и транскрипта в ней нет                                                                 | `decision-log.test.ts`                  | AD-7       |
| T12 | `reason` — из фиксированных шаблонов, текст бэкенда в него не попадает                                                                 | `gate.test.ts`                          | AD-6       |
| T13 | `enforce` отклонён; выведенный вендорский бэкенд отклонён; `dry-run` в `block-only` запрещён                                           | `settings.test.ts`                      | AD-9       |
| T14 | дефолт — выключено/тень/локальный носитель; значения ключа вендора в репозитории нет                                                   | `settings.test.ts`                      | AD-1, AD-8 |
| T15 | внешний хост (в т.ч. вендорский) отклоняется валидацией; `127.0.0.1.<чужой домен>` не loopback; внешний адрес не доходит до транспорта | `settings.test.ts`, `backends.test.ts`  | AD-10      |

### 2.3 Существующие upstream-тесты

```console
$ npx vitest run src/permissions
 Test Files  13 passed (13)
      Tests  1070 passed (1070)      # classifier 24, autoMode 98, permission-manager 557, …
```

Файлы upstream-тестов (`classifier.test.ts`, `autoMode.test.ts`, `permission-manager.test.ts` и
остальные) **не изменялись** — ни один тест не переписан под дельту:

```console
$ git status --short -- packages/core/src/permissions packages/core/src/config
 M packages/core/src/config/config.ts
 M packages/core/src/permissions/autoMode.ts
 M packages/core/src/permissions/classifier.ts
```

### 2.4 Полный прогон `packages/core`

```console
$ npx vitest run            # рабочий локаль хоста (ru)
 Test Files  6 failed | 818 passed | 1 skipped (825)
      Tests  14 failed | 31495 passed | 10 skipped (31519)
```

Падения — предсуществующие и/или локаль-зависимые; ни одно не в `permissions/**`, `jev/**`,
`autoMode` или `classifier`. Классификация:

| Группа                                                       | Кол-во | Доказательство предсуществования                                              |
| ------------------------------------------------------------ | ------ | ----------------------------------------------------------------------------- |
| `config.test.ts` — форматирование чисел локали               | 3      | проходят при `LANG=C` (`Intl.NumberFormat` даёт `1 000` вместо `1,000`)       |
| `agents/runtime/agent-statistics.test.ts` — то же            | 1      | известный дефект upstream (зафиксирован ещё прогоном `hr-20260926-200537-00`) |
| `extension/extensionManager.test.ts` — git-clone-мок/env     | 7      | воспроизведены на чистом baseline `224c974a21` (дельты нет)                   |
| `utils/git-worktrees.test.ts` — соседние worktree в дереве   | 1      | воспроизведён на baseline                                                     |
| `core/anthropicContentGenerator` — env агентной сессии       | 1      | воспроизведён на baseline                                                     |
| `sandbox/file-worker-protocol.test.ts` — флейк под нагрузкой | 1      | 3/3 прохода в изоляции; файл не импортирует дельту                            |

Проверка «на чистом baseline» выполнялась так: рабочий коммит зафиксирован, затем
`git checkout 224c974a21 -- packages/core/src` (врезки и всего `jev/**` в графе нет) и повторный
прогон тех же файлов; получены **те же 9 падений**, после чего дерево восстановлено из коммита.

Отдельно: полный прогон **выявил и позволил починить** реальную поломку, внесённую дельтой —
`src/tools/shell.test.ts` падал на загрузке (`createHash` undefined) из-за вычисления хеша спека
на уровне модуля; после перехода на ленивый `jevQuestionSpecSha256()` файл зелёный (360 тестов).

### 2.5 Типы, линт, сборка

```console
$ npx tsc --noEmit -p packages/core
EXIT=0

$ npx eslint packages/core/src/jev packages/core/src/permissions/classifier.ts \
      packages/core/src/permissions/autoMode.ts packages/core/src/config/config.ts
EXIT=0

$ rm -rf packages/core/dist && corepack pnpm --filter @qwen-code/qwen-code-core run build
Successfully copied files.

$ corepack pnpm run build            # сборка всего репозитория, включая CLI и esbuild-бандлы
… все пакеты: Done …
EXIT=0
```

Сборка — это и проверка критерия §8.2: значение `stage: 'jev'` принято типами во всех местах
использования (`ClassifierResult` и `AutoModeDecision`), бандл собирается.

Линт попутно потребовал kebab-case для имён файлов пакета (`check-file/filename-naming-convention`):
`callContext.ts` → `call-context.ts`, `decisionLog.ts` → `decision-log.ts` (последнее —
унаследованное нарушение из переноса). Спека §1 упоминает старые имена — расхождение
зафиксировано ниже в §7.

### 2.6 Fitness-контроль

```console
$ arch-ml control check . --no-color --ascii
Правил: 27, нарушений: 0 (error: 0, warn: 0)
Итог: PASS
```

Два правила ruleset потребовали правки — обе вне кода ядра (сам ruleset — артефакт проекта, не
upstream-файл):

1. `key_read_from_env_only` требовал чтения `TYPESAFE_API_KEY` из окружения в
   `backends/typesafe.ts`. Файла больше нет (ADR-004 Rev 2: ключа вендора в контуре нет), поэтому
   правило заменено на `no_vendor_key_in_core` — прямой запрет `TYPESAFE_API_KEY|process.env`
   в коде пакета (тесты исключены: имя переменной там — часть сканера утечек, а за значение
   отвечает `no_hardcoded_secret_values`).
2. `upstream_delta_scope` расширен на `permissions/autoMode.ts` (см. §7.1) — третья точка
   касания, вызванная аддитивным расширением union `stage`.

Правило `upstream_delta_scope` проверено на невакуозность: при подстановке постороннего пути
(`packages/core/src/tools/foo.ts`) оно даёт 1 нарушение.

## 3. Демонстрационный прогон (shadow, `dry-run`, без сети)

```console
$ npx tsx pilot/jev-shadow-demo.ts
Jev-стадия: демонстрационный прогон (shadow, backend=dry-run)
spec_hash = a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d
дефолты   = enabled:false mode:shadow backend:system-one-http timeoutMs:3000
endpoint  = http://127.0.0.1:11435/v1/systemone (AD-10: только loopback/периметр)
периметр  = internalHosts: ["127.0.0.1","::1","localhost"]

Решения не изменились: 10/10 (AD-3, T1)

Журнал решений: 10 записей (AD-7)
  #   инструмент           класс        backend    conf   вердикт          via            вырезано t
  1   read_file            read_only    dry-run    0.85   no-decision      jev-fallback   0        2ms
  2   write_file           write_local  dry-run    0.80   no-decision      jev-fallback   0        1ms
  3   run_shell_command    network      dry-run    0.80   no-decision      jev-fallback   0        0ms
  4   run_shell_command    destructive  dry-run    0.85   block            jev-fallback   0        0ms
  5   run_shell_command    destructive  dry-run    0.85   block            jev-fallback   0        0ms
  6   run_shell_command    read_only    dry-run    0.85   no-decision      jev-fallback   1        1ms
  7   run_shell_command    network      dry-run    0.80   no-decision      jev-fallback   1        0ms
  8   run_shell_command    read_only    dry-run    0.85   no-decision      jev-fallback   0        2ms
  9   run_shell_command    read_only    dry-run    0.85   no-decision      jev-fallback   0        0ms
  10  write_file           write_local  dry-run    0.80   no-decision      jev-fallback   0        0ms

Итог: вердиктов block = 2 (в shadow не применяются, AD-9); вырезано секретов = 2 (AD-8); отказов = 0 (AD-3)
```

Фрагмент `state`, уходящего носителю (сценарий 6 — редакция секретов до запроса, AD-4/AD-8):

```json
{"policy":"Agents may read freely. Writes are reversible or confirmed. Spending, publishing, deletion, credential access and external transmission require confirmation or a human.","proposed_call":{"tool":"run_shell_command","command":"cat .env && export GITHUB_TOKEN=[REDACTED]","path_class":"workspace","cwd_class":"workspace"},"deterministic":{"flags":{}}}
вырезано мест: 1; подстрок исходного секрета: нет
```

Фрагмент журнала (записи 1–3, без содержимого аргументов и транскрипта, AD-7):

```json
{"backend":"dry-run","class":"read_only","confidence":0.85,"durationMs":2,"probabilities":{"destructive":0.03,"network":0.03,"other":0.03,"read_only":0.85,"spend":0.03,"write_local":0.03},"redactions":0,"specHash":"a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d","tool":"read_file","ts":"2026-09-26T18:14:35.080Z","verdict":"no-decision","via":"jev-fallback"}
{"backend":"dry-run","class":"write_local","confidence":0.8,"durationMs":1,"probabilities":{"destructive":0.04,"network":0.04,"other":0.04,"read_only":0.04,"spend":0.04,"write_local":0.8},"redactions":0,"specHash":"a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d","tool":"write_file","ts":"2026-09-26T18:14:35.082Z","verdict":"no-decision","via":"jev-fallback"}
{"backend":"dry-run","class":"network","confidence":0.8,"durationMs":0,"probabilities":{"destructive":0.04,"network":0.04,"other":0.04,"read_only":0.04,"spend":0.04,"write_local":0.04},"redactions":0,"specHash":"a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d","tool":"run_shell_command","ts":"2026-09-26T18:14:35.083Z","verdict":"no-decision","via":"jev-fallback"}
```

Проверка периметра в самом прогоне:

```console
Проверка периметра носителя решения (AD-10, ADR-004):
  внешний endpoint отклонён: settings.jev.endpoint указывает на внешний хост "collector.example.com":
  Jev-стадия адресует только loopback (127.0.0.1/::1) или внутренний хост периметра, объявленный в settings.jev.internalHosts (AD-10, ADR-004)
```

### 3.1 Опциональный режим `--live` (локальный носитель)

```console
$ npx tsx pilot/jev-shadow-demo.ts --live
Локальный носитель решения (--live): один вызов system-one-http по loopback
  штатная деградация: unavailable=transport → эскалация (AD-3)
```

На этой машине локальный сервер не поднят (носитель — Ollaya на GB10, `127.0.0.1:11435` здесь
не слушает), поэтому зафиксирована именно **штатная деградация**: недоступность носителя не
превращается в разрешение и не роняет прогон — управление уходит существующему пути (AD-3, AD-10).
Боевой ответ носителя этим прогоном **не проверялся** (см. §6).

## 4. Дельта против baseline

```console
$ git diff --stat 224c974a21 HEAD -- packages/core/src
 packages/core/src/config/config.ts                 |  34 +++
 packages/core/src/jev/__tests__/backends.test.ts   | 145 ++++++---
 .../core/src/jev/__tests__/classifier-seam.test.ts | 228 --------------
 .../classifier-stage0-translation.test.ts          | 220 ++++++++++++++
 .../src/jev/__tests__/classifier-stage0.test.ts    | 326 +++++++++++++++++++++
 .../{decisionLog.test.ts => decision-log.test.ts}  |   6 +-
 packages/core/src/jev/__tests__/questions.test.ts  |   6 +-
 packages/core/src/jev/__tests__/settings.test.ts   | 137 +++++++--
 packages/core/src/jev/__tests__/stage.test.ts      |  36 +--
 packages/core/src/jev/__tests__/testUtils.ts       |  29 +-
 packages/core/src/jev/backends/index.ts            |  43 ++-
 .../src/jev/backends/{typesafe.ts => systemone.ts} | 149 ++++------
 packages/core/src/jev/call-context.ts              | 109 +++++++
 packages/core/src/jev/{decisionLog.ts => decision-log.ts} | 4 +-
 packages/core/src/jev/questions.ts                 |  21 +-
 packages/core/src/jev/settings.ts                  | 149 ++++++---
 packages/core/src/jev/stage.ts                     |  52 +++-
 packages/core/src/jev/types.ts                     |  42 ++-
 packages/core/src/permissions/autoMode.ts          |   7 +-
 packages/core/src/permissions/classifier.ts        |  51 +++-
 20 files changed, 1267 insertions(+), 527 deletions(-)
```

Вне `packages/core/src/jev/**` изменены три файла:

| Файл                                          | Что именно                                                                                                      |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `packages/core/src/permissions/classifier.ts` | врезка `runJevStage` перед stage-1, аддитивное расширение union `stage` значением `'jev'`, ссылка на ADR-002 §1 |
| `packages/core/src/config/config.ts`          | опциональный блок `params.jev`, приватное поле `jevSettings`, геттер `getJevSettings()`                         |
| `packages/core/src/permissions/autoMode.ts`   | **один union**: `stage: 'jev' \| 'fast' \| 'thinking'` в `AutoModeDecision` (см. §7.1)                          |

Проверяется машинно правилом `upstream_delta_scope` (PASS).

Прочие изменённые артефакты (вне `packages/core/src`): `pilot/jev-shadow-demo.ts` (режим `--live`,
новые дефолты), `CONSTRAINTS.yaml`, `docs/specs/jev-gate-evidence.md` (этот файл),
`docs/diagrams/jev-gate.architecture.json`. Диаграмма и `CONSTRAINTS.yaml` прошли через
prettier из pre-commit-хука репозитория (канонизация кавычек в YAML и раскрытие JSON-объектов),
поэтому их диффы длиннее смысловой правки; смысловая часть — замена внешнего вендора на
локальный носитель в диаграмме (ADR-004 Rev 2) и две правки правил (§2.6).

## 5. План отката (проверен на этой базе)

- **L0 (мгновенный):** `settings.jev.enabled=false` — поведение AUTO возвращается к upstream;
  доказано тестом T1 (в том числе на `evaluateAutoMode`).
- **L1 (сессионный):** circuit breaker выключает стадию после серии отказов (T6).
- **L2 (кодовый):** `git revert` коммита врезки в `arch/jev-gate`; пакет `packages/core/src/jev/**`
  изолирован и удаляется целиком.
- **L3 (полный):** drop ветки `arch/jev-gate`; база `224c974a21` / `upstream/main` и ветка-архив
  `arch/jev-stage1` не тронуты.
- Сигналы к откату: падение upstream-тестов классификатора/autoMode (сейчас 1070/1070 зелёные),
  рост ложных блоков, `unavailable` > 20 % (пилот-01: носитель zero-shot даёт `cascade/reject`).

## 6. Честные границы этого прогона

1. **Боевой носитель не проверялся.** Локального сервера на этой машине нет; `--live` показал
   `unavailable=transport`. Все 118 тестов — на фиктивных транспортах. Числа качества
   (agreement, ECE, латентность) не получены и не заявляются.
2. **Пороги §5 спеки не калиброваны** — стартовые консервативные, права на `allow` не дают (AD-9).
3. **`settings.jev` из `settings.json` в ядро не доходит** (см. §7.2): блок объявлен в
   `ConfigParameters`, но CLI-слой его не передаёт — включение стадии сегодня возможно только
   программно (тесты, пилот). Это следствие ограничения AD-1 на два файла, а не забывчивость.
4. **Оговорка §10.3 спеки (порядок hook-решений `permissionDecision` и Jev-стадии) не снята**:
   тест совместимости хуков и стадии не написан — он обязателен до включения `block-only` на
   реальных сессиях.
5. **Русскоязычная страта не измерена** (пилот-01 дал 7 примеров, `cascade/reject` для zero-shot).

## 7. Расхождения с решениями, зафиксированными ранее

### 7.1 Третья точка касания upstream: `permissions/autoMode.ts`

AD-1 и постановка задачи называют **два** изменяемых upstream-файла. Критерий приёмки §8.2
требует, чтобы `stage: 'jev'` был «принят типами во всех местах использования», а ADR-002 §3 —
аддитивного расширения union. Union `stage` объявлен в базе **дважды**: в `ClassifierResult`
и в `AutoModeDecision` (`autoMode.ts:507`), и `evaluateAutoMode` присваивает `result.stage`
напрямую. Поэтому без правки `autoMode.ts` `tsc` падает — два требования не выполнимы
одновременно.

Сделана **минимальная** правка, ровно один union без логики:

```ts
-      stage: 'fast' | 'thinking';
+      stage: 'jev' | 'fast' | 'thinking';
```

Правило `upstream_delta_scope` расширено на этот файл с комментарием-обоснованием. Альтернативы:
(а) не расширять union и вернуть `stage: 'fast'` — прямое нарушение ADR-002 §3 и T3;
(б) расширить тип до `string` — теряется проверка значений. Оба отвергнуты; решение — за
архитектором (см. `open_questions`).

### 7.2 Проброс `settings.jev` из `settings.json`

Спека §6 показывает блок `settings.json` с `jev`, но цепочка `settings.json → ConfigParameters`
живёт в `packages/cli/src/config/config.ts` (и `settingsSchema.ts`), которые AD-1 запрещает
трогать. Блок объявлен в ядре (`ConfigParameters.jev` + резолвер + геттер), дефолты — выключено.
Включение стадии без правки CLI возможно только для кода, конструирующего `Config` напрямую.

> **Снято пакетом 2** (спека §12-бис C8, ADR-001 Rev 2): дельта расширена на эти два CLI-файла,
> проводка сделана, дефолт остался выключенным. См. §9 ниже.

### 7.3 Имена файлов пакета

Спека §1 указывает `jev/decisionLog.ts` и `backends/typesafe.ts`. Фактически:
`jev/decision-log.ts` (kebab-case — требование eslint-правила `check-file/filename-naming-convention`,
унаследованное нарушение из переноса) и `backends/systemone.ts` (спека §12 C3). Спека не
переписывалась: это её расхождение с кодом, которое стоит поправить в следующей ревизии.

### 7.4 `host`-проверка периметра усилена

Проверка loopback шла через `startsWith('127.')`, из-за чего `127.0.0.1.evil.example` считался
локальным адресом. Теперь хост разбирается как `URL.hostname` и 127/8 принимается только целиком
как IPv4-литерал; пустой хост — fail-closed ошибка. Покрыто тестами T15.

## 8. Артефакты

- Код дельты: `packages/core/src/jev/**` (CMP-001…CMP-006), врезка — `permissions/classifier.ts`.
- Проводка настроек (пакет 2): `packages/cli/src/config/settingsSchema.ts`,
  `packages/cli/src/config/config.ts`; производный `packages/vscode-ide-companion/schemas/settings.schema.json`.
- Тесты: `packages/core/src/jev/__tests__/**` (11 файлов, 143 теста на момент ревизии пакета 2).
- Пилотный прогон: `pilot/jev-shadow-demo.ts` (`--live` для локального носителя).
- Спека и решения: `docs/specs/jev-gate-spec.md`, `docs/adr/ADR-001…005`, `ARCHITECTURE-SPINE.md`.
- Ruleset: `CONSTRAINTS.yaml` (= `.arch-handoff/CONSTRAINTS.yaml`, различия только в кавычках
  после prettier), 28 правил, PASS.

---

# Ревизия пакета 2 (C8 + §10.3/C12)

Раздел добавлен работой пакета 2 поверх Rev 1: проводка `settings.jev` из `settings.json` в ядро
(спека §12-бис C8, ADR-001 Rev 2) и тест совместимости решений хуков с Jev-стадией (оговорка §10.3,
пункт 2 списка «остаётся открытым» и §13.2).

- База дельты не менялась: `224c974a21`; предыдущий коммит ветки — `f7ae81365c`.
- Хеш замороженного спека вопросов v1 не изменился:
  `a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d`.

## 9. Проводка настроек (C8)

### 9.1 Что изменено

| Файл                                                         | Правка                                                                                                                                                                                                                                       |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `packages/cli/src/config/settingsSchema.ts`                  | Объявление ключа `jev`: 8 полей (`enabled`, `mode`, `backend`, `endpoint`, `internalHosts`, `timeoutMs`, `dailyBudgetUsd`, `decisionLogPath`), `category: 'Security'`, `showInDialog: false`. Логики нет.                                    |
| `packages/cli/src/config/config.ts`                          | Одна строка в объекте `ConfigParameters`: `jev: settings.jev` (+ комментарий со ссылкой на ADR-002 §1).                                                                                                                                      |
| `packages/core/src/jev/settings.ts`                          | Закрытая форма блока: `JEV_SETTINGS_KEYS`, `assertJevSettingsShape` — не-объект и неизвестный ключ внутри `jev` дают `JevSettingsError` на загрузке настроек.                                                                                |
| `packages/core/src/jev/__tests__/settings.test.ts`           | Тесты формы блока (частичный блок, неизвестный ключ, не-объект); в сканер секретов добавлены два CLI-файла дельты.                                                                                                                           |
| `packages/core/src/jev/__tests__/cli-wiring.test.ts`         | **Новый.** C8-тесты: значение видно в `Config`, отсутствие ключа = выключено, `enforce`/внешний endpoint/неизвестный ключ роняют загрузку; структурная проверка объявления и проброса в CLI.                                                 |
| `packages/vscode-ide-companion/schemas/settings.schema.json` | **Производный артефакт.** Генератор `scripts/generate-settings-schema.ts` (запускается в `npm run build` и проверяется CI на свежесть) добавил 42 строки описания блока. Механический результат объявления в схеме, не семантическая правка. |

Логика стадии в CLI не появилась: правки upstream-файлов — объявление ключа и передача значения.
Проверяется структурным тестом (`cli-wiring.test.ts`, «в CLI нет логики стадии») и правилом дельты.

### 9.2 Поведение

- **Нет ключа `jev`** → `Config.getJevSettings()` даёт `enabled: false`, `mode: 'shadow'`; решения
  AUTO идентичны upstream (T1 прежней ревизии остаётся в силе).
- **Частичный блок** (`{"jev": {"enabled": true}}`) → остальные поля добираются дефолтами ядра:
  `mode: 'shadow'`, `backend: 'system-one-http'`, `endpoint` — loopback, `timeoutMs: 3000`,
  `internalHosts` — объявленный минимум.
- **Неизвестное поле внутри `jev`** (`jev.modee`) → `JevSettingsError` с перечислением допустимых
  ключей. Причина: загрузчик настроек проверяет только ключи верхнего уровня и о неизвестных
  **вложенных** пишет лишь в debug-лог, то есть опечатка прошла бы молча и стадия пошла бы с
  дефолтом вместо намерения оператора. Тот же приём, что у соседнего блока `omni`
  (`assertKnownOmniSettingKeys`), но проверка живёт в ядре (CMP-006), а не в CLI.
- **`mode: 'enforce'`** → ошибка на загрузке настроек (AD-9, ADR-003 §7), тихого отката к
  `shadow` нет.
- **Внешний endpoint** (не loopback и не из `internalHosts`) → ошибка конфигурации с явным
  сообщением; отката на другой бэкенд или адрес нет (AD-10, ADR-003 §2, ADR-004).
- **Хост из `internalHosts`** → принимается (проверяется позитивным тестом, чтобы правило не
  выродилось в «отвергаем всё»).

Дефолт репозитория остаётся выключенным: объявление ключа в схеме не включает стадию.

## 10. Порядок применения хуков и стадии (§10.3, AD-2)

### 10.1 Найденный фактический порядок

Чтение кода планировщика (`packages/core/src/core/coreToolScheduler.ts`) и ACP-пути
(`packages/cli/src/acp-integration/session/Session.ts`):

```
детерминированные слои AUTO (L5.1 accept-edits, L5.2 allowlist,
  L5.2.5 destructive-commands, ask-правила, L5.2.6 external write)
  → Jev-стадия (stage-0 внутри classifyAction)
    → L4-подтверждение; перед диалогом — PermissionRequest-хук
      → исполнение; в его начале — PreToolUse-хук (allow | deny | ask)
```

То есть **хуки применяются после стадии**, а не до неё. Формулировка задачи предполагала порядок
«детерминированные слои → хуки → стадия»; фактический порядок — «детерминированные слои → стадия →
хуки». Расхождение зафиксировано здесь, в коде не правилось (условие задачи), стадия оставлена в
`shadow` (дефолт не менялся) и вынесено архитектору (см. `open_questions`).

Уровни, зафиксированные тестами (`hook-compat.test.ts`):

| Требование                                          | Как держится                                                                                                                                                                       | Тест                                                                                 |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| (а) `deny` хука не ослабляется стадией (AD-2)       | Стадия отрабатывает раньше и не имеет исхода «разрешить»: `decided=true` только при `block`. Хук-`deny` — терминальный отказ исполнения.                                           | «deny хука — терминальное решение», «стадия не умеет разрешать», «старший слой deny» |
| (б) `ask` хука остаётся `ask`                       | Слой хука возвращает `blockType: 'ask'`; стадия в этом пути не участвует. Превращение в отказ в планировщике — только по явной политике (нет диалога / fixed-policy).              | «ask хука возвращается как ask», «превращение ask в отказ — по явной политике»       |
| (в) `allow` хука не отменяет детерминированный deny | Детерминированный блок (`blocked:destructive-command`) → `kind: 'blocked'` — терминальный отказ; исполнение не начинается, поэтому PreToolUse-хук для вызова не срабатывает вовсе. | «детерминированный блок терминален»                                                  |
| Порядок слоёв                                       | Индексы точек вызова в обоих файлах: `evaluateAutoMode(` < `firePermissionRequestHook(` < `firePreToolUseHook(`.                                                                   | «решение AUTO принимается до срабатывания хуков»                                     |

### 10.2 Почему это проверка, а не рассуждение

Тест «фактический порядок слоёв» читает исходники планировщика и ACP-сессии и падает с явным
текстом, если порядок изменится (фрагмент не найден или точки поменялись местами). Рантайм-seam
для порядка в этих файлах отсутствует, а §10.3 требует именно зафиксировать факт, а не
предположить его. Поведенческие части (а)–(в) проверяются на настоящих функциях слоёв
(`firePreToolUseHook`, `evaluateAutoMode`, `applyAutoModeDecision`, `runJevStage`) с фиктивным
транспортом; сеть не используется.

### 10.3 Остаточные связки, найденные при разборе (стадия в `shadow`, поэтому не блокируют)

1. **Серия блоков уводит вызов в ручное подтверждение.** При 3-м подряд блоке
   (`AUTO_MODE_DENIAL_LIMITS.maxConsecutiveBlock`) `applyAutoModeDecision` возвращает
   `kind: 'fallback'`, а не `blocked`: вызов уходит в диалог, перед которым срабатывает
   `PermissionRequest`-хук и может ответить `allow`. Права разрешать у стадии при этом не
   появляется (решение принимает политика upstream + оператор/хук), но блок `Jev` в этом
   состоянии не является терминальным. Тест: «подряд идущие блоки уводят вызов в ручное
   подтверждение». Вопрос архитектору: должен ли `block` стадии быть исключён из denial-streak
   fallback при переводе в `block-only`.
2. **ACP-путь трактует хук-`ask` как отказ.** В `Session.ts` ветка `!preHookResult.shouldProceed`
   возвращает ошибку исполнения без bounce (в ядре `ask` уходит в `awaiting_approval`, если
   оператора можно спросить). Это поведение upstream, стадия в нём не участвует; фиксируется как
   разница путей, а не как дефект стадии.
3. **Fixed-policy вызовы не проходят через стадию вовсе.** `executionOrigin.kind === 'fixed_policy'`
   пропускает L3→L5 (включая `classifyAction`) и оставляет только PreToolUse-хук. То есть
   детерминированные слои и Jev-стадия не гейтят такие вызовы; для стадии это означает, что
   «полное покрытие AUTO» — не её свойство, а свойство периметра врезки (ADR-002 §1).
4. **IDE-схема остаётся терпимой к неизвестным ключам.** Генератор JSON Schema не эмитит
   `additionalProperties: false` ни для одного объектного блока, а загрузчик настроек не валидирует
   `jsonSchemaOverride` (комментарий в `core/config/config.ts`). Поэтому «ошибка схемы» реализована
   как ошибка загрузки настроек в CMP-006 — единственное место, где такие ошибки действительно
   исполняются. Публикуемая схема добавляет только описание полей.

## 11. Вывод проверок пакета 2

```console
$ cd packages/core && npx vitest run src/jev
 Test Files  11 passed (11)
      Tests  143 passed (143)          # было 118 в Rev 1 (+25: cli-wiring 11, hook-compat 11, форма блока 3)

$ cd packages/core && npx vitest run src/permissions
 Test Files  13 passed (13)
      Tests  1070 passed (1070)

$ cd packages/core && npx vitest run src/hooks src/core/toolHookTriggers.test.ts
 Test Files  31 passed (31)
      Tests  1220 passed (1220)

$ cd packages/core && npx tsc --noEmit          # 0 ошибок
$ cd packages/cli  && npx tsc --noEmit          # 0 ошибок
$ npm run build                                  # сборка репозитория прошла (включая генерацию settings.schema.json)

$ npx eslint <изменённые файлы>                  # без замечаний
$ arch-ml control check .
Правил: 28, нарушений: 0 (error: 0, warn: 0)
Итог: PASS

$ npx tsx -e "…jevQuestionSpecSha256()"
a3adbdb9407d9a98df540f6c7d3476de30708028b916b878ef296a0092e0f96d   # не изменился

$ cd packages/cli && npx vitest run src/config/settingsSchema.test.ts
      Tests  60 passed (60)
$ cd packages/cli && npx vitest run src/config/config.test.ts
      Tests  482 passed (482)         # проводка не сломала существующие пути загрузки конфига
$ cd packages/cli && npx vitest run src/config/settings.test.ts src/config/settingsUtils.test.ts
      Tests  279 passed (279)
```

Правило `upstream_delta_scope` не нарушено: правки в `packages/core/src` и `packages/cli/src` —
только `jev/**` (включая новые тесты), `config/config.ts`, `config/settingsSchema.ts`.

## 12. Тесты пакета 2 в разрезе требований задачи

| Требование задачи                                        | Тест                                                                                                                        |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| (а) `settings.json` с `jev.enabled=true` доходит до ядра | `cli-wiring.test.ts` → «enabled=true из settings.json видно в Config»                                                       |
| (б) отсутствие ключа → стадия выключена                  | `cli-wiring.test.ts` → «отсутствие ключа jev → стадия выключена» (и T1 прежней ревизии)                                     |
| (в) `enforce` и внешний endpoint отвергаются на загрузке | `cli-wiring.test.ts` → «mode=enforce отвергается на загрузке настроек», «внешний endpoint отвергается на загрузке настроек» |
| (г) частичный блок добирается дефолтами                  | `settings.test.ts` → «частичный блок добирается дефолтами ядра»; `cli-wiring.test.ts` → одноимённый тест                    |
| Неизвестные поля внутри `jev` — ошибка                   | `settings.test.ts` → «неизвестный ключ внутри jev», «блок jev не-объект»; `cli-wiring.test.ts` → те же на уровне `Config`   |
| §10.3(а)(б)(в) и порядок слоёв                           | `hook-compat.test.ts` (11 тестов)                                                                                           |
