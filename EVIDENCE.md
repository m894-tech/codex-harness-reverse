# Evidence: утверждение → проверка → статус

Статусы: **static** (строки бинаря/конфиг), **live** (файлы на машине), **NOT MEASURED**.

## Объект

| Утверждение | Проверка | Статус |
|---|---|---|
| Бинарь и размер | `ls -la .../CodexCLI.app/Contents/MacOS/codex` → 298 723 264 байт | live |
| Rust/Mach-O arm64 | `strings` даёт Rust-пути `cargo/registry/.../tokio-1.52.3` | static |
| Версия приложения | `defaults read CFBundleShortVersionString` → 26.1117.31558 | live |
| Версия CLI | `~/.codex/version.json` → latest_version 0.147.0 | live |
| Промпты в бинаре | `strings -n 60` → 53 056 строк, JSON с `instructions_template` | static |

## Промпт

| Утверждение | Проверка | Статус |
|---|---|---|
| 8 вариантов `instructions_template` | извлечено 8 уникальных, 17.8K–26.2K симв. | static |
| Вариант по модели | строки `based on GPT-6` / `based on GPT-5` | static |
| Авторизация переживает ходы | дословно в шаблоне | static |
| Приоритет пользователя над SKILL.md/AGENTS.md | дословно: `must take precedence over any guidelines provided in skills` | static |
| Отдельный persistent-режим | `persistent_instructions`, 5773 симв. | static |

## Разрешения

| Утверждение | Проверка | Статус |
|---|---|---|
| Три режима sandbox | enum `read-only`/`workspace-write`/`danger-full-access` | static |
| `never` + `danger-full-access` запрещён | строка 6746 с объяснением | static |
| Reviewer auto_review | enum `ApprovalsReviewer: user\|auto_review\|guardian_subagent` | static |
| Классификатор `high`/`low` | `classifier_instructions`, 3655 симв. | static |
| Запрет обхода отказа | `rejection_instructions` | static |

## Субагенты

| Утверждение | Проверка | Статус |
|---|---|---|
| Спавн только по явному запросу | протокол `spawn_agent` | static |
| Канонические имена | `/root/task1/task_3` | static |
| `fork_turns` none/all | дословно | static |
| Инструменты управления | enum `spawn_agent…interrupt_agent` | static |
| `wait_agent` скупо | протокол | static |

## Конфигурация на диске

| Утверждение | Проверка | Статус |
|---|---|---|
| 26 агентов-специалистов | `ls ~/.codex/agents/*.toml` → 26 | live |
| `developer_instructions` у всех | `grep -l` → 26/26 | live |
| AGENTS.md с 5 принципами | `cat ~/.codex/AGENTS.md` | live |
| Хуки PreCompact/SessionStart | `~/.codex/hooks.json` | live |
| Sandbox фактически | `config.toml`: `danger-full-access`, `approval_policy=never` | live |
| 72 скилла, lock-файл с git-источниками | `~/.agents/skills`, `.skill-lock.json` | live |
| Allowlist скиллов по роли | блок `MANAGED SKILL ACCESS` в `agents/devops-sre.toml` | live |
| Marketplace-плагины | `config.toml` секции `marketplaces.*`, `plugins.*` | live |
| Автоматизации | `ls ~/.codex/automations` → 10 | live |

## NOT MEASURED

| Что | Почему |
|---|---|
| Реальный вызов классификатора | нужен запуск сессии с auto_review |
| Работа `spawn_agent` end-to-end | не запускал вложенный тред |
| Поведение на несовместимой конфигурации | не ломал конфиг специально |
| Точность промптов после `strings` | возможны артефакты склейки в длинных литералах |

## Воспроизведение проверки

```bash
strings -n 60 /Applications/ChatGPT.app/Contents/Resources/codex-cli/CodexCLI.app/Contents/MacOS/codex > s.txt
grep -c 'instructions_template' s.txt
cat ~/.codex/config.toml | head -20
ls ~/.codex/agents | wc -l
cat ~/.codex/hooks.json
```

---

# Дополнение: слои, покрытые во второй итерации

## Инструменты и MCP

| Утверждение | Проверка | Статус |
|---|---|---|
| `exec_command` работает через PTY | строка «Runs a command in a PTY, returning output or a session ID» | static |
| `apply_patch` — единственный путь правки | промпт: `Do not create or edit files with cat or other shell write tricks` | static |
| Формат патча валидируется | ошибки `*** Begin Patch`, `*** End Patch`, `exactly one argument` | static |
| Per-tool approval у MCP | `config.toml`: `[mcp_servers.github.tools.create_pull_request] approval_mode = "approve"` | live |
| MCP stdio + remote url | `data-gateway` (command/args) и `clickhouse` (url) | live |
| Отдельный env на сервер | `[mcp_servers.github.env]` с прокси | live |
| `RawMcpServerConfig` 30 полей | строки бинаря | static |

## Rules — детерминированный allowlist

| Утверждение | Проверка | Статус |
|---|---|---|
| Формат `prefix_rule(pattern=[...], decision="allow")` | `~/.codex/rules/default.rules` | live |
| 60 правил на машине | `wc -l` → 60 | live |
| Решение без модели | семантика prefix_rule в правилах | static |
| Отключается `--ignore-rules` | примеры команд в самих правилах | live |

## Automations

| Утверждение | Проверка | Статус |
|---|---|---|
| 12 автоматизаций | `ls ~/.codex/automations` → 12 | live |
| Расписание — iCalendar rrule | `rrule = "FREQ=DAILY;BYHOUR=4;BYMINUTE=1"` | live |
| Модель на каждую задачу | `model = "gpt-5.6-sol"`, `reasoning_effort = "high"` | live |
| Уведомления только о падениях | `notification_policy = "failed_runs_only"` | live |
| Привязка к проекту | `target = { type = "project", project_id = "…" }` | live |
| Память на задачу | `memory.md` рядом с `automation.toml` | live |
| Джиттер запуска | `.run-jitter-salt` (uuid) | live |

## Память

| Утверждение | Проверка | Статус |
|---|---|---|
| 4 уровня | `raw_memories.md` 207 KB, `MEMORY.md` 150 KB, `memory_summary.md` 30 KB, `rollout_summaries/` | live |
| Группировка по Task Group с `applies_to: cwd=…` | `head MEMORY.md` | live |
| Модель извлечения и консолидации отдельно | `MemoriesToml` 15 полей в строках | static |
| Включено на машине | `[memories] generate_memories = true, use_memories = true` | live |

## Роли

| Утверждение | Проверка | Статус |
|---|---|---|
| 26 ролей | `ls ~/.codex/agents/*.toml` → 26 | live |
| Схема из 5 полей | `name`, `description`, `developer_instructions`, `[[skills.config]]`, комментарий-источник | live |
| Allowlist только у 2 из 26 | подсчёт `MANAGED SKILL ACCESS` → 2 (`software-engineer`, `devops-sre`) | live |
| Роли генерируются импортёром | заголовок `# Managed by m894-tech/agents codex_agents/import_agents.py` | live |
| `skills.config` выключает скиллы | `path` + `enabled = false` | live |
| Ролевая гигиена в промпте | «Не принимай за it-director…», «Не изображай приёмку tech-reviewer» | live |

## Сессии и app-server

| Утверждение | Проверка | Статус |
|---|---|---|
| Формат `rollout-<ISO>-<uuid>.jsonl.zst` | `find ~/.codex/sessions` | live |
| Сжатие zstd | расширение `.jsonl.zst` | live |
| Типы записей | enum'ы `SessionMeta`/`TurnContext`/`ResponseItem`/`RolloutItem`/`Compacted` в строках | static |
| App-server — JSON-RPC по stdio | пример `initialize`/`thread/start` в rules | live |
| События стрима | `EventMsg` enum, включая `GuardianAssessment` | static |
| 12 событий хуков | подсчёт вхождений в бинаре | static |

## Добавлено в NOT MEASURED

| Что | Почему |
|---|---|
| Точные JSON-схемы тулов | собираются кодом, в строках только описания |
| `skill_access.toml` | на машине отсутствует, есть только упоминание в промпте роли |
| `requirements.toml` | файла нет, известен по строкам валидации |
| `guardian_subagent` | enum есть, промпта не найдено |
| Фан-аут >1 треда | `max_concurrent_threads_per_session` не запускал |

---

# Промпты как система

| Утверждение | Проверка | Статус |
|---|---|---|
| 8 шаблонов в 5 группах | кластеризация по набору секций → 5 групп | static |
| Внутри группы тексты идентичны | diff #5 vs #6: одно пустое отличие | static |
| 3 секции всегда | `# Personality`, `# Working with the user`, строка-идентичность | static |
| Двухканальная модель | `commentary` / `final` + запрет вопросов в commentary | static |
| Плейсхолдеры только plugin_id/connector_id | подсчёт `{{ }}` → 2 имени, по 5 вхождений | static |
| `<environment_context>` инъекция | 12 вхождений маркера | static |
| Sandbox-текст — шаблон на режим | 3 отдельных текста по режимам | static |
| Скиллы: 3 уровня резолва | `# Using skills`: файловая ФС, environment, orchestrator через `skills.read` | static |
| Приоритет plugin > browser > computer use | `## Decision Boundary` | static |
| Анти-правило «не выдумывай требований» | `# Rules for getting work done` | static |
| Запрет контрастивного обрамления | `## Writing style`: `NEVER use contrastive framing` | static |
| Запрет точек с запятой и двоеточий | там же | static |
| Пути файловых ссылок | `### Formatting rules` | static |
| Дефект двойного экранирования | 194 литеральных `\n` в первом проходе, исправлено | live-проверено |
| Потеря апострофов при раскодировании | 34 случая, починено регэкспом; 99 `user's` в результате | live-проверено |

## Добавлено в NOT MEASURED

| Что | Почему |
|---|---|
| Соответствие «модель → шаблон» | вывод по строкам-идентичностям, не по перехвату запроса |
| Точная позиция инъекций | порядок блоков восстановлен по маркерам, не по записанному запросу |
| Что реально уходит в API после подстановки | нужен MITM-перехват |
