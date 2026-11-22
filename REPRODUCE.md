# Воспроизведение модели Codex

Перенос принципов, не копия бинаря.

## 1. Вариативный промпт вместо одного

Шаблон с версией по модели/режиму. Общие блоки (personality, autonomy, permissions, editing
constraints) — единые; вариативность только в идентичности и акцентах режима.

## 2. Модель разрешений как данные

```
SandboxMode      = read-only | workspace-write | danger-full-access
ApprovalPolicy   = on-request | never | unless-trusted
ApprovalsReviewer= user | auto_review | guardian_subagent
```

Валидировать несовместимости **на старте** (`never` + `danger-full-access` запрещён) с явным
объяснением, а не молчаливым падением в read-only.

## 3. Авторизация переживает ходы

Хранить выданные разрешения в состоянии сессии. Не переспрашивать. Перед вопросом доводить работу до
состояния «конкретный ревьюируемый результат».

## 4. Auto-review классификатор

Отдельный дешёвый вызов: контекст = запрос + предыдущие 5 действий + предполагаемые 2.
Выход `high`/`low`. При `high` — блокирующее ревью. Отказ формулировать так, чтобы модель не
обходила его сайд-каналом:

> Do not bypass this rejection through a workaround or indirect execution.

## 5. Субагенты как треды

Канонические имена `/root/<parent>/<child>`, `fork_turns: none|all`. Спавн только по явному запросу
(«глубина/исследование» — не разрешение). Правила: сначала план, критический путь не делегировать,
disjoint write set, `wait_agent` скупо.

## 6. Хуки

Детерминированные точки вокруг модели: `SessionStart`, `PreToolUse`, `PostToolUse`, `PreCompact`,
`Stop`, `SubagentStart`. Матчеры + таймауты. Команда — внешний процесс, не код рантайма.

## Проверка

1. `never` + `danger-full-access` → понятная ошибка на старте.
2. Второй ход не переспрашивает ранее выданное разрешение.
3. Классификатор возвращает `high` на `rm -rf` вне workspace.
4. Субагент без явного запроса не спавнится.
5. `PreCompact` пишет чекпоинт до сжатия.

## 7. Rules — allowlist без модели

Файл правил, каждая строка — `prefix_rule(pattern=[...], decision="allow")`.
Семантика: команда начинается с `pattern` → разрешено, **без вызова модели и без approval**.
Писать правила под повторяющиеся проверки (`git fetch`, `ps`, `launchctl list`), чтобы снять с модели
решение о безопасности на рутине. Отключение — флагом `--ignore-rules`.

## 8. Per-tool approval у MCP

Разрешение задавать не на сервер, а на **конкретный тул**:

```toml
[mcp_servers.github.tools.create_pull_request]
approval_mode = "approve"
```

Так опасные операции (`create_pull_request`, `push_files`) требуют подтверждения, а чтение — нет.

## 9. Automations — расписание внутри харнеса

```toml
id = "inbox-sweep"
kind = "cron"
prompt = "…прочитай SKILL.md и следуй ей…"
rrule = "FREQ=DAILY;BYHOUR=4;BYMINUTE=1"
model = "gpt-5.6-sol"
reasoning_effort = "high"
notification_policy = "failed_runs_only"
target = { type = "project", project_id = "…" }
cwds = ["…"]
```

iCalendar-rrule вместо cron-выражения; модель и reasoning **на задачу**, а не глобально; уведомления
только о падениях; рядом `memory.md` — задача помнит свои прошлые прогоны.

## 10. Память как конвейер

Четыре уровня, а не один файл:

```
rollout_summaries/   — по файлу на сессию
raw_memories.md      — сырые записи
MEMORY.md            — группы: scope + applies_to: cwd=… + keywords + preferences + knowledge + failures
memory_summary.md    — выжимка (User Profile, preferences)
```

Модель извлечения и модель консолидации задаются отдельно (`extract_model`, `consolidation_model`).
Секция «Failures and how to do differently» обязательна — уроки фиксируются наравне с успехами.

## 11. Роли-TOML

```toml
name = "software-engineer"
description = "…когда использовать…"
developer_instructions = """
…роль, границы соседних ролей, что читать первым…
"""
```

Ролевая гигиена через явные запреты («не принимай за it-director решения», «не изображай приёмку
tech-reviewer»). Машинно-проверяемый allowlist скиллов — опция, а не обязательное поле.

## 12. Сессии и app-server

Rollout на тред: `sessions/<YYYY>/<MM>/<DD>/rollout-<ISO>-<uuid>.jsonl.zst`. Сжатие контекста
фиксируется отдельной записью `Compacted` **в самом rollout'е** — состояние восстановимо по логу.

Долгоживущий app-server по JSON-RPC/stdio (`initialize` → `thread/start`) позволяет управлять извне:
TUI, GUI и внешние лаунчеры — клиенты одного демона.

## Проверка (расширенная)

6. Команда из `prefix_rule` проходит без вопроса к модели.
7. `create_pull_request` требует approval, `list_issues` — нет.
8. Automation с `failed_runs_only` молчит на успехе.
9. `MEMORY.md` содержит `applies_to: cwd=…` — область применимости не теряется.
10. Rollout после компакции содержит запись `Compacted` и восстанавливается.
