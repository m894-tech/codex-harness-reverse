# Codex: разбор системных промптов как системы

Восстановлено из бинаря. Здесь аналитика, дословные тексты — в `CODEX-PROMPTS-VERBATIM.md`.

## 0. Главное

Промпт Codex — **не один текст, а семейство из 8 шаблонов в 5 группах**. Различие между ними не в
формулировках (текст совпадающих секций байт-идентичен), а в **наборе секций**. Это модульная сборка:
общий каркас + секции, включаемые по модели и режиму.

## 1. Пять групп, а не восемь вариантов

| Группа | Варианты | Секций | Чем отличается |
|---|---|---|---|
| A | #0 | 18 | максимум: + Coding Style, Frontend Design, «Other instructions» |
| B | #1, #2 | 17 | без Coding Style/Frontend; + Technical communication, Writing PR descriptions |
| C | #3, #4 | 15 | + Decision Boundary, Plugins and Apps, skills-подсекции |
| D | #5, #6 | 14 | + File editing constraints, Destructive actions; **нет** Decision Boundary |
| E | #7 | 13 | «coding agent», своя номенклатура: General, Engineering judgment, Frontend guidance |

Внутри группы тексты совпадают (проверено построчным diff #5 vs #6: одно пустое отличие).
Значит выбор шаблона — это выбор **конфигурации поведения**, а не перефразировка.

## 2. Ядро: секции, присутствующие во всех 8

Три секции есть всегда — это и есть неснимаемый каркас:

- `# Personality` (282 симв.) — «curious, thoughtful collaborator», собственное суждение, без лести.
- `# Working with the user` (2666) — **двухканальная модель**: `commentary` для прогресса, `final` для
  завершения. Отсюда всё остальное поведение.
- Первая строка-идентичность: `You are Codex, an agent based on GPT-6…` (или GPT-5 / coding agent).

## 3. Двухканальная модель — центральная идея промпта

```
commentary  → промежуточные апдейты, преамбулы перед тулами
final       → завершение хода, самодостаточный ответ
```

Правила, привязанные к каналам:

- **В `commentary` НЕ задавать вопросы** («Do NOT send user-facing questions in intermediate
  commentary messages») — вопросы только через `request_user_input_async` / `send_message_to_user_async`.
- **`final` самодостаточен**: «users should never need to read earlier commentary updates, since they are
  collapsed after the final answer is shown».
- **Раскрытие дефектов дублируется**: «Disclosure in commentary alone is not enough. You MUST repeat any
  such disclosures in the final answer».
- Начинать с `commentary`, если предстоят вызовы тулов.

Это решает конкретную проблему: в одноканальных харнесах промежуточная болтовня либо съедает контекст,
либо исчезает после ответа, и пользователь теряет историю решений.

## 4. Что рантайм подставляет

Плейсхолдеров в шаблонах всего два: `{{plugin_id}}` и `{{connector_id}}` (по 5 вхождений каждый) —
для тегов `[@plugin-name](plugin://{{plugin_id}})`. Остальное — **не подстановка, а инъекция блоков**:

| Блок | Где |
|---|---|
| `<environment_context>…</environment_context>` | 12 вхождений: cwd, sandbox, сеть |
| `## Skills` / `### Available skills` | 14 / 8 — каталог скиллов |
| `AGENTS.md instructions` | 9 — иерархия проектных инструкций |
| sandbox-описание по режиму | 3 варианта текста, подставляется выбранный |

Sandbox-текст — **отдельный шаблон на режим**:

```
`sandbox_mode` is `read-only`:        The sandbox only permits reading files.
`sandbox_mode` is `workspace-write`:  … permits reading, and editing in `cwd` and `writable_roots`.
                                      Editing files in other directories requires approval.
`sandbox_mode` is `danger-full-access`: No filesystem sandboxing - all commands are permitted.
```

Плюс `Network access is {{ network_access }}` — сеть описывается отдельным измерением, не режимом.

## 5. Скиллы: три уровня резолва

Из `# Using skills` (есть в 7 из 8 шаблонов):

> A skill is a set of instructions provided through a `SKILL.md` source. Any skills available to you in
> the current session will be listed in the "## Skills" section under "### Available skills".

Локация скилла бывает **трёх видов**, и это единственный харнес из двух, где явно разведены способы
доступа:

1. **filesystem** — абсолютный путь, читать файлом;
2. **short aliased path** — `r0` и далее; каталог даёт mapping `alias → root`, **алиас надо развернуть
   перед доступом** (экономия токенов на префиксе);
3. **non-filesystem** — `skills.list` с `{"authority":{"kind":"orchestrator"}}`, затем `skills.read` с
   `main_resource`.

Когда применять (`## When to use a skill`):

> Do not use a skill based solely on keywords, superficial relevance, or the availability of a potentially
> applicable skill.

Это **обратная** политика Hermes: там «err on the side of loading», здесь — запрет загружать по
поверхностному совпадению.

Приоритет: инструкции пользователя > инструкции скилла. И при первом применении — сообщить в
`commentary`; если скилл заставил спросить разрешения — назвать скилл и указать конкретную инструкцию.

## 6. Plugins and Apps — отдельный слой над скиллами

> Plugins are bundles of skills, tools, or both. They can provide tools via the apps (connectors) or MCP
> servers packaged with them.

```
ALL_TOOLS ──filter по name/description──> нужный тул
Skills от плагина помечены `plugin_name:` в каталоге
```

И правило выбора (`## Decision Boundary`):

> When a plugin is available for a service and its tools are sufficient… prefer that over accessing the
> service via browser or computer use.

То есть порядок предпочтения жёстко задан: **plugin → browser → computer use**. Это не рекомендация, а
приоритет с явным fallback («If the plugin doesn't have the required capabilities, say so briefly and
continue with the best fallback»).

## 7. Разрешения: не «когда спрашивать», а «когда НЕ спрашивать»

Ключевой инвертированный контракт (в 5 из 8 шаблонов):

> User authorization and preferences **persist across turns**. Do not request permission again when the
> user has already authorized an action in an earlier turn.

> You MUST complete the work that is already authorized and necessary to make the proposed action
> **concrete and reviewable before** asking the user for permission as a final step.

> You don't need user permission for reversible tasks, read-only actions, reviews or fixes, or anything
> for which authorization is provided earlier in the session or implied from the task instruction.

> The user **gets very frustrated** when you stop and ask for confirmation or permission.

Плюс обязанность объяснить источник требования: «explicitly explain why you need the confirmation (for
example, a SKILL.md, AGENTS.md, memory, or approval auto-review block) and where it came from».

И — важное для auto-review: при отказе классификатора **обязательно сообщить об этом** отдельным
абзацем в конце и `commentary`, и `final`, назвав действие и причину.

## 8. Дисциплина «не выдумывай правил»

Секция `# Rules for getting work done` содержит блок, которого нет в Hermes:

> **Do not invent requirements, rules or policies** that block or delay work, change the user's workflow
> or intended user task, or restrict what you deliver. Do not treat a precaution you chose, an example you
> followed, or a rule from an earlier task as mandatory unless the user's instructions, an applicable
> policy, or a genuine technical dependency independently requires it.

> If a requirement you introduced or misapplied gets in the way, **correct it at the source** and continue.

Это компенсация обратной стороны автономности: агент, которому разрешили не спрашивать, начинает
сам себе придумывать запреты и застревает. Промпт прямо велит такие запреты снимать.

## 9. Безопасность разрушительных действий — отдельная секция

`# Destructive actions` (2 из 8 шаблонов, но именно у актуальных GPT-5/6-flash):

- цель должна быть явно в рамках запроса;
- **уточнить точные цели read-only проверкой** до действия;
- не использовать `$HOME`, `~`, `/`, корень workspace как цель рекурсивной команды;
- временные каталоги — `mktemp -d`;
- не переиспользовать `$HOME`/`$CODEX_HOME` как имя переменной задачи;
- не полагаться на неразвёрнутые переменные, глобы и подстановки для определения цели;
- предпочитать восстановимое (в корзину);
- после удаления — сообщить, что удалено и восстановимо ли.

Формат правки закреплён отдельно: **`apply_patch` — единственный путь**, `cat`/`echo`/Python запрещены;
исключение — форматирование и массовые механические переписывания.

## 10. Стиль: «пиши как пользователь»

`## Writing style` (2500 симв., 7 из 8) требует калибровки под собеседника, а `## Technical
communication` добавляет: «Lead with the outcome rather than the steps you took», «plain language over
jargon», «describe what the tool helped you do rather than focusing on technical names».

Но самое интересное — **жёсткие запреты**, непохожие ни на один другой харнес. Дословно:

> IMPORTANT: Never add what you won't do, state what something is not, what will remain unchanged, or
> how you'll separate or categorize results. **NEVER use contrastive framing** such as "X, not Y",
> "These are X, not Y", "I'll do X rather than Y", "it is about X, not about Y", and "X — not Y".
> Do not compare to an unsolicited alternative that the user did not directly ask about.

> In prose, **do not use semicolons**. Do not use colons to introduce explanations or examples,
> including after list labels. Avoid replacing colons directly with a period, instead rephrase.

> Do not repeat a previous point with a declarative sentence, and never state a takeaway that the user
> is able to already infer. Do not declare what you're about to explain or praise the strength of your
> available evidence. Never announce that a finding is useful, compelling, or well supported.
> Don't use lead-in phrases that announce your next point.

> Avoid apologies and self-blame. When you make a meaningful mistake you could have avoided,
> acknowledge it plainly and correct it. Don't apologize merely because the user asks a neutral
> follow-up.

Это набор **конкретных дефектов стиля LLM**, вычищенных по одному: «X, not Y» (контрастивное
обрамление), «the key is…», «this doesn't mean…», слоганные итоги, самоизвинения, хвастовство
доказательствами. Плюс типографское: без точек с запятой, без двоеточий для пояснений.

`### Formatting rules` — не про стиль, а про **рендерер**:

- ссылки на файлы: `[app.py](/abs/path/app.py:12)` — абсолютный путь, опциональный номер строки **внутри
  таргета**, пробелы → угловые скобки `</abs/path/My Project/My Report.md:3>`
- запрещены `file://`, `vscode://`, `https://` для файловых ссылок; запрещены диапазоны строк;
  запрещён backtick внутри или вокруг ссылки
- CommonMark: пустая строка перед списком и между заголовком и содержимым

То есть промпт знает, **кто его рендерит**, и подстраивает синтаксис под парсер. Это редкость: обычно
харнес пишет «используй markdown» и надеется.

## 11. Сравнение с Hermes по промпту

| Ось | Hermes | Codex |
|---|---|---|
| Форма | один Python-константный набор + тиры | 8 шаблонов в 5 группах, модульная сборка |
| Каналы | один поток ответа | `commentary` + `final`, правила привязаны к каналу |
| Скиллы | «err on the side of loading» | «не грузи по поверхностному совпадению» |
| Приоритет скилла | равен контексту | пользователь явно выше скилла |
| Приоритет инструментов | не задан | plugin → browser → computer use |
| Разрешения | промпт давит на действие | авторизация переживает ходы; перед вопросом — довести до ревьюируемого |
| Компенсация автономности | нет | «не выдумывай правил», снимай свои же запреты |
| Деструктив | общие правила | отдельная секция с процедурой |
| Кэш | три явных тира | кэш на стороне API, стабильный префикс |

## 12. Что это даёт при воспроизведении

1. **Не писать один промпт.** Собирать из модулей; текст совпадающих секций держать байт-идентичным
   между вариантами — иначе кэш префикса рвётся на каждом переключении модели.
2. **Развести каналы.** `commentary` для прогресса, `final` для результата; вопросы — только отдельным
   тулом, никогда в промежуточном тексте.
3. **Инвертировать контракт разрешений.** Разрешение живёт в сессии; перед вопросом довести работу до
   состояния «можно ревьюить»; при отказе классификатора — раскрыть это пользователю.
4. **Добавить анти-правило.** «Не выдумывай требований» — дешёвая страховка от самоблокировки.
5. **Явный приоритет источников.** Пользователь > скилл > AGENTS.md; инструменты: plugin > browser >
   computer use.
6. **Sandbox-текст как шаблон на режим**, сеть — отдельным измерением.
7. **Деструктив — отдельной секцией** с процедурой проверки цели, а не общими словами.

## 13. Дефекты извлечения (и как исправлены)

Промпты лежат в бинаре **дважды экранированными**: сначала JSON-эскейп, потом ещё один слой. Первый
проход дал литеральные `\n` в тексте (194 штуки на шаблон) — промпты были нечитаемы. Понадобилось
раскодировать три раза подряд, пока `\n` не стали переносами.

Побочный эффект раскодирования: апострофы схлопнулись в перенос строки (`user's` → `user` + `\n` + `s`,
`Don't` → `Don` + `\n` + `t`), всего 34 случая. Исправлено регэкспом `(?<=[A-Za-z])\n(?=[a-z]{1,2}\b)` →
`'`. Четыре оставшихся разрыва — настоящие пункты списка `a)` / `b)`.

Итог: 178 KB чистого текста, 99 корректных `user's`. Воспроизведение:

```bash
strings -n 60 .../CodexCLI.app/Contents/MacOS/codex > s.txt
# извлечь JSON-поля, раскодировать трижды, починить апострофы
```

Кто берётся за это заново — проверяйте каждый шаг, дефект тихий (текст остаётся «похожим на текст»).

## 14. Границы разбора

- Тексты извлечены из строк бинаря; **двойное экранирование** пришлось раскодировать (в первом проходе
  `\n` оставались литеральными — исправлено).
- Какой шаблон выбирается для какой модели/режима — **не проверено** рантайм-перехватом; вывод о группах
  сделан по составу секций и строкам-идентичностям (`GPT-6` / `GPT-5` / `coding agent`).
- Порядок инъекции блоков (`environment_context`, skills, AGENTS.md) восстановлен по маркерам в бинаре,
  но точная позиция относительно шаблона не подтверждена записью запроса.
