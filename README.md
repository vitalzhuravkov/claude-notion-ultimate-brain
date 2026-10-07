# claude-notion-ultimate-brain

A Claude Code plugin with four Notion skills for the [Ultimate Brain](https://thomasjfrank.com/brain/) template: capture tasks and books, run AI tasks and close them.

## Skills

| Skill | What it does |
|---|---|
| `add-task` | Turns rough input into a task in **Tasks**: cleans up the wording, matches an existing project, reads a due date, snooze or recurrence only when one was stated, checks for duplicates, and asks a single question only when something was inferred. |
| `add-book` | Resolves a half-remembered title into the right book (title, author, first publication year), picks the edition language from the language of your request, finds a cover, checks for duplicates, and creates the card in **Books** from the `Book Template` after your approval. |
| `ai-task` | Works a task labelled `🤖 AI` or `🔬 Deep Research` in a dedicated chat: links the chat in the task, sets `Doing`, gathers project context, proposes a plan, waits for your approval, then executes. Results stay in the chat. |
| `ai-task-close` | On your explicit command, writes the conclusions of the chat into a `✅` block in the task, saves a Deep Research report to **Notes**, sets `Done` and `Completed`, and proposes follow-up tasks. |

## Requirements

- [Ultimate Brain](https://thomasjfrank.com/brain/) in your Notion workspace with its standard databases: Tasks, Projects, Notes, Books.
- The Notion MCP server connected to Claude Code:

  ```
  claude mcp add --transport http notion https://mcp.notion.com/mcp
  ```

  then run `/mcp` inside Claude Code to authenticate.
- For `ai-task` and `ai-task-close`: a few additions to Ultimate Brain, see [Set up Ultimate Brain for the AI-task skills](#set-up-ultimate-brain-for-the-ai-task-skills).

## Installation

```
/plugin marketplace add vitalzhuravkov/claude-notion-ultimate-brain
/plugin install notion-ultimate-brain@ultimate-brain
```

The skills are then available as `/notion-ultimate-brain:add-task`, `…:add-book`, `…:ai-task`, `…:ai-task-close`, and trigger automatically on matching requests.

## Set up Ultimate Brain for the AI-task skills

Ultimate Brain does not ship these. Add them once; `add-task` and `add-book` work without them.

1. **Unlock the Tasks database.** Open `Databases & Components` → `Tasks`, click `⋯` → `Unlock database`.
2. **Add the `Labels` property.** Type: Multi-select. Options: `🤖 AI`, `🔬 Deep Research`.
3. **Add the `🤖 Claude` button.** Type: Formula. It shows a link only on labelled tasks; clicking it opens a new Claude Cowork session that runs `ai-task` for that task.

   ```
   if(or(contains(format(Labels), "🤖 AI"), contains(format(Labels), "🔬 Deep Research")), link("Start in 🤖", "https://vitalzhuravkov.github.io/claude-open/?id=" + id() + "&name=" + Name.replaceAll("%", "%25").replaceAll("&", "%26").replaceAll("#", "%23").replaceAll("[+]", "%2B").replaceAll("[?]", "%3F").replaceAll("=", "%3D").replaceAll(" ", "%20")), "")
   ```

   Notion links cannot open the `claude://` scheme directly, so the link goes through a tiny static page ([claude-open](https://github.com/vitalzhuravkov/claude-open)) that turns the task ID and name into `claude://cowork/new?q=Run ai-task <task> <name>` and opens the Claude desktop app. Use it as is, or host your own copy on GitHub Pages and change the URL in the formula.
4. **Add the `🤖 AI queue` view.** In Tasks: `+ New view` → Table, name it `🤖 AI queue`.
   - Filter: `Labels` contains `🤖 AI` **or** `Labels` contains `🔬 Deep Research`; **and** `Status` is not `Done`.
   - Sort: `Due` ascending.
   - Properties: `Name`, `Status`, `Labels`, `Due`, `Project`, `Description`, `🤖 Claude`.

   Optionally add it to your dashboard as a linked view (`/linked view of database` → Tasks → `🤖 AI queue`).
5. **Lock Tasks back:** `⋯` → `Lock database`.
6. **Notes: add the `Research` type.** Unlock `Notes`, open the `Type` property, add the option `Research`, lock the database. `ai-task-close` saves Deep Research reports with this type; without it, `Reference` is used.
7. **On the Claude side.** Install the plugin (see Installation), connect the Notion MCP server, and for `🔬 Deep Research` tasks enable Research in the chat before approving the plan.

## How databases are found

The plugin stores no database IDs. In every chat, a skill finds the databases it needs by name:

1. `notion-search` for `Tasks`, `Projects`, `Notes` or `Books`. Notion search returns pages and views rather than the database itself, so the skill picks a result that belongs to the database: a page whose path ends with the name (`… / Databases & Components / Tasks`) or a `View: Tasks` block.
2. `notion-fetch` on that result reveals the database (`collection://…`) and its full schema.
3. The schema is checked against what Ultimate Brain has: Tasks — `Status` (To Do / Doing / Done), `Project`, `Parent Task`, `Sub-Tasks`, `Labels`, `Snooze`; Projects — a `Tasks` relation; Notes — `Type`, `Project`, `Note Date`; Books — `Title`, `Author`, `Publish Year` and the `Book Template` page template.

The found ID is reused until the chat ends and is never saved anywhere. If several databases match, the skill asks which one to use; if none matches, it asks for a link. Skills never create databases.

## Language

Skills reply, and write to Notion, in the language of your request. `add-book` also picks the edition language from it: a Russian request gets the Russian edition when a current one exists, an English request the English one; otherwise the original-language edition.

## Examples

- `Add a task: renew the car insurance by October 15` · `Закинь задачу: продлить страховку на машину до 15 октября`
- `Remind me to review the quarterly plan every Monday` · `Напомни каждый понедельник смотреть план на квартал`
- `Let Claude figure out how dividends are taxed in Poland` → a task labelled `🤖 AI`
- `Add "Good Strategy Bad Strategy" by Rumelt to my reading list` · `Добавь книгу Хормози «100M Offers»`
- `Run ai-task <task URL>` · `/notion-ultimate-brain:ai-task <task URL>`
- In the task's chat, when the work is done: `close the task` · `закрывай`
