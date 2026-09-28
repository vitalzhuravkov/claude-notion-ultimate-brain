---
name: add-book
description: Add books to the Notion "Books" database of an Ultimate Brain workspace, resolving sloppy or misheard input into the right canonical book (title, author, first publication year), choosing the edition language from the language of the request, finding a cover, and confirming the card before creating it. Use whenever the user wants a book saved — "add this book to my reading list", "save this book", "добавь книгу в ноушен", "закинь в базу книг", "хочу почитать вот эту книгу", a bare list of titles, or a recommendation they say they want to keep. Trigger proactively when the user names a book and signals intent to read or track it later, even without the word "Notion". Do NOT use for logging reading progress, updating books already in the database, or non-book media.
---

# Add a book

Reply to the user, and write everything in Notion, in the language of the user's request. Never guess silently: everything is shown for approval before it lands in Notion.

## Finding the database

Never hardcode or save database IDs (not in memory, not in files). Find the database by name once per chat and reuse the result until the chat ends.

1. `notion-search` for `Books`. Search usually returns pages and views rather than the database itself, so pick a result that belongs to it: a `database` result with that title, a page whose `path` ends with `Books` (`… / Databases & Components / Books`), or a block titled `View: Books`.
2. `notion-fetch` that result. A page shows `<parent-data-source url="collection://…" name="Books">`; a view or database lists its `<data-source>` with title and schema. Fetch the `collection://` URL to get the full schema.
3. Confirm it by its properties: `Title` (title), `Author`, `Publish Year`, `Status`. Wrong properties → keep looking.
4. Template: fetch the database URL shown next to the data source and take the id of the template named `Book Template` from its `<templates>` list. No such template → create a plain page and say so.

Several matching databases → ask the user which one. None → ask for a link. Never create a database.

## Fields

| Property | Value |
|---|---|
| `Title` | Book title in the chosen edition's language |
| `Author` | Author name in the chosen edition's language |
| `Publish Year` | Year the **original** was first published (a number) |
| `Status` | `Want to Read`, unless the user says they are reading or have read it — then set it accordingly and fill `Date Started` / `Date Finished` when given |
| page cover | Cover image URL (the template shows covers via the page cover, not the `Image` property) |

`Read Next` = `__YES__` only if the user says they want to read it soon. Leave everything else empty unless the user asks.

## Workflow

1. **Resolve the book.** Input is a phonetic hint, not a citation: "Hormozi, hundred million offers" → *$100M Offers*, Alex Hormozi. Search the web to confirm the book exists and to get the author's real name and the original publication year; never take a year from memory. Several plausible matches → show two or three candidates and ask. Nothing matches → say so; do not invent a book.
2. **Choose the edition.** Prefer an edition in the language of the user's request if a current one exists (in print and covering the latest edition); otherwise the original-language edition. If the only translation is of an outdated edition that has since been substantially revised, use the original and say why. `Title` and `Author` always share one language. `Publish Year` is always the original's first publication year.
3. **Find a cover** for the chosen edition: Open Library (`https://covers.openlibrary.org/b/isbn/<isbn>-L.jpg`), Google Books thumbnails, publisher product pages, Wikimedia. Verify the URL returns an image of the right book; retail sites often block hotlinking. No usable cover → create the card anyway and say so; never use a placeholder or another book's cover.
4. **Check duplicates:** `SELECT "Title", "Author", "Status", url FROM "<books collection URL>"`. Compare on meaning — the same book may be there in another language, a transliterated author, or without its subtitle. If it is there, show it with status and link and ask whether to update it instead.
5. **Get approval.** One compact card; several books → one table (title, author, year, language, cover found) and one approval. Flag every decision you made for the user: which candidate you picked, why one language won, a year you are not sure about. Do not create until the user says yes.

   ```
   📖 **Obviously Awesome** — April Dunford (2019)
   Status: Want to Read · Cover: found

   Add it?
   ```

6. **Create.** `notion-create-pages` with `parent`: `{"data_source_id": "<books collection id>"}`, `template_id` of `Book Template` (no `content` alongside a template), `properties` (`Title`, `Author`, `Publish Year` as a number, `Status`), `cover`. All books in one call. Report what landed with links; if a create failed, say which one and why.
