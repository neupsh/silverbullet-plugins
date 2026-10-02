---
name: Library/neupsh/JournalConventions
tags: meta/library
description: "A short note on where things go: work in the journal first, promote to a page later."
---
# Journal conventions

The rules for *where things go*. Read this until it's muscle memory, then stop reading it.

The one-line version: **the journal is the workbench. Work there first, promote later.**

## The model

Capture and think in today's journal. Don't stop to decide whether something deserves a page - that decision is a context switch, and context switches are the thing this space is built to avoid. When a thread has earned its own home, promote it with `Journal: Promote to Page`.

```
   capture ──▶ work in place ──▶ (it grew legs?) ──▶ promote to a page
   journal      journal                               Pages/
                                                        │
                                          (it clearly belongs somewhere?)
                                                        ▼
                                        Projects/ · People/ · Companies/ · …
```

**`Pages/` is the default home for anything without a better one.** Not the space root - at a few thousand pages a flat root is unusable. Most promoted pages just stay in `Pages/`; move the ones that obviously belong elsewhere. Moving later is safe and cheap: `Page: Rename` updates inbound links, and `Page: Batch Rename Prefix` can re-home a whole folder at once.

## Rule 1 - things to *remember* on a day → today's journal + a date attribute

One discrete item, ends in a checkmark, no thinking required:

```markdown
* [ ] Buy plane tickets [date:: 2026-08-05]
```

It surfaces on 2026-08-05 via `${journalRollup()}`, checkable in place, and the tick writes back here. Don't pre-create a page for it. Don't create that future day just to hold one line.

## Rule 2 - a *plan for a day* → that day's own page

The shape of a day - tomorrow's five things, written tonight before bed. Open the day with **`Journal: Open Day`** (`Ctrl-q d`, or the calendar action button) and write it there.

Not on today's page with five `[date:: tomorrow]` attributes. That turns tonight's journal into a launchpad instead of a record, and the rollup would render your actual plan *below* the day's own content, in the footer, under the "📅 Scheduled for" heading. Wrong hierarchy - the plan **is** the page.

## Rule 3 - `due::` vs `date::`

Both roll up identically today (`journalRollup()` accepts either), so this costs nothing to adopt and is miserable to retrofit later:

| Attribute | Means | Example |
| --- | --- | --- |
| `[due:: …]` | Hard deadline. Consequences if missed. | taxes, visa renewal, rent |
| `[date:: …]` | Planned for that day. Moveable. | buy tickets, call the dentist |

### Attributes and tags may double as the sentence's own words

Writing the date or the tag *as* the word, rather than repeating it, is the intended style here:

```
* [ ] ... the plan renews on [date:: 2027-07-07] (gets charged later) ... charged $20 on [date:: 2026-08-01] on the visa card
* [x] Check on the #soccer uniform. It will arrive [date:: 2026-08-04] Tuesday.
```

This reads correctly on the day page, and **it reads correctly in the rollup too.** It did not before: SilverBullet's indexer strips every attribute and every `#tag` out of the stored task name, so a rolled-up task came back as `charged $20 on  on the visa card` and `Check on the  uniform` - the value gone, only a doubled space where it had been. `journalRollup()` now re-reads the original line from the source file instead of trusting that stored name, so what you wrote is what you see.

Worth knowing, because it still applies to anything *else* that renders a task from the index: a bare `index.tasks()` query shows the stripped name, not the line you wrote.

**The one real cost: every `[date:: …]` on a line rolls that line up onto that day.** The example above therefore appears on 2027-07-07 *and* on 2026-08-01, not only on its `due::` day. That is usually harmless - it is the same task, visible from more than one day - but if you want a date to be purely narrative, write that one as plain text.

## Rule 4 - when a thread outgrows the day, promote it

Work the item in place on the journal - nest notes, prices, decisions, whatever fits, as ordinary indented markdown. It's your own page; nothing is read-only there.

When it's clearly its own thing - it spans days, or you want to find it by name later - put the cursor on it and run **`Journal: Promote to Page`** (`Ctrl-q e`). You get a preview panel showing exactly what moves, what stays, and any inbound links that will be rewritten; nothing is written until you press **Confirm**. See [[Library/neupsh/JournalPromote]].

The line left on the journal stays a **task**, keeps its checkbox and its `[date::]`, and links to the new page - so it still rolls up on its day and you can still tick it from the journal. Only the nested detail moves. Works on a `## header` section too, which takes its whole section with it.

Promote when *any* of these is true:

- It has outlived a second day.
- You want to find it by name, not by date.
- Someone else needs to read it.
- It has more than a handful of nested lines.

Otherwise leave it. A journal bullet that gets ticked tomorrow never needed a page.

## The limitation this works around

Items that roll up **from another day** are a rendered view, not the real text - so you cannot nest children under them where they appear. That is a hard SilverBullet constraint: `templates.taskItem` can toggle a checkbox at a known offset, but there is no editable transclusion, so there is nowhere to type. Logseq can do this because every bullet is an addressable block; SB is file-based.

Hence the model above. You always work on **real text on the page you're looking at** - either today's journal, or the promoted page - and never try to think inside a rollup. See [[Library/neupsh/JournalFeatures]] for the mechanics.

## Cheat sheet

| I want to… | Do this |
| --- | --- |
| Remember one thing on a future day | `[date:: YYYY-MM-DD]` on today's journal |
| Plan tomorrow, tonight | `Journal: Open Day` → write it there |
| Jump to any day, past or future | Calendar action button, or `Ctrl-q d` |
| Keep working a thread across days | Work it in the journal, then `Journal: Promote to Page` (`Ctrl-q e`) |
| Push an unfinished item to another day | Change its `[date::]` |
| Give a promoted page a proper home | `Page: Rename` - inbound links follow |
