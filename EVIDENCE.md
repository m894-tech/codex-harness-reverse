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
