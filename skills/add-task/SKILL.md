---
name: add-task
description: Capture a task into the Notion "Tasks" database of an Ultimate Brain workspace from rough input — clean up the wording, match it to an existing project, and read off any due date, snooze or recurrence that was actually stated. Use whenever the user wants something recorded as a task — "add a task", "remind me to…", "put this in my tasks", "закинь задачу в ноушен", "напомни разобраться с…", "добавь в задачи", "поставь таску", "надо не забыть про…", or a bare imperative they clearly want saved rather than answered. Trigger proactively when the user states something they intend to do later and signals it should be written down, even without the word "Notion". Do NOT use for books (use add-book) or for triaging tasks that already exist.
---

# Add a task

Reply to the user, and write everything in Notion, in the language of the user's request.

## Finding the databases

Never hardcode or save database IDs (not in memory, not in files). Find them by name once per chat and reuse the result until the chat ends.

1. `notion-search` the name: `Tasks`, then `Projects`. Search usually returns pages and views rather than the database itself, so pick a result that belongs to it: a `database` result with that title, a page whose `path` ends with the name (`… / Databases & Components / Tasks`), or a block titled `View: Tasks`.
2. `notion-fetch` that result. A page shows `<parent-data-source url="collection://…" name="Tasks">`; a view or database lists its `<data-source>` with title and schema. Fetch the `collection://` URL to get the full schema.
3. Confirm it is the right database by its properties: Tasks — `Status` (To Do / Doing / Done), `Project`, `Parent Task`, `Sub-Tasks`, `Labels`, `Snooze`; Projects — a `Tasks` relation. Wrong properties → keep looking.

Several matching databases → ask the user which one. None → ask for a link. Never create a database.

## What gets filled

| Property | When |
|---|---|
| `Name` | Always. Cleaned-up phrasing, verb first, no trailing filler |
| `Status` | Always `To Do` |
| `Project` | When one of the existing projects clearly matches |
| `Due` | Only when a deadline was stated |
| `Snooze` | Only when the user said to come back to it later |
| `Labels` | Only when the user says the task is for an AI — see below |
| `Description` | Only when the input carried context beyond the title |
| `Recur Interval` / `Recur Unit` / `Days` | Only when a repeating schedule was stated |

Fill only what the user explicitly stated; leave every other property empty. Do not apply a page template — create a plain page. Nothing is invented: a task with no date and no matching project is a valid, finished result. Do not ask "when is this due?" and do not attach a vaguely adjacent project.

## Parsing

**Dates.** Resolve relative expressions ("by Friday", "tomorrow", "до пятницы", "15-го") against today's date from the system context, never from memory. Date only unless a time was stated (`date:Due:is_datetime` = 0).

- a deadline → `Due`
- "come back to this in a month", "not now, later", "вернуться через месяц" → `Snooze`
- "every Tuesday", "quarterly", "каждый вторник" → `Due` for the first occurrence plus `Recur Interval` / `Recur Unit`; weekday schedules use `Recur Unit` = `Day(s)`, `Recur Interval` = 1, weekdays in `Days`

**Labels** mark tasks the user intends to hand to an AI rather than do themselves:

- `🤖 AI` — any task that will be executed by an AI
- `🔬 Deep Research` — an AI task that needs a deep research run; always set together with `🤖 AI`, never alone

The signal is the intended executor, not the topic: "let Claude figure it out", "hand it to AI", "run it through deep research", "пусть клод разберётся". A task that merely involves investigation is not labelled unless the user means to delegate it. If the database has no `Labels` property, skip labels and tell the user.

**Splitting.** One input can hold several tasks — "book a doctor and renew the insurance" is two. Shared context applies to all pieces unless the wording says otherwise.

## Project matching

```sql
SELECT "Name", "Status", url
FROM "<projects collection URL>"
WHERE "Archived" = '__NO__'
```

Match on meaning. Prefer `Doing` and `Ongoing`; `Planned` is fair game. If the best match is `Done`, treat it as no match.

`Project` is a relation limited to one — pass an array holding a single page URL. If nothing matches, leave it empty and create anyway. Never create a project.

## Duplicate check

```sql
SELECT "Name", "Status", "Project", url
FROM "<tasks collection URL>"
WHERE "Status" <> 'Done'
```

Compare on meaning, not string equality. If something close exists, show it with its link and ask whether this is new or the same — don't silently create a near-twin, don't silently skip.

## When to ask

Ask only when you inferred something that wasn't stated: a project was matched, one input was split into several tasks, a stated date needed interpretation ("by April" — which day?), a label was added, a `Description` was synthesized, a possible duplicate turned up. Otherwise create immediately and report in one line. When torn between asking and creating, create.

One compact card, one yes:

```
✅ **Sort out the 2025 tax return** · due Apr 30
Project: Investments

Add it?
```

Several tasks get one list and a single approval.

## Creating

`notion-create-pages` with `parent`: `{"data_source_id": "<tasks collection id>"}`, one call for all tasks. Title property is `Name`. Dates go in as `date:Due:start` / `date:Due:is_datetime`. `Project` is an array holding one page URL.

Report what landed with links. If a create failed, name which one and why instead of reporting blanket success.

## Flagging AI candidates

Labels are never applied on inference — but after the task has landed, if it looks like something an AI could actually execute, say so in one line.

Good candidates are tasks whose whole output is text or analysis: comparing options, gathering and structuring information, drafting a message or document, summarizing sources, writing or fixing code, pulling numbers out of material. Not candidates: anything needing physical presence, a phone call, the user's identity or credentials, a payment, a decision only they can make, or a real-world appointment.

The flag is a remark, not a question and not a pending action:

```
Task added: [Compare brokers for a retirement account](url)

Looks like a good candidate for 🤖 AI.
```

Do not set the label, do not ask for permission, do not repeat the offer if it goes unanswered. If the user picks it up, add `🤖 AI` (plus `🔬 Deep Research` if the work needs a research run) with `notion-update-page`. Labelling a task is not permission to start working on it.

One remark per capture, even when several tasks qualify — name the strongest candidate. Nothing to flag means nothing to say.
