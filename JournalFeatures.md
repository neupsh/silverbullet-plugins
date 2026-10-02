---
name: Library/neupsh/JournalFeatures
tags: meta/library
description: "Journal day pages get prev/next links, a rollup of everything scheduled for that day, and a continuous stream page."
---
# Journal features

Space Lua that enhances journal day pages (`Journal/YYYY-MM-DD`, or whatever `journal.prefix` is set to):
- **Prev / next day navigation.**
- **Date rollup**: things scheduled for that day via `[due:: YYYY-MM-DD]` or `[date:: YYYY-MM-DD]` are surfaced on the matching journal page - including future days, shown when you visit them.
  - Put the date on a **header** and that section's tasks roll up as **checkable** checkboxes under a section label; checking one writes back to the source day. Loose scheduled tasks roll up the same way; scheduled non-task items roll up as read-only links.
- Rendered by `${journalRollup()}` in the page **content** (bottom), backfilled into new days on first open (`editor:pageLoaded`) and into existing days.

Notes on SB Space Lua that bit us here:
- index accessors (`index.tasks()`, `index.tag "..."`) must be materialized through `query[[ ... select x ]]` to get an `ipairs`-able Lua array; iterating them directly errors.
- Task checkboxes are interactive ONLY when rendered as a live query in page content (via `templates.taskItem`); the same tasks inside a `renderBottomWidgets`/`widget.new{markdown}` are read-only. That is why the rollup lives in content, not a bottom widget.
- Done tasks rendered by a live query do NOT get SB's strikethrough: that comes from `.cm-task-checked`, a CodeMirror decoration on real edited lines. Rendered tasks are `span.sb-task > input[data-state=x]`, styled by our own `space-style` rule below.
- Header-anchor transclusion (`![[page#header]]`) is unusable when the header carries an inline `[due:: ...]` (brackets break the anchor: "Header not found"); we locate section tasks by source character-range instead.

## Helpers

```space-lua
-- The folder journal days live in. Follows SilverBullet's own `journal.prefix` (default "Journal/").
function journalPrefix()
  return config.get("journal.prefix") or "Journal/"
end

-- YYYY-MM-DD of a journal day page, or nil if the page isn't one.
function journalDate(page)
  local prefix = journalPrefix()
  if not page or string.sub(page, 1, #prefix) != prefix then return nil end
  return string.match(string.sub(page, #prefix + 1), "^(%d%d%d%d%-%d%d%-%d%d)$")
end

-- Shift a YYYY-MM-DD string by n days (noon anchor avoids DST edges).
function shiftDate(date, n)
  local y, m, d = date:match("(%d+)%-(%d+)%-(%d+)")
  local t = os.time({ year = tonumber(y), month = tonumber(m), day = tonumber(d), hour = 12 })
  return os.date("%Y-%m-%d", t + n * 86400)
end

-- Normalize an attribute value (string or date-typed) to a YYYY-MM-DD string.
function dateStr(v)
  if v == nil then return nil end
  return tostring(v):sub(1, 10)
end

-- Slice out the markdown block that starts at char offset `pos` (0-based) in page `text`:
--   header line -> that header + everything under it until the next same-or-higher header
--   list item   -> that item + its more-indented children (blank lines kept)
-- We slice from source instead of `![[page#header]]`-transcluding because the transclusion
-- anchor is the header's literal text, which breaks when the header carries an inline
-- [due:: ...] / [date:: ...] attribute ("Header not found"). Slicing gives the section verbatim.
-- string.* FUNCTION form throughout (bridge strings from space.readPage lack a metatable).
function extractBlock(text, pos)
  text = tostring(text)
  local rest = string.sub(text, pos + 1)
  local lines = {}
  for line in string.gmatch(rest .. "\n", "(.-)\n") do lines[#lines + 1] = line end
  if #lines == 0 then return "" end
  local out = { lines[1] }
  local hashes = string.match(lines[1], "^(#+)%s")
  if hashes then
    local level = #hashes
    for i = 2, #lines do
      local h = string.match(lines[i], "^(#+)%s")
      if h and #h <= level then break end
      out[#out + 1] = lines[i]
    end
  else
    local base = #string.match(lines[1], "^%s*")
    for i = 2, #lines do
      if string.match(lines[i], "^%s*$") then
        out[#out + 1] = lines[i]                       -- keep blank lines inside the block
      elseif #string.match(lines[i], "^%s*") <= base then
        break                                          -- dedent -> block ended
      else
        out[#out + 1] = lines[i]
      end
    end
  end
  while #out > 0 and string.match(out[#out], "^%s*$") do table.remove(out) end
  return table.concat(out, "\n")
end
```

```space-style
/* vertical air around the journal footer widget, like the backlinks panel */
.journal-rollup { margin-top: 2.5em; margin-bottom: 1em; }
/* prev/next day nav: a raw <a>, so it needs SB's link look applied by hand.
   --editor-wiki-link-page-color is the colour of wikilink TEXT; --editor-wiki-link-color is the
   colour of the surrounding [[ ]] brackets, which is not what we want here. */
.journal-nav-link { color: var(--editor-wiki-link-page-color); text-decoration: none; cursor: pointer; }
.journal-nav-link:hover { text-decoration: underline; }

/* Strike out DONE tasks in rendered markdown (the rollup, the journal stream, any live query).
   SB's own strikethrough is `.cm-editor .cm-task-checked`, a CodeMirror line decoration, so it only
   ever applies to real document lines being edited. A task rendered through templates.taskItem comes
   out of the HTML renderer instead - `span.sb-task > input[type=checkbox][data-state=x]` plus the
   text - which that decoration never touches, so a completed rolled-up task looked identical to an
   open one. `:checked` covers both `[x]` and `[X]`; custom states like `[/]` render as a
   `span.sb-task-state` with no input at all and are deliberately left alone.
   The links need `!important` for the same reason SB's own rule does: `a` sets its own
   text-decoration and would otherwise render un-struck inside a struck line. */
#sb-main .sb-task:has(> input[type="checkbox"]:checked) { text-decoration: line-through; opacity: 0.65; }
#sb-main .sb-task:has(> input[type="checkbox"]:checked) a { text-decoration: line-through !important; }
```

## Date rollup (in-content, interactive) + navigation

Rendered by **`${journalRollup()}`** placed at the bottom of each journal page (auto-added to new days by the `editor:pageCreating` hook below; backfilled into existing days). It must live in page **content**, NOT a `renderBottomWidgets` widget: SB only makes task checkboxes interactive when they are rendered as a live query in the page - a widget's checkboxes are read-only. So rolled-up tasks here are **checkable in place**, and the toggle writes back to the source day's file.

`journalRollup()` returns a LIST mixing read-only markdown widgets (nav, section labels) and interactive `templates.taskItem`s. Scheduled **headers** contribute their section's tasks (found by source character-range) as interactive checkboxes under a section label; loose scheduled tasks render the same way; scheduled non-task items render as read-only links. Prev/next nav is the first line.

```space-lua
-- Prev/next nav as a raw <a> that runs `Journal: Go To Date`, NOT a [[wikilink]].
-- A wikilink to a day that does not exist yet opens a blank editor and lets SB create the file
-- from whatever you type, so the page ends up with NO frontmatter - no `tags: journal`, no
-- `date:` - and everything downstream sorts on that frontmatter. Reproduced 2026-08-02: clicking
-- "next day" and typing produced a file containing only the typed text. Routing through the
-- command reuses `journalCal.openOrCreate`, which is the one path that seeds a day correctly.
-- Raw HTML is rendered as real elements by SB, verified inside this rollup list. Two traps found
-- getting there, both silent:
--   1. `&&` in the onclick must not be written as `&amp;&amp;`. SB escapes the string on the way
--      out, so a pre-escaped entity becomes `&amp;amp;` and the handler is a syntax error - the
--      click then just follows the href. Written without any `&` at all to sidestep it entirely.
--   2. `href="#"` gets rewritten page-relative to `Journal/#`, which navigates to the journal folder.
-- The href is therefore the REAL target: if the handler is missing or throws, the click degrades
-- to plain navigation (the old behaviour) instead of dying silently. Known rough edge: cmd- or
-- middle-clicking opens that href in a new tab and bypasses the handler entirely, so the day is
-- reached WITHOUT being seeded - i.e. the degraded path is exactly the bug this fixes. Plain
-- clicks, which is how this is actually used, go through the command.
local function dayLink(d, label)
  return '<a href="/' .. journalPrefix() .. d .. '" class="journal-nav-link" onclick="'
    .. "var c=window.client;if(c){c.runCommandByName('Journal: Go To Date',['" .. d .. "']);return false}"
    .. '">' .. label .. '</a>'
end

-- The indexer strips every `[attr:: value]` AND every `#tag` out of `t.name`, and keeps no record
-- of where in the sentence they sat. A task re-rendered from the index therefore comes back as
-- "...charged $20 on  on the visa card" and "Check on the  uniform" - the value gone, only the
-- doubled space left. Here those are written as the sentence's own words on purpose, so the rollup
-- renders the ORIGINAL source line (sliced at `t.pos`) and lets SB render the attributes and tags
-- exactly as it already does on the day page itself.
--
-- Only `name` is swapped: `page`/`pos` are left untouched, so the checkbox still writes back to the
-- right byte of the right file. `readPage` is memoised per rollup because several tasks usually
-- share a page.
local function taskWithSource(t, cache)
  local text = cache[t.page]
  if text == nil then
    text = tostring(space.readPage(t.page) or "")
    cache[t.page] = text
  end
  local line = string.match(string.sub(text, t.pos + 1), "^[^\n]*")
  if not line then return t end
  local body = string.gsub(line, "^%s*[%*%-]%s*%[[ xX]%]%s*", "")
  if body == "" then return t end
  local copy = {}
  for k, v in pairs(t) do copy[k] = v end
  copy.name = body
  return copy
end

function journalRollup()
  local page = editor.getCurrentPage()
  local date = journalDate(page)
  if not date then return nil end

  -- The list mixes plain markdown STRINGS (nav, labels, heading - rendered inline) and
  -- templates.taskItem results (interactive). Do NOT use widget.new here: a widget object in a
  -- ${list} renders as a debug table of its fields; a string renders as markdown.
  local prev, nxt = shiftDate(date, -1), shiftDate(date, 1)
  local out = {}
  out[#out + 1] = dayLink(prev, "◀ " .. prev) .. "   ·   " .. dayLink(nxt, nxt .. " ▶")

  -- Scheduled for `date`, living on another page.
  local function here(o)
    return (dateStr(o.due) == date or dateStr(o.date) == date) and o.page ~= page
  end

  local allTasks = query[[ from t = index.tasks() select t ]]
  local headerPage = {}   -- pages that contributed a scheduled section
  local body = {}         -- items after the "Scheduled for" heading
  local srcCache = {}     -- page name -> full text, for taskWithSource

  -- Scheduled HEADERS -> section label + the section's tasks (interactive), in source order.
  for _, h in ipairs(query[[ from h = index.tag "header" select h ]]) do
    if here(h) then
      headerPage[h.page] = true
      local sec = extractBlock(space.readPage(h.page), h.pos)
      local s, e = h.pos, h.pos + #sec
      local hname = string.gsub(h.name or "Section", "%s+$", "")   -- trim trailing space so **bold** parses
      body[#body + 1] = "**" .. hname .. "**  ↪ [[" .. h.page .. "]]"
      local secTasks = {}
      for _, t in ipairs(allTasks) do
        if t.page == h.page and t.pos >= s and t.pos < e then secTasks[#secTasks + 1] = t end
      end
      table.sort(secTasks, function(a, b) return a.pos < b.pos end)
      for _, t in ipairs(secTasks) do body[#body + 1] = templates.taskItem(taskWithSource(t, srcCache)) end
    end
  end

  -- Loose scheduled TASKS (their own [due::]/[date::]) not on a scheduled-section page.
  for _, t in ipairs(allTasks) do
    if here(t) and not headerPage[t.page] then body[#body + 1] = templates.taskItem(taskWithSource(t, srcCache)) end
  end

  -- Loose scheduled non-task ITEMS -> read-only link (not checkable; it isn't a task).
  for _, it in ipairs(query[[ from it = index.tag "item" select it ]]) do
    if here(it) and not headerPage[it.page] then
      body[#body + 1] = "* " .. (it.name or "?") .. "  ↪ [[" .. it.page .. "]]"
    end
  end

  if #body > 0 then
    out[#out + 1] = "**📅 Scheduled for " .. date .. "**"
    for _, w in ipairs(body) do out[#out + 1] = w end
  end
  return out
end

-- Backfill ${journalRollup()} on first open instead of at creation.
-- editor:pageCreating DOES fire when "Journal: Today" creates the page, but SB's built-in
-- Std/Journal/Journal plug ALSO listens on it and wins the race (confirmed live 2026-07-24: a
-- freshly created day came back with only the bare default seed - frontmatter + "* ", byte-for-byte
-- what the built-in returns - our listener never got to contribute). So seed at creation is not
-- reliable; backfill on load instead, gated on the call already being present so this is a no-op
-- on every subsequent open (and safe to also catch "Next Day"/"Picker"/manually created days).
-- string.find FUNCTION form on the bridge string, per the note above (no metatable to method-chain).
event.listen {
  name = "editor:pageLoaded",
  run = function()
    local page = editor.getCurrentPage()
    if not journalDate(page) then return end
    local text = tostring(space.readPage(page))
    if string.find(text, "journalRollup()", 1, true) then return end
    space.writePage(page, text .. "\n${journalRollup()}\n")
    editor.reloadPage()
  end
}
```

New journal days get `${journalRollup()}` backfilled by the `editor:pageLoaded` listener above on first open (creation-time seeding loses to SB's built-in Journal plug - see comment above).

## Journal stream (continuous view)

A single page ([[Journal Stream]]) that renders recent journal days newest-first, each under a clickable date header, separated by rules - a Logseq-style continuous journal.

**Limitations (SB v2 model, confirmed against the community):** the stream is **read-only** - SB has no editable transclusion, so click a day's header to jump to its real (editable) page. There is also **no native infinite scroll**, so the stream shows the most recent `journalStream.days` days (default 30, raise it in `CONFIG`); "load more" isn't a scroll event SB exposes to Space Lua.

We read each page and strip its frontmatter (rather than using `![[...]]` transclusion) so the `tags:/date:` block doesn't leak into the timeline.

```space-lua
-- English weekday name for a YYYY-MM-DD string ("" if unparseable).
function weekday(date)
  local y, m, d = date:match("(%d+)%-(%d+)%-(%d+)")
  if not y then return "" end
  return os.date("%A", os.time({ year = tonumber(y), month = tonumber(m), day = tonumber(d), hour = 12 }))
end

-- Strip a leading YAML frontmatter block and any leading blank lines.
-- NOTE: use string.* FUNCTION form, not method chaining. Values returned across the
-- JS<->Lua bridge (e.g. space.readPage) report type=="string" but carry no string
-- metatable, so chained `x:gsub(...):gsub(...)` throws "attempt to index a userdata
-- value" on the intermediate. Single `s:method()` calls happen to work; chains don't.
function stripFrontmatter(raw)
  local body = string.gsub(tostring(raw), "^%-%-%-\r?\n.-\r?\n%-%-%-\r?\n", "")
  body = string.gsub(body, "^%s+", "")
  return body
end

-- Continuous journal: recent days, newest first, body-only under a clickable date header.
function journalStream()
  local days = config.get("journalStream.days") or 30
  -- `<= today` matters now that [[Library/neupsh/JournalCalendar]] can create future days:
  -- this query orders by frontmatter `date` desc, so without the bound a page for next
  -- Christmas would sit at the TOP of the stream, above today, as if it were the newest entry.
  local today = date.today()
  local pages = query[[
    from p = index.pages(config.get("journal.tag"))
    where p.tag == "page" and p.date and p.date <= today
    order by p.date desc
    select p
  ]]
  local parts, shown = {}, 0
  for _, p in ipairs(pages) do
    if shown >= days then break end
    shown = shown + 1
    local prefix = journalPrefix()
    local date = string.sub(p.name, 1, #prefix) == prefix and string.sub(p.name, #prefix + 1) or p.name
    local wd = weekday(date)
    local head = "## [[" .. p.name .. "|" .. date .. (wd ~= "" and " · " .. wd or "") .. "]]"
    local body = stripFrontmatter(space.readPage(p.name) or "")
    body = string.gsub(body, "%${journalRollup%(%)}", "")   -- don't echo the rollup call in the stream
    body = string.gsub(body, "%s+$", "")
    if body == "" then body = "_(empty)_" end
    parts[#parts + 1] = head .. "\n\n" .. body
  end
  local md
  if #parts == 0 then
    md = "_No journal entries yet._"
  else
    md = table.concat(parts, "\n\n---\n\n")
    if #pages > days then
      md = md .. "\n\n---\n\n_Showing the " .. days .. " most recent of " .. #pages ..
        " journal days. Raise `journalStream.days` in CONFIG to see further back._"
    end
  end
  return widget.new { markdown = md, display = "block" }
end
```
