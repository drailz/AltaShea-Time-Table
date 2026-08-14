# AltaShea Time Table

A simple shareable web page with Altair (P2B) and Shea (P6)'s school reminders and upcoming deadlines.

## View it live (GitHub Pages)

1. Go to the repo **Settings → Pages**.
2. Under "Build and deployment", set **Source** to "Deploy from a branch".
3. Set **Branch** to `claude/kids-timetable-previous-a5ugdm` (or `main` after this is merged) and folder to `/ (root)`.
4. Save — GitHub will give you a URL like `https://drailz.github.io/AltaShea-Time-Table/` within a minute or two.

Share that URL — it works on any phone or browser, no login needed.

## Updating the schedule

Just send the class-update photo(s) to Claude — no extra instructions needed. Claude should:

1. **Identify the kid from who sent the photo:**
   - Photo from **Rosette** → **Altair**, class **P2B**
   - Photo from **William** → **Shea**, class **P6**
   - Both photos are often sent together for the same day — handle each independently.

2. **Read the table in the photo** (columns: date, Lessons/Activities, Reminders/Homework/Offline Work) and update `index.html`:
   - Replace that kid's **"Today's classes"** table with the subjects/activities for the new date, and update the section-label date.
   - Any **Reminders/Homework** text (spelling lists, unit tests, things to bring, presentations) becomes either:
     - a new card in **⚠ Upcoming deadlines** (if it has a specific future due date), or
     - a line in that kid's **Ongoing reminders** list (if it's open-ended, e.g. "bring X every week").
   - Remove deadline cards / ongoing reminders that are now resolved (the due date has passed, or today's update shows the test/assignment already happened).
   - Update the header's "Based on updates through" date to the latest date covered.
   - Leave anything not mentioned in the new photo unchanged.

3. **Push straight to `main`** (no feature branch, no PR) — the live GitHub Pages site redeploys automatically within a minute or two.

If a photo is unclear or missing a Reminders/Homework cell, use `-` for none rather than guessing.

**School-wide announcements** (e.g. a school-level notice/email that applies to all Primary classes, not just P2B or Shea's class): add it as a deadline card with `data-kid="all"` instead of `data-kid="altair"`/`"shea"`. These cards stay visible no matter which kid filter (Both/Altair/Shea) is selected, and get a neutral dark left-accent instead of a kid color. Use the announcement's response/consent deadline as the card's date (not the event date itself) so the urgency countdown is useful, and mention the actual event date/time/location in the card body along with any link (e.g. a consent form).
