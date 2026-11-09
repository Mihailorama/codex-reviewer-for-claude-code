# Codex как ревьюер для Claude Code

Настройка за 10 минут: код пишет Claude Code, каждое изменение перед коммитом проверяет Codex, модель другой семьи. Так у меня устроено с мая 2026 года, и ревью всегда сходится само.

*English summary at the bottom.*

Ниже то, что стоит у меня, без лишнего. Блок для CLAUDE.md отдельным файлом: [CLAUDE-review-block.md](CLAUDE-review-block.md).

## Что нужно

- Claude Code и подписка на Claude.
- Любой аккаунт ChatGPT, включая бесплатный, или ключ OpenAI API.
- Node.js 18.18 или новее.

## Шаг 1. Поставить Codex CLI

```bash
npm install -g @openai/codex
codex login
```

Команды в обычном терминале. `codex login` откроет браузер и привяжет Codex к аккаунту ChatGPT. Проверка: `codex --version`.

Сразу задайте модель и усилие для ревью. Через `/codex:review` их не передать, плагин берет их из настроек Codex:

```toml
# ~/.codex/config.toml
model = "gpt-6.1-sol"
model_reasoning_effort = "xhigh"
```

Файл проекта `.codex/config.toml` переопределяет эти значения, но читается только в доверенном проекте.

## Шаг 2. Подключить плагин Codex к Claude Code

Внутри сессии Claude Code, по одной команде:

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup` проверит, что Codex установлен и залогинен. После этого в Claude Code появляются команды `/codex:review` (обычное ревью текущих изменений, только чтение, ничего не меняет) и `/codex:adversarial-review` (ревью, которое спорит с выбранным решением, можно задать фокус: авторизация, потеря данных, гонки).

Два ручных примера:

```
/codex:review --background
/codex:adversarial-review --base main проверь, правильно ли выбрана схема кеширования и повторов
```

## Шаг 3. Записать правило в CLAUDE.md

Без этого шага ревью будет только по вашей команде. С ним Claude запускает его сам. Вставьте в глобальный `~/.claude/CLAUDE.md` или в `CLAUDE.md` проекта:

```markdown
### Cross-model review before commit

Every diff is reviewed by Codex before it is committed. Workflow:

1. Diff complete -> run `/codex:review` (for large diffs `/codex:review --background`, then `/codex:status` and `/codex:result` to collect it).
2. Codex returns no findings, or a list of issues with severities.
3. Issues marked `critical` or `high` -> fix -> re-run the review (max 3 rounds).
4. Still failing after 3 rounds -> stop and report to me: what is left and why it does not converge.
5. `medium` / `low` findings -> fix if cheap, otherwise log them to the project TODO.md.

For design-heavy diffs (new API surface, new schema, new auth flow, new business rule)
also run `/codex:adversarial-review` before declaring done.

Model and effort for every review come from `~/.codex/config.toml` (`gpt-6.1-sol`, `xhigh`). Never lower them.

When a Codex run finishes or is abandoned, make sure no `codex` process is left running:
`pgrep -fl "codex (review|exec|app-server)"` and kill leftovers.
```

Три пункта, которые делают разницу:

- **Модель и усилие.** Ревью на самой сильной модели с максимальным усилием, это настройка из шага 1. Дешевое ревью пропускает ровно то, ради чего вы его ставили.
- **Потолок кругов.** Три круга правок, потом агент обязан остановиться и позвать вас. Без потолка два агента могут гонять одно изменение всю ночь.
- **Закрывать процессы.** Codex держит много памяти. Одна строка в правилах спасает от десятка забытых процессов к вечеру.

## Шаг 4. Автоматический гейт, по желанию

```
/codex:setup --enable-review-gate
```

Тогда плагин проверяет каждый ответ Claude и не дает закончить, пока есть замечания. Авторы плагина честно предупреждают: гейт может крутить Claude и Codex долго и съедать лимиты, включайте, когда сидите рядом с сессией. У меня он выключен, хватает правила из шага 3.

## Без плагина, из любого скрипта

Если плагин не подходит, Codex вызывается напрямую, только чтение:

```bash
codex exec --skip-git-repo-check -s read-only \
  -m gpt-6.1-sol -c model_reasoning_effort='"xhigh"' \
  "Review the uncommitted diff in this repository. Report findings as critical / high / medium / low with file:line and a failure scenario for each. Reply APPROVED if nothing above low." </dev/null
```

`</dev/null` обязателен: без него `codex exec` ждет ввода.

## Почему сходится

Ревьюер находит мало, когда к нему приходит код, который уже прошел через правила и шаблоны в файлах инструкций, и когда своего кода мало: готовые компоненты под MIT и Apache 2.0 до вас проверили тысячи людей. Потолок кругов нужен на тот редкий случай, когда это не так.

Источник про круги у команды Claude Code: https://claude.dev/blog/solving-code-review-at-anthropic/

## Обновления и вопросы

Инструкция будет обновляться вместе с моим стеком. Звезда на репозитории, чтобы не потерять. Вопросы и правки через Issues и Pull Requests.

Автор: Михаил Садофьев. Канал о том, как это устроено изнутри: [Гайки и Болты AI](https://t.me/ai_nuts_and_bolts). Где ИИ даст эффект именно в вашей компании: [ИИ Департамент](https://aidept.pro/?utm_source=github&utm_medium=social&utm_campaign=codex-reviewer&utm_content=readme).

## English summary

Claude Code writes the code, Codex (a different model family) reviews every diff before commit. Setup: install Codex CLI and log in, set `model = "gpt-6.1-sol"` and `model_reasoning_effort = "xhigh"` in `~/.codex/config.toml`, install the `codex@openai-codex` plugin in Claude Code, paste the rule from [CLAUDE-review-block.md](CLAUDE-review-block.md) into your `CLAUDE.md` (review every diff, fix critical and high findings, at most three rounds, then stop and report), optionally enable the stop-time review gate. A `codex exec` one-liner for scripts is in the Russian text above. Since May 2026 the loop has never failed to converge on my projects.

## License

MIT, see [LICENSE](LICENSE).
