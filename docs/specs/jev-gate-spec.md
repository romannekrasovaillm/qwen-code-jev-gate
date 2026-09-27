# Дельта-спека: Jev-стадия (stage-0) в гейте разрешений AUTO

- Проект: форк `qwen-code` (ветка `arch/jev-stage1`), маршрут **Critical**.
- Решения: ADR-001 (дельта форка), ADR-002 (стадия-0, escalate-only), ADR-003 (адаптер, периметр).
- Инварианты: `ARCHITECTURE-SPINE.md` AD-1 … AD-9.
- Статус: **readiness-гейт пройден с оговорками** (см. §7) → допустим handoff на walking skeleton.
- Область: только режимы `shadow` и `block-only`. Режим `enforce` — вне области (AD-9).

## 1. Компоненты и владение

| ID      | Компонент                  | Путь                                                                                                                    | Ответственность                                                                                       |
| ------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| CMP-001 | Врезка стадии-0            | `packages/core/src/permissions/classifier.ts` (правка), `packages/core/src/jev/stage.ts` (логика)                       | Вызвать Jev-стадию перед stage-1 LLM, собрать `ClassifierResult`, соблюсти монотонность и fail-closed |
| CMP-002 | Адаптер бэкенда            | `packages/core/src/jev/backends/{systemone,dryrun}.ts`, `packages/core/src/jev/backends/index.ts`                       | Транспорт к бэкенду, маппинг ответа в `JevDecision`, таймаут, классификация ошибок                    |
| CMP-003 | Гейт-логика                | `packages/core/src/jev/gate.ts`                                                                                         | Инварианты ответа (AD-5), вычисление вердикта, синтез `reason` (AD-6)                                 |
| CMP-004 | Сборщик `state` и редакция | `packages/core/src/jev/state.ts`                                                                                        | Белый список полей `state`, детерминированная редакция секретов (AD-4, AD-8)                          |
| CMP-005 | Журнал решений             | `packages/core/src/jev/decision-log.ts`                                                                                 | Запись фиксированной схемы, ротация, отсутствие содержимого (AD-7)                                    |
| CMP-006 | Настройки                  | `packages/core/src/jev/settings.ts`, правка `packages/core/src/config/config.ts` + проводка `packages/cli/src/config/*` | Резолв `settings.jev`, дефолты, валидация режима и периметра                                          |
| INT-001 | Локальный носитель решений | `http://127.0.0.1:11435/v1/systemone` (loopback, Jev-совместимая форма)                                                 | Типизированные решения (`choice`/`score`/`noul`), `probabilities`, `confidence`                       |

Владение: CMP-001…CMP-006 — команда проекта (ветка форка); INT-001 — внешний вендор (TypeSafe, early access).

## 2. Контракты (TypeScript)

```ts
// packages/core/src/jev/types.ts

export type JevMode = 'shadow' | 'block-only'; // 'enforce' вне области v1 (AD-9)
export type JevBackendName =
  | 'system-one-http'
  | 'system-one-adapter'
  | 'dry-run'; // Rev 2: вендорский бэкенд выведен (ADR-004)

export interface JevSettings {
  enabled: boolean; // дефолт: false (AD-1)
  mode: JevMode; // дефолт: 'shadow'
  backend: JevBackendName; // дефолт: 'system-one-http' (локальный Jev-совместимый сервер, ADR-004)
  endpoint: string; // дефолт: 'http://127.0.0.1:11435/v1/systemone' — локальный носитель (Ollaya)
  internalHosts: string[]; // дефолт: ['127.0.0.1', '::1', 'localhost'] — единственные допустимые хосты (AD-10)
  timeoutMs: number; // дефолт: 3000 — по эмпирике пилота-01 (CPU-носитель 2.2 с); пересматривается по пилоту
  dailyBudgetUsd?: number; // предохранитель стоимости (для локального носителя — не требуется, поле сохранено для adapter)
  decisionLogPath?: string; // дефолт: вне репозитория, каталог сессии
}

/** Вопросы описываются данными, не кодом бэкенда: один спек — все кандидаты пилота. */
export interface JevQuestionSpec {
  type: 'choice' | 'noul' | 'score';
  instructions: string;
  criteria?: Record<string, string | null> | string[];
}

export type JevQuestionSet = Record<string, JevQuestionSpec>;

export interface JevAnswerChoice {
  type: 'choice';
  choice: string;
  probabilities: Record<string, number>;
  confidence: number;
}
export interface JevAnswerNoul {
  type: 'noul';
  noul: number;
}
export interface JevAnswerScore {
  type: 'score';
  score: number;
  probabilities: Record<string, number>;
  confidence: number;
}
export type JevAnswer = JevAnswerChoice | JevAnswerNoul | JevAnswerScore;

export interface JevDecision {
  answers: Record<string, JevAnswer>;
  model: string; // 'jev-latest' | имя LLM-adapter | 'dry-run'
  backend: JevBackendName;
  durationMs: number;
  usage?: { inputTokens: number; outputTokens: number };
  /** Инфраструктурный отказ (таймаут/сеть/schema/невалидные вероятности). Семантика = ClassifierResult.unavailable (AD-3). */
  unavailable?: boolean;
  unavailableReason?:
    | 'timeout'
    | 'transport'
    | 'http'
    | 'schema'
    | 'budget'
    | 'circuit-open';
}

export interface JevBackend {
  readonly name: JevBackendName;
  decide(args: {
    state: Record<string, unknown>; // результат CMP-004 (белый список)
    questions: JevQuestionSet;
    signal?: AbortSignal;
    timeoutMs: number;
  }): Promise<JevDecision>;
}
```

```ts
// packages/core/src/jev/stage.ts — контракт стадии-0

export interface JevStageInput {
  toolName: string;
  toolParams: Record<string, unknown>;
  filePath?: string;
  isPathWithinWorkspace: boolean;
  cwdClass: 'workspace' | 'external';
  /** Вердикт детерминированных слоёв, посчитанный ДО стадии (AD-2). */
  determinsticVerdict: 'allow-eligible' | 'deny' | 'ask';
  config: Config;
  signal?: AbortSignal;
}

export interface JevStageOutcome {
  /** true — стадия принимает решение сама; false — управление уходит в существующий LLM-каскад. */
  decided: boolean;
  shouldBlock?: boolean;
  reason?: string; // синтезирован кодом (AD-6)
  decision: JevDecision; // всегда присутствует (для журнала и shadow-режима)
}

export async function runJevStage(
  input: JevStageInput,
): Promise<JevStageOutcome>;
```

Правила контракта:

- `runJevStage` **никогда** не бросает исключение: любой отказ превращается в `unavailable` + `decided=false` (AD-3).
- `decided=true` допустимо только при `shouldBlock=true` (режим `block-only`) и `mode='block-only'`. В `shadow` — всегда `decided=false`.
- `determinsticVerdict ∈ {deny, ask}` → стадия не вызывается (ранний выход в врезке, AD-2).

## 3. Спек вопросов (заморожен, AD-9)

Спек — данные, а не код: одна константа, используемая и в ядре, и в пилоте. Хеш спека печатается в журнал и отчёт. Смена любой формулировки = новая версия = новый прогон пилота.

```ts
// packages/core/src/jev/questions.ts  — ЗАМОРОЖЕНО v1 (spec_hash печатается в decision log)
export const JEV_QUESTIONS_V1: JevQuestionSet = {
  permission_class: {
    type: 'choice',
    instructions: 'Which permission category does `proposed_call` fall into?',
    criteria: {
      read_only:
        'Reads files, queries data, observes state. No mutation of anything.',
      write_local:
        'Creates or modifies files inside the working directory only.',
      network: 'Sends data to or fetches data from an external service.',
      spend: 'Moves money, buys resources, or incurs billable cost.',
      destructive:
        'Deletes data, drops tables, force-pushes, or overwrites history irreversibly.',
      other: 'None of the above.',
    },
  },
  is_destructive: {
    type: 'noul',
    instructions: 'Could `proposed_call` irreversibly destroy data or state?',
  },
  is_reversible: {
    type: 'noul',
    instructions:
      'Can the effect of `proposed_call` be undone without data loss?',
  },
  touches_secrets: {
    type: 'noul',
    instructions:
      'Does `proposed_call` read, print, or transmit credentials or private keys?',
  },
  exfiltrates_data: {
    type: 'noul',
    instructions:
      'Does `proposed_call` send local or user data to an external destination?',
  },
};
```

Требования к формулировкам (из скилла `jev-guardrails-tool-gating`, проверяются на ревью):

- Политика — в доверенном поле `policy` (строка-константа в `state.ts`), недоверенные аргументы — в `proposed_call` с маркерами; **никогда не смешивать**.
- `criteria` описывают **разделяющие** категории, а не синонимы; `other` обязателен.
- Никаких имён моделей/цен/дат внутри вопросов — это знание кода.
- Noul-вопрос = один тип угрозы. Разрушительность (`is_destructive`) — **независимый** гейт: проверяется даже при уверенном `write_local`.

## 4. Сборка `state` (CMP-004)

```ts
export interface JevStateInput {
  toolName: string;
  toolParams: Record<string, unknown>;
  cwdClass: 'workspace' | 'external';
  isPathWithinWorkspace?: boolean;
  deterministicFlags: {
    destructiveRule?: string;
    forcedAskRule?: string;
    externalWrite?: boolean;
  };
}

/** Белый список: всё, что не перечислено, в state не попадает (AD-4). */
export function buildJevState(input: JevStateInput): Record<string, unknown>;
```

Разрешённые поля `state`: `policy` (константа), `proposed_call { tool, command|args (после редакции), path_class, cwd_class }`, `deterministic { flags }`. Запрещено: содержимое файлов, результаты инструментов, транскрипт, `env`, любые значения, похожие на секреты.

Редакция секретов — детерминированная функция с тестами:

```ts
export function redactSecrets(value: string): {
  value: string;
  redacted: number;
};
// Паттерны: значения после KEY=/TOKEN=/PASSWORD=/SECRET=, Bearer/JWT, PEM-блоки,
// AWS/GCP-подобные идентификаторы, URL userinfo (scheme://user:pass@host), длинные base64/hex-строки.
```

Правила: редакция выполняется **до** формирования запроса; число вырезанных мест пишется в журнал (без значений); ложные срабатывания разбираются в пилоте (ADR-003, отрицательные последствия).

## 5. Вердикт гейта (CMP-003)

```ts
export interface JevGateResult {
  verdict: 'block' | 'no-decision';
  reason?: string; // синтезирован кодом, детерминированный шаблон
  invariantViolation?:
    | 'read_only_vs_destructive'
    | 'secrets_and_exfiltration'
    | 'unknown_class';
}

export function evaluateJevGate(d: JevDecision, mode: JevMode): JevGateResult;
```

Логика v1 (escalate-only):

| Условие                                                 | Исход                                                           |
| ------------------------------------------------------- | --------------------------------------------------------------- |
| `unavailable` (таймаут/сеть/schema/бюджет/circuit)      | `no-decision` → LLM-каскад (AD-3)                               |
| `permission_class` вне объявленного множества           | `no-decision` + `invariantViolation='unknown_class'`            |
| `read_only` ∧ `is_destructive > 0.9`                    | `no-decision` + `invariantViolation='read_only_vs_destructive'` |
| `touches_secrets > 0.5` ∧ `exfiltrates_data > 0.5`      | `block` (жёсткий исход, не зависящий от confidence)             |
| `is_destructive > 0.5` ∧ `is_reversible < 0.7`          | `block`                                                         |
| `permission_class = 'destructive'` ∧ `confidence ≥ 0.7` | `block`                                                         |
| иначе                                                   | `no-decision` (решает существующий путь)                        |

`reason` — только из шаблонов вида:

- `Jev: destructive and not reversible (is_destructive=0.93, is_reversible=0.12) — manual review required`
- `Jev: credentials read plus external transmission (touches_secrets=0.88, exfiltrates_data=0.91) — action blocked`

Пороги выше — **стартовые консервативные** (решение принято до калибровки, AD-9); они зафиксированы константами с комментарием «до reliability curve» и меняются только по решению владельца с отчётом пилота. Ответ Jev никогда не приводит к `allow` (AD-2).

## 6. Врезка и настройки

Врезка — единственная, в `classifier.ts` в начале `classifyAction()` (после построения входа, до `sideQuery` stage-1):

```ts
// ADR-002 §1: единственная точка врезки (AD-1).
const jev = await runJevStage({ /* ... */ });           // никогда не бросает (AD-3)
if (jev.decided) {
  return {
    shouldBlock: jev.shouldBlock!,
    reason: jev.reason!,
    model: jev.decision.model,
    durationMs: jev.decision.durationMs,
    usage: jev.decision.usage && { inputTokens: ..., outputTokens: ... },
    stage: 'jev',
  } satisfies ClassifierResult;
}
// иначе — без изменений: существующий stage-1 → stage-2
```

Изменения в upstream-файлах строго ограничены (AD-1): `classifier.ts` — врезка + расширение union `stage` значением `'jev'`; `config/config.ts` — новый опциональный блок настроек и резолвер. Настройки (дефолты — выключено):

```jsonc
// settings.json (фрагмент)
{
  "jev": {
    "enabled": false,
    "mode": "shadow",
    "backend": "system-one-http",
    "endpoint": "http://127.0.0.1:11435/v1/systemone",
    "timeoutMs": 3000,
  },
}
```

Валидация настроек — в `CMP-006`: неизвестный `mode`/`backend` → ошибка конфигурации; `mode='enforce'` отклоняется (AD-9); `backend='typesafe-api'` без `TYPESAFE_API_KEY` в env → ошибка конфигурации с явным сообщением (не тихий откат к другому бэкенду, ADR-003 §2).

## 7. Обязательные тесты (конформность, AD-1 … AD-9)

Walking skeleton считается доказанным только при зелёных тестах ниже. Каждый тест ссылается на инвариант.

| #   | Тест                                                                                                                                                              | Инвариант                |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ | ---- |
| T1  | `settings.jev.enabled=false` → решения AUTO идентичны upstream (запуск того же входа через `autoMode` до/после врезки: одинаковые `via`, `shouldBlock`, `reason`) | AD-3                     |
| T2  | `determinsticVerdict ∈ {deny, ask}` → бэкенд Jev не вызывается вовсе (spy на транспорт, 0 вызовов)                                                                | AD-2                     |
| T3  | Jev вернул `block` там, где детерминированные слои дали `allow-eligible` → решение `block`; обратный случай (Jev «no-decision» поверх `deny`) — решения не меняет | AD-2                     |
| T4  | Таймаут бэкенда (`dry-run` с искусственной задержкой > `timeoutMs`) → `unavailable='timeout'`, `decided=false`, управление ушло в stage-1 (spy: stage-1 вызван)   | AD-3                     |
| T5  | HTTP 500 / битый JSON / вероятности не в [0,1] или не суммируются к 1 → `unavailable='schema'                                                                     | 'http'`, `decided=false` | AD-3 |
| T6  | 3 отказа подряд → circuit breaker открыт, 4-й вызов не идёт в сеть (`circuit-open`), существующий путь работает                                                   | AD-3                     |
| T7  | `read_only` ∧ `is_destructive=0.95` → `no-decision` + `invariantViolation='read_only_vs_destructive'`                                                             | AD-5                     |
| T8  | `touches_secrets=0.9` ∧ `exfiltrates_data=0.9` → `block`, независимо от `permission_class` и `confidence`                                                         | AD-5                     |
| T9  | `state` из аргументов с `GITHUB_TOKEN=ghp_…`/`Bearer eyJ…`/PEM → в запросе этих подстрок нет; в журнале — только счётчик вырезаний                                | AD-4, AD-8               |
| T10 | В `state` отсутствуют ключи `messages`, `transcript`, `fileContent`, `env` при любых входных параметрах (property-based прогон по случайным tool-params)          | AD-4                     |
| T11 | Записи журнала валидируются схемой; аргументы инструмента и транскрипт в записи отсутствуют                                                                       | AD-7                     |
| T12 | `reason` при блоке равен одному из фиксированных шаблонов (snapshot) и не содержит текста, порождённого бэкендом                                                  | AD-6                     |
| T13 | `mode='enforce'` в конфиге → ошибка валидации; `backend='typesafe-api'` без `TYPESAFE_API_KEY` → ошибка с явным сообщением                                        | AD-9, ADR-003            |
| T14 | Дефолтный конфиг репозитория содержит `jev.enabled=false`; поиск по репозиторию не находит реального значения ключа                                               | AD-1, AD-8               |

Тесты пишутся в стиле репозитория (vitest, `packages/core/src/jev/__tests__/`, фиктивные транспорты, без сети).

## 8. Критерии приёмки walking skeleton (гейт A4)

1. `pnpm -C <repo> test packages/core` — зелёный, включая T1–T14.
2. `pnpm -C <repo> build` — сборка проходит (проверка, что `stage: 'jev'` принят типами во всех местах использования).
3. Демонстрационный прогон в режиме `shadow` на локальных сценариях (без сети, `backend='dry-run'`): журнал содержит записи по каждому вызову инструмента, решения не изменились (сравнение с `enabled=false`).
4. Дифф против `upstream/main`: изменённые upstream-файлы — только `permissions/classifier.ts` и `config/config.ts`; строка-врезка помечена ссылкой на ADR-002.
5. `fitness_check` по `.arch-handoff/CONSTRAINTS.yaml` — PASS.
6. Отчёт-эвиденс (короткий md в `docs/specs/` или `pilot/`) с выводом тестов, списком изменённых файлов и хешем спека вопросов.

## 9. План отката

- **L0 (мгновенный, без кода):** `settings.jev.enabled=false` — поведение возвращается к upstream-пути (AD-3, T1 это доказывает).
- **L1 (сессионный):** circuit breaker — стадия выключается сама при серии отказов, без вмешательства.
- **L2 (кодовый):** `git revert` коммита врезки в ветке `arch/jev-stage1`; пакет `packages/core/src/jev/**` изолирован, удаляется целиком.
- **L3 (полный):** drop ветки `arch/jev-stage1`; основное дерево владельца и upstream-история не тронуты (ADR-001).
- Сигналы к откату: рост ложных блоков на реальных сессиях, `unavailable` > 20 % вызовов, p95 Jev-стадии > бюджета, любые сомнения в корректности вердикта → L0 немедленно.

## 10. Readiness-гейт: **PASS с оговорками** (перед handoff)

Пройдено: маршрут определён (Critical), ADR-001…003 записаны, спайн чист (`spine_lint`), точка врезки подтверждена чтением кода, контракт `ClassifierResult` зафиксирован, тесты и критерии приёмки сформулированы до реализации, откат описан.

Оговорки (обязательны к снятию до расширения области — переходу в `enforce` и в другие точки):

1. **Ключа `TYPESAFE_API_KEY` нет** → пилот стартует в сухом режиме, боевой бэкенд не проверялся; `typesafe-api` не считается доказанным до прогона пилота.
2. **Пороги в §5 не калиброваны** на нашем корпусе — они консервативны и не дают права на `allow`.
3. **Порядок применения хук-решений (`permissionDecision`) и Jev-стадии — верифицирован** (пакет 2, коммит `7c0d564f4e`): `детерминированные слои AUTO → Jev-стадия (внутри classifyAction) → L4/подтверждение (PermissionRequest-хук) → исполнение (PreToolUse-хук)`. Хуки применяются **после** стадии. Требования «deny хука не ослабляется стадией», «`ask` остаётся `ask`», «`allow` не отменяет детерминированный deny» выполняются конструктивно (стадия отрабатывает раньше и не имеет исхода «разрешить»). Остаточная связка «block → denial-streak → ручное подтверждение → `allow` от хука» закрыта инвариантом **AD-13** (терминальность block-вердикта).
4. **Русскоязычный контент** в аргументах инструментов: качество Jev на нём не измерено (пилот обязан включать русскоязычную подвыборку).
5. **Незакоммиченный мусор в основном дереве форка** (`games/snake`, `run-deepseek-agent.sh`) — работа ведётся в worktree, но при интеграции ветки его необходимо развести по отдельным коммитам/PR.

## 11. Открытые вопросы (владельцу/владельцу домена)

- Разрешение на боевой вызов внешнего вендора из контура разрешений (ADR-003 §4–5): подтвердить закрытый периметр (без транскрипта) как достаточный.
- Допустимо ли включать Jev-стадию для сессий с чужими/клиентскими репозиториями (приватность третьих лиц).
- Нужен ли отдельный контур для `guardrails` на входе промпта сразу после v1 (Deferred спайна) — приоритет относительно Jev-роутинга моделей.

## 12. Решения архитектора по расхождениям реализации (Rev 2, 2026-09-26)

Основание: прогон `hr-20260926-200537-00` (claude-code, 1519.9 с, `status=partial`), 4 конфликта, 6 открытых вопросов. Ниже — решения; принятое — в силе, непринятое — с причиной.

**C1. Точки врезки нет в baseline форка. Признано: это ошибка архитектора в фактуре. РЕШЕНО (ADR-005).**
`packages/core/src/permissions/classifier.ts` и весь AUTO-каскад (`autoMode.ts`, `dangerousRules.ts`, `classifier-prompts/*`) присутствуют в `upstream/main`, но **отсутствуют** в `main` форка (`db7ec117c`, 2026-03-26). Решение владельца A3 (2026-09-26):

- **База проекта — актуальный `upstream/main`** (`61e7b92b09`, 26.09.2026), рабочая ветка `arch/jev-gate` (ADR-005). Ветка `arch/jev-stage1` — архив-источник кода и тестов.
- **Seam снят**: врезка делается в настоящий `classifyAction()` актуальной базы. Перенос пакета `packages/core/src/jev/**` — `git checkout arch/jev-stage1 -- packages/core/src/jev`.
- Оговорка §10.3 (порядок hook-решений `permissionDecision` и Jev-стадии) сохраняется и теперь проверяема: механизм в актуальной базе существует (`hooks/types.ts`, `hookSpecificOutput.permissionDecision`), тест совместимости обязателен до включения `block-only` на реальных сессиях.

**C2. Расширение `JevSettings` полями `endpoint` и `internalHosts` — принято.** Без них AD-10/ADR-004 невыполнимы. Спека §2 обновлена.

**C3. Переименование бэкенда `typesafe-api` → `system-one-http` — принято** (Rev 2, ADR-004). Имя вводило в заблуждение: endpoint локальный, вендор выведен из контура; `system-one-http` описывает протокол, а не вендора.

**C4. Дефолтный endpoint — `http://127.0.0.1:11435/v1/systemone`, timeout 3000 мс.** Это фактический адрес развёрнутого носителя (Ollaya на GB10, модель `laya:en`). Бюджет 1200 мс из §2 Rev 1 отменён: пилот-01 дал 2.2 с медианы на CPU-пути (6.7 с на русскоязычном запросе), то есть 1200 мс недостижимы на текущем носителе. 3000 мс — рабочее значение с пометкой «пересматривается»; латентностный бюджет как NFR остаётся открытым (см. `pilot/pilot-report-01-laya-zeroshot.md` §«Что меняется»).

**C5. Fitness-коллизия `no_hardcoded_secret_values` — исправлена в ruleset.** Правило сужено до наших артефактов (`packages/core/src/jev/**`, `pilot/**`, `docs/adr/**`, `docs/specs/**`, `ARCHITECTURE-SPINE.md`) с `exclude_glob: docs/users/**`. Upstream-документация с примером `TAVILY_API_KEY` под правило не попадает — правка upstream-файлов по-прежнему запрещена (AD-1), и это правильно: наш ruleset не должен требовать изменений в upstream.

**C6. Манифест носителя в ядре не заводим.** Достаточно: (а) поля журнала решений (`backend`, `model`, `digest`, `specHash`) — AD-7; (б) `pilot/local-backends-survey.md` как реестр допущенных носителей с лицензиями — AD-11. Манифест в ядро не тащим: это данные поставки, а не логика гейта (AD-1, минимизация дельты).

**Не принято как отдельные пункты, но фиксируется:** замечание «вне `jev/**` только два разрешённых файла» — подтверждено (`git diff --stat` харнесса: 21 файл, 5332 вставки, вне `jev/**` — только `classifier.ts` и `config/config.ts`); 106 тестов в 8 файлах зелёные; `tsc`/eslint/сборка без ошибок; 1 предсуществующий красный тест (`agent-statistics`, локаль-зависимый) — не блокирует (A4 принимает его как известный дефект upstream, фиксируем в эвиденсе).

## 13. Что осталось непокрытым (для следующего пакета)

1. ~~Смена базы ветки на актуальный upstream (C1)~~ — **выполнено решением ADR-005**: база `arch/jev-gate` от `upstream/main` (`61e7b92b09`); перенос пакета и новая врезка — задачи следующего пакета.
2. Тест совместимости hook-решений и Jev-стадии (оговорка §10.3).
3. Матрица кандидатов носителя (`kev` 0.8B/4B, `decider:0.8b`, `laya:multilingual`) на замороженном спеке + курированный набор 200–300 примеров (пилот-01 дал только разведку на 7 примерах).
4. Домен-адаптация (fine-tune/temp-fit) — без неё класс разрушительности остаётся недооценённым (`rm -rf` → 0.34).

## 12-бис. Решения архитектора по итогам прогона на базе upstream/main (Rev 3, 2026-09-26)

Основание: прогон `hr-20260926-205640-02` (`status=complete`, коммит `6e34e588d3`, 118/118 тестов `jev`, 1070/1070 `permissions`, `tsc` 0, fitness PASS, хеш спека не изменился).

**C7. `permissions/autoMode.ts` — ратифицирован как третье объявление дельты.** Правка ровно одна: аддитивный union `AutoModeDecision.stage` со значением `'jev'`, без логики. Причина: union объявлен в базе дважды (`ClassifierResult` и `AutoModeDecision`), без второго объявления не проходит `tsc` (критерий §8.2). Отклонение от «ровно двух файлов» признано обоснованным и ратифицировано в ADR-001 Rev 2; правило `upstream_delta_scope` в `CONSTRAINTS.yaml` — единственный источник правды о составе дельты.

**C8. Проводка `settings.jev` через CLI разрешена** (`packages/cli/src/config/config.ts`, `packages/cli/src/config/settingsSchema.ts`): без неё настройка из `settings.json` не доходит до ядра и стадия включается только программно — то есть фича неработоспособна для пользователя. Правки аддитивные (ключ в схеме + проброс в `ConfigParameters`), логика по-прежнему только в `packages/core/src/jev/**`. Правило дельты расширено на два CLI-файла.

**C9. Имена файлов пакета — kebab-case** (`decision-log.ts`, `backends/systemone.ts`) по требованию eslint репозитория; спека §1/§2 приведена в соответствие. Это же сняло конфликт «спека против реализации» без изменения поведения.

**C10. Предохранитель `dailyBudgetUsd` по умолчанию не применяется.** Для локального носителя цена решения нулевая (ADR-004), поэтому дефолт-значение из Rev 1 отменено; поле остаётся доступным явной настройкой — на случай возврата платного бэкенда (что потребует отмены AD-10).

**C11. Правило AD-8 переформулировано харнессом корректно:** вместо «чтения `TYPESAFE_API_KEY` из окружения» действует `no_vendor_key_in_core` — прямой запрет имени ключа и `process.env` в коде пакета. Это точнее исходной формулировки после ADR-004: ключа вендора в контуре нет вообще, а не «читается безопасно». Принято.

**C12. Поломка, найденная полным прогоном, устранена правильно.** Хеш спека вопросов считался на уровне модуля, из-за чего `questions.ts`, став достижимым из классификатора, ломал `src/tools/shell.test.ts` (мок `crypto`). Переведено на ленивый `jevQuestionSpecSha256()`; значение хеша не изменилось. Это подтверждает ценность полного прогона, а не только прогона по своему пакету — критерий §8 «зелёный `packages/core`» остаётся обязательным.

**Остаётся открытым (перенесено в следующий пакет):**

1. **Проводка настроек** (C8) — реализовать, затем повторить проверки.
2. **Тест совместимости хуков и стадии** (оговорка §10.3): порядок применения `hookSpecificOutput.permissionDecision` (`allow|deny|ask`) относительно `stage-0` — обязателен до включения `block-only` на реальных сессиях. Формулировка требования: (а) `deny` от хука не может быть ослаблен Jev-стадией; (б) `ask` от хука не превращается в `block` без политики; (в) `allow` от хука не отменяет детерминированный `deny` (AD-2).
3. **Бюджет латентности как NFR** — открыт (пилот-01: 2.2 с медиана CPU; рус. 6.7 с). Решение о целевом бюджете требует данных пилота по носителю на нашей машине.

## 13. Что осталось непокрытым (для следующего пакета)

1. ~~Смена базы ветки на актуальный upstream (C1)~~ — **выполнено**: база `arch/jev-gate` от `upstream/main` (`61e7b92b09`); врезка сделана в настоящий `classifyAction()` (коммит `6e34e588d3`).
2. Проводка `settings.jev` (C8) и тест совместимости хуков (§10.3).
3. Матрица кандидатов носителя (`laya:multilingual`, `kev` 0.8B/4B, `decider:0.8b`, `baseline_ml`, `existing_llm`) на замороженном спеке + курированный набор 200–300 примеров (пилот-01 — разведка на 7 примерах).
4. Домен-адаптация (fine-tune/temp-fit): без неё класс разрушительности недооценён (`rm -rf` → 0.34).

## 12-тер. Решения архитектора по итогам пакета 2 (Rev 4, 2026-09-27)

Основание: прогон `hr-20260927-054006-03` (`status=complete`, коммит `7c0d564f4e`; 1213 тестов core, 1220 hooks, tsc/build/eslint чисто, fitness 28/28 PASS, хеш спека неизменен).

**C13. Фактический порядок применения хуков ратифицирован.** Принят как есть: `детерминированные слои AUTO → Jev-стадия → L4/подтверждение (PermissionRequest) → исполнение (PreToolUse)`. Обоснование: стадия отрабатывает раньше хуков и не имеет исхода «разрешить», поэтому три исходных требования выполняются конструктивно, без дополнительного кода. Формулировка §10.3 спеки приведена в соответствие. Стадия остаётся в `shadow` до закрытия C14 (AD-13) — тогда можно включать `block-only`.

**C14. Терминальность block-вердикта — новый инвариант AD-13 (принято).** Найденная связка «`block` → 3 подряд → fallback в ручное подтверждение → `PermissionRequest`-хук отвечает `allow`» — это обход нашей политики чужой механикой. Решение: `block` стадии терминален, в `denial-streak` не участвует, автоматическим `allow` от хука не отменяется; обход — только изменение настроек или решение человека в штатном диалоге. Реализация — пакет 3.

**C15. `executionOrigin.kind='fixed_policy'` — вне области стадии (принято, занесено в Deferred).** Такие вызовы инициирует политика агента, а не модель; гейтить их Jev-стадией — расширение периметра без обоснования. Условие возврата зафиксировано: если `fixed_policy`-вызов начнёт формироваться из недоверенных данных — отдельный ADR.

**C16. IDE-подсветка `jev` в `settings.schema.json` — не расширяем (принято).** Схема — производный артефакт генератора; достаточно рантайм-валидации в ядре (`assertJevSettingsShape`, закрытая форма блока). Ошибка загрузки настроек — единственный источник правды; редакторская диагностика для одного ключа не стоит правки общего генератора. Занесено в Deferred с условием возврата.

**C17. Бюджет латентности — остаётся открытым NFR, но с кандидат-значением.** Факты: `laya:multilingual` (322M, CPU) — 0.71–1.0 с; `laya:en` (421M, CPU) — 2.1–6.7 с. Кандидат: **p95 ≤ 1200 мс** при носителе класса 322M и эскалации при превышении таймаута (`timeoutMs=3000` остаётся верхней границей деградации, не целью). Утверждение NFR — отдельным решением владельца после калибровки носителя (пилот-03 и далее).

**Принято к сведению (риски, не требующие решения сейчас):**

1. Новые тестовые файлы в `packages/cli/**` правилом дельты запрещены → проверка проводки настроек сделана двухслойно (поведенчески на ядре + структурно по исходникам CLI). Это приемлемый компромисс: расширять правило ради тестов CLI не будем — тест на ядре проверяет то же поведение.
2. Производный `settings.schema.json` (42 строки) меняется генератором; правило `upstream_delta_scope` смотрит только `packages/{core,cli}/src`, поэтому производный артефакт вне дельты — так и должно быть.
3. Закрытая форма блока `jev` (неизвестный ключ → ошибка загрузки) — правильное решение: опечатка `jev.modee` не должна проходить молча.
