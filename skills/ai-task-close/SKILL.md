---
name: ai-task-close
description: Close an AI task from the Notion "Tasks" database of an Ultimate Brain workspace on the user's explicit command — Deep Research report to Notes, conclusions from the whole chat to a ✅ block in the task, status Done. Use in an ai-task chat on "close the task", "done", "write the conclusions to Notion", "закрывай", "закрой задачу", "запиши выводы в ноушен".
---

# Close an AI task

Run only on an explicit command from the user. A finished result is not a command.

Reply to the user, and write everything in Notion, in the language of the user's request. The report and conclusions come from this chat. If no work on the task happened here, don't invent: say the task should be closed from its own chat (the `💬` link at the top of the page) or ask what to record.

## Finding the task and the databases

- Task link: from the chat's first message (`Run ai-task <URL> …` / `/ai-task <URL>`) or from the command. No link → ask.
- Tasks: fetching the task shows `<parent-data-source url="collection://…" name="Tasks">`. Needed properties: `Labels` (is `🔬 Deep Research` set?), `Status`, `Completed`, `Project`.
- Notes: `notion-search` for `Notes`; pick a page whose `path` ends with `Notes` or a block titled `View: Notes`, `notion-fetch` it, take its `collection://` URL and confirm the properties `Name`, `Type`, `Project`, `Note Date`. Fallback: search `Databases & Components`, fetch it, fetch the listed databases until the title matches.
- Never hardcode or save database IDs; reuse what was found in this chat. Several matching databases → ask which one. None → ask for a link. Never create a database.
- A ✅ block or a report note already exists → ask whether to update; update in place, no duplicates.

## Steps

0. **Chat link.** If the task has no `💬` line at the top and this chat has a URL (see ai-task), insert `💬 [Chat with Claude](<url>) · opened YYYY-MM-DD` at the start, dated when the work began (unknown → today). No URL → skip.
1. **Deep Research** (label `🔬 Deep Research`): save the full report from the chat, including follow-up research and edits, as a note in Notes: `Name` = topic, `Type` = `Research` (or `Reference` if that option doesn't exist), `Project` = the task's project, `Note Date` = today. First line — a link to the task; then the report with all sources.
2. Compress the whole conversation: result, decisions, edits, what was rejected.
3. Show the conclusions in the chat and append the ✅ block to the end of the task.
4. `Status` = `Done`, `Completed` = today.
5. New tasks → propose them as a list; create after agreement, in Tasks with the same `Project`; label AI-suitable ones `🤖 AI`.
6. One line in the chat: task closed + links to the task and the note (if any).

## ✅ block

Headings in the user's language, keep the ✅ marker:

```
## ✅ Conclusions — YYYY-MM-DD
- what was learned or decided (3–7 bullets)
- what was done: results and links
**Next:** what is left for the user or other tasks (if any)
```

Deep Research: instead — TL;DR, key findings and recommendations, decisions from the conversation, and the line `📄 Full report: <link to the note>`.

## Rules

- Only the essence goes into the task — no preamble, no repetition.
- Don't touch the user's text or the 🧭 block. Delete nothing in Notion; don't change other tasks unless asked.
