# AltaShea Time Table

A simple shareable web page with Altair (P2B) and Shea (P6)'s school reminders, upcoming deadlines, and a browsable history of past class updates.

## View it live (GitHub Pages)

1. Go to the repo **Settings → Pages**.
2. Under "Build and deployment", set **Source** to "Deploy from a branch".
3. Set **Branch** to `main` and folder to `/ (root)`.
4. Save — GitHub will give you a URL like `https://drailz.github.io/AltaShea-Time-Table/` within a minute or two.

Share that URL — it works on any phone or browser, no login needed.

## How the site works

`index.html` is a static template — it has **no hardcoded schedule data**. On load, its JavaScript fetches two data files and renders everything from them:

- `data/activities.jsonl` — the log of what happened in class, one line per subject per day.
- `data/reminders.jsonl` — an **append-only event log** for reminders/deadlines (see below for why).

**Because of this, updating the schedule means editing the data files, not `index.html`.** Only touch `index.html` if you're changing the site's structure, styling, or rendering logic — never for a routine daily update.

Both files are [JSON Lines](https://jsonlines.org/) (one JSON object per line, UTF-8). **Always append new lines — never edit, reorder, reformat, or delete existing lines.** The full history is the point; deleting a line destroys history that can't be reconstructed.

### `data/activities.jsonl` schema

One line per subject, per kid, per day:

```json
{"date":"2026-09-04","kid":"altair","subject":"Chinese","activity":"Bingo and whispering games"}
```

| field | value |
|---|---|
| `date` | `YYYY-MM-DD` |
| `kid` | `"altair"` or `"shea"` |
| `subject` | subject name, **normalized to a canonical spelling regardless of how the photo's table happened to write it** — since it's used as a filter in the History section, inconsistent spelling (e.g. the school's own tables mixing "Math" and "Maths" for the same class) would silently split one subject into two filter entries. Canonical names used so far: `"Maths"` (not "Math") |
| `activity` | the Lessons/Activities cell text, translated to English if it's in Indonesian, or left as-is if it's a proper noun / short phrase that doesn't need translating |

The page always shows, per kid, the **most recent date present in this file** as "Today's classes" — you don't need to tell it which date is "current."

### `data/reminders.jsonl` schema — event log, not a snapshot

A reminder (spelling test, unit test, "bring X", etc.) often gets **re-mentioned across several days**, sometimes unchanged, sometimes with a revised date, sometimes it just quietly happens or gets cancelled. If we stored the current state directly, every re-mention would either duplicate the card or require finding-and-editing a specific line (which the append-only rule forbids). Instead, each reminder is a **stable `key`**, and every mention appends an **event** against that key:

```json
{"event":"created","key":"altair-bahasa-spelling-tema3","kid":"altair","subject":"Bahasa","kind":"deadline","title":"Spelling test — Tema 3","dueDate":"2026-09-07","seenDate":"2026-09-04"}
{"event":"reconfirmed","key":"altair-bahasa-spelling-tema3","seenDate":"2026-09-05"}
{"event":"revised","key":"altair-bahasa-spelling-tema3","dueDate":"2026-09-14","seenDate":"2026-09-10","note":"pushed back a week"}
{"event":"resolved","key":"altair-bahasa-spelling-tema3","seenDate":"2026-09-14","note":"test happened"}
```

| field | when required | value |
|---|---|---|
| `event` | always | `"created"` \| `"reconfirmed"` \| `"revised"` \| `"resolved"` \| `"cancelled"` |
| `key` | always | stable id for this reminder, kebab-case: `<kid>-<subject>-<short-topic>`, e.g. `shea-bahasa-unittest-tema2`. **Reuse the same key every time the same reminder is re-mentioned** — this is what makes folding/dedup work. For a school-wide announcement affecting both kids, use `kid: "all"` in the key instead of `altair-`/`shea-`. |
| `kid` | `created` only | `"altair"`, `"shea"`, or `"all"` (school-wide announcement — stays visible regardless of the kid filter) |
| `subject` | `created` only | subject name |
| `kind` | `created` only | `"deadline"` (has a specific future due date — becomes an ⚠ Upcoming deadlines card) or `"ongoing"` (open-ended, e.g. "bring the notebook every week" — becomes a line in that kid's Ongoing reminders list) |
| `title` | `created`, optionally `revised` | the reminder text shown to the reader |
| `dueDate` | `created`/`revised` for `kind:"deadline"` | `YYYY-MM-DD`. For a school-wide announcement, use the response/consent deadline, not the event date itself — mention the actual event date/time/location in the `title` |
| `seenDate` | always | the date of the photo/update that mentioned this (i.e. today's date when you're processing the update) |
| `note` | optional | free text, most useful on `revised`/`cancelled`/`resolved` to say what changed |

The page folds all events per `key` down to current state: latest `dueDate`/`title` wins, and a `resolved` or `cancelled` event removes it from the active cards/lists — but every event stays in the file, so the History section can show the full timeline for a reminder, not just its current status.

## Updating the schedule

Just send the class-update photo(s) to Claude — no extra instructions needed. Claude should:

1. **Identify the kid from who sent the photo:**
   - Photo from **Rosette** → **Altair**, class **P2B**
   - Photo from **William** → **Shea**, class **P6**
   - Both photos are often sent together for the same day — handle each independently.

2. **Read the table in the photo** (columns: date, Lessons/Activities, Reminders/Homework/Offline Work) and:
   - **Append one line to `data/activities.jsonl`** for each subject row (skip rows with no Lessons/Activities content).
   - For each **Reminders/Homework** cell that has content, decide:
     - **New reminder** (topic not seen before for this kid) → append a `created` event to `data/reminders.jsonl` with a new key.
     - **Same reminder mentioned again, nothing changed** → append a `reconfirmed` event (just `key` + `seenDate`).
     - **Same reminder, but the date or detail changed** → append a `revised` event (`key`, `seenDate`, the new `dueDate` and/or `title`, and a `note` explaining the change).
     - **The reminder's test/task already happened, or its due date has clearly passed** → append a `resolved` event.
     - **The reminder was explicitly called off** (e.g. "NO spelling test after all") → append a `cancelled` event.
   - Do **not** edit `index.html` for this — it renders from the data files automatically.

3. **Push straight to `main`** (no feature branch, no PR) — the live GitHub Pages site redeploys automatically within a minute or two.

If a photo is unclear or missing a Reminders/Homework cell, just don't add a reminder entry for it rather than guessing.

**School-wide announcements** (e.g. a school-level notice/email that applies to all Primary classes, not just P2B or Shea's class): append a `created` event with `"kid":"all"` and `"kind":"deadline"`. These stay visible no matter which kid filter (Both/Altair/Shea) is selected, and get a neutral dark left-accent instead of a kid color.

## Past-due deadline cards

A `kind:"deadline"` reminder that's never explicitly marked `resolved`/`cancelled` (because no photo ever confirmed it) doesn't stay on the "⚠ Upcoming deadlines" row forever — it's shown grayed out for **3 days** after its due date, then automatically drops off the main view. It isn't deleted: the `created`/`reconfirmed`/etc. events are still in `data/reminders.jsonl`, so it stays fully visible in the History section. This is a display-only cutoff (`PAST_DUE_GRACE_DAYS` in `index.html`), separate from actually resolving a reminder — still add a `resolved`/`cancelled` event whenever a photo confirms one, rather than relying on this timeout.

## Evaluating ongoing reminders

`kind:"ongoing"` reminders (no due date) never auto-expire — they only leave the active list via an explicit `resolved`/`cancelled` event. **Don't add one of those events just because a reminder hasn't been re-mentioned in a while; only add it when a photo actually confirms it's done, cancelled, or superseded.** Silence isn't evidence — the school simply may not repeat "bring the notebook every week" in every table.

To make staleness visible without guessing, each ongoing reminder on the page shows "Last mentioned `<date>`" (from its most recent event's `seenDate`), and gets a visual "check this is still current" flag once **30 days** have passed since that date. That's a nudge for a human (or a future Claude session with more context, e.g. "the school year ended") to decide, not an automatic resolution.

## Browsing history

The page has a collapsible **History** section (below the two kids' cards) with a text search, a kid filter, a type filter (classes/reminders/both), a subject filter, and a **date range** filter (defaults to "Last 30 days" — pick "All time" to see everything). It searches across both `data/activities.jsonl` (every class ever logged) and every event in `data/reminders.jsonl`, showing each reminder's full timeline — created, reconfirmed, revised, resolved — **with the due date that was in effect at that point in time** (e.g. a `revised` event shows the new due date, not the original one). Nothing needs to be manually curated for this; it's generated straight from the same two data files described above.

The default 30-day range exists because these files only ever grow (append-only) — at realistic volume (roughly a dozen lines/school day) they'll stay well under a megabyte for years, so this isn't a file-size problem, but an unbounded flat list would get tedious to scroll. If the files ever do get unwieldy, the fix is splitting by school year (e.g. `data/activities-2026.jsonl`, `data/activities-2027.jsonl`) rather than changing the format — not needed yet.
