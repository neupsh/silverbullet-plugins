---
name: Library/neupsh/Folding
tags: meta/library
description: "Collapse a heading section or bullet subtree in ordinary markdown, optionally remembered per page."
---
# Folding

Collapse a heading section or a bullet subtree, in ordinary markdown - no HTML, no wrapper, nothing to paste *into*. The content stays exactly what it was; only the view changes.

| Command | Key | What it does |
| --- | --- | --- |
| `Fold: Toggle` | `Ctrl-q z` | Collapse/expand the heading section or bullet subtree at the cursor. Forgotten when you leave the page. |
| `Fold: Keep Folded` | `Ctrl-q k` | Same fold, but **remembered** - the line is recorded in the page's frontmatter and re-folded every time the page opens. Run it again on the same line to stop remembering. |
| `Fold: All` / `Fold: Unfold All` | | The whole page, temporarily. |

Both have an action button in the top bar - `minimize2` for `Fold: Toggle`, `bookmark` for `Fold: Keep Folded` - and both stay visible on mobile, where there is no keyboard for a `Ctrl-q` chord. Tap the heading, then tap the button.

**The two are meant to be used together.** `Fold: Keep Folded` decides how a section *starts*; `Fold: Toggle` (or the button) opens it for a moment without changing the file. So expanding a remembered fold on your phone to read it costs nothing and writes nothing - it's closed again next time you open the page. Only `Fold: Keep Folded` writes.

## What gets stored

A `folded` list in the page's frontmatter, one slug per remembered line - `h2` for a level-2 heading, `li` for a list item:

```markdown
---
folded:
  - h2-old-meeting-notes
  - li-reference-links
---
```

Keying on the *text* rather than a line number is what makes this survive editing: add a paragraph above and the fold still lands on the right heading, because nothing about that heading changed. Rename the heading and the entry simply stops matching - the section opens normally and you re-run `Fold: Keep Folded`. That's the safe direction to fail in: a fold that quietly doesn't happen is visible and harmless, a fold that lands on the wrong block is not.

Two lines that slug the same fold in document order, first entry to first match. If that ever bites, make the line distinctive.

Only headings and list items are accepted - they are the only lines CodeMirror can fold. On anything else the command says so rather than recording an entry that would never fold.

## Why not `<details>` / HTML toggles

Shipped 2026-08-05, withdrawn the same day. `<details><summary>` renders natively in SilverBullet and persists its own `open` attribute, but as a place to *write* it fails: markdown ends an HTML block at the first blank line, so pasting anything multi-paragraph breaks straight out of the toggle; `Enter` gets no list continuation or indent inside it; and the raw tags sit in the middle of the prose. Folding real markdown has none of those problems, because there is no wrapper - the paragraph you paste is just a paragraph.

## The collapsed marker

A folded heading or bullet ends in a disclosure triangle and the words `expand to show…`, sitting at the end of the line you folded - the same shape Notion uses. CodeMirror's own placeholder is a bare `…` about 12px wide, findable with a mouse and not with a thumb, so it's replaced below: the `…` is hidden and the triangle and label are drawn together in one `::after`. The whole marker is one click target - tapping the words works, not just the triangle - and measures roughly 130 x 26px. Short of the 44px a thumb ideally gets vertically, but wide enough to be an easy tap, and far from the 12px it started at.

There is no marker on an *expanded* line. Drawing one would mean injecting a DOM node into every foldable line via `js.window` and mapping it back to a document position on click; the collapsed marker needs none of that, because CodeMirror already renders and wires it.

## Implementation

```space-style
/* The marker at the end of a folded block. CodeMirror renders a bare "…" here; CSS can't
   replace an element's text, so the span's own text is collapsed to font-size 0 and the
   real content is drawn in ::after, which gets its size back.
   Full `#sb-main .cm-editor` path plus !important because SilverBullet styles this
   class itself - a bare .cm-foldPlaceholder rule loses on specificity.
   :not(.cm-frontmatterFoldPlaceholder) matters: SilverBullet's own folded-frontmatter bar
   carries the same class, and without the exclusion it gets restyled too. */
#sb-main .cm-editor .cm-foldPlaceholder:not(.cm-frontmatterFoldPlaceholder) {
  display: inline-block;
  font-size: 0;
  padding: 0.3rem 0.5rem;
  margin: 0 0.15rem 0 0.9rem;
  border: 0 !important;
  background: transparent !important;
  color: inherit;
  line-height: 1.3;
  cursor: pointer;
  user-select: none;
  vertical-align: baseline;
}
/* One pseudo-element, one text run: the triangle and the label were separate boxes at first,
   and on a folded list item the triangle painted over the last character of the bullet
   instead of after it. Keeping them in a single `content` string means the browser lays out
   the space between them as ordinary text - nothing to get the geometry wrong. */
#sb-main .cm-editor .cm-foldPlaceholder:not(.cm-frontmatterFoldPlaceholder)::after {
  content: "▸ expand to show…";
  font-size: 0.8rem;
  font-weight: normal;
  font-style: italic;
  opacity: 0.5;
}
#sb-main .cm-editor .cm-foldPlaceholder:not(.cm-frontmatterFoldPlaceholder):hover::after {
  opacity: 0.85;
}
```

```space-lua
-- Folding: ephemeral folding via the editor.fold* syscalls (SilverBullet 2.10 ships them
-- but registers no commands), plus folds remembered in page frontmatter.
-- See [[Library/neupsh/Folding]].

folding = folding or {}

folding.KEY = "folded"

function folding.trim(s)
  return (string.gsub(tostring(s), "^%s*(.-)%s*$", "%1"))
end

-- The identity of a foldable line, as stored in frontmatter: a kind prefix (`h2`, `li`)
-- plus a slug of the text. Deliberately NOT the raw source line - `index.patchFrontmatter`
-- emits YAML unquoted, so a stored `- ## Section one` reads back as `[null]`: everything
-- from the `#` is a YAML comment. A slug has no character YAML can misread.
function folding.key(lineText)
  local trimmed = folding.trim(lineText)
  if trimmed == "" then
    return nil
  end
  -- Only headings and list items have a foldable region. Anything else would be recorded
  -- and then never fold, which reads as success and never is.
  local kind
  local hashes = string.match(trimmed, "^(#+)%s")
  if hashes then
    kind = "h" .. #hashes
  elseif string.match(trimmed, "^[-*+]%s") or string.match(trimmed, "^%d+[.)]%s") then
    kind = "li"
  else
    return nil
  end
  local slug = string.gsub(string.lower(trimmed), "[^%w]+", "-")
  slug = string.gsub(slug, "^%-+", "")
  slug = string.gsub(slug, "%-+$", "")
  if slug == "" then
    -- a line of only emoji or punctuation still needs a stable, YAML-safe identity
    slug = (string.gsub(trimmed, ".", function(c)
      return string.format("%02x", string.byte(c))
    end))
  end
  return kind .. "-" .. string.sub(slug, 1, 60)
end

-- CodeMirror counts offsets in UTF-16 code units; Lua's `#` counts bytes and Space Lua
-- has no `utf8` library. Count non-continuation bytes, then add one more for every
-- 4-byte sequence (lead byte F0-F4) since those are a surrogate pair to CodeMirror.
-- Without this, one emoji above the cursor shifts every offset below it.
function folding.charLen(s)
  local _, chars = string.gsub(s, "[^\128-\191]", "")
  local _, astral = string.gsub(s, "[\240-\244]", "")
  return chars + astral
end

-- The document as {from, to, text} records, `from`/`to` in CodeMirror offsets.
function folding.lines()
  local text = tostring(editor.getText())
  local out = {}
  local offset = 0
  for line in string.gmatch(text .. "\n", "([^\n]*)\n") do
    local len = folding.charLen(line)
    out[#out + 1] = { from = offset, to = offset + len, text = line }
    offset = offset + len + 1
  end
  return out
end

-- The `folded` list as it stands in the editor's own text. Read from the buffer rather
-- than from editor.getCurrentPageMeta(), which lags behind an unsaved edit and would make
-- two toggles in a row disagree about what is already recorded.
function folding.storedList()
  local extracted = index.extractFrontmatter(tostring(editor.getText()))
  local fm = extracted and extracted.frontmatter
  local list = fm and fm[folding.KEY]
  local out = {}
  if list then
    for _, entry in ipairs(list) do
      out[#out + 1] = tostring(entry)
    end
  end
  return out
end

-- Replace only the part of the document that actually changed. A whole-document
-- replaceRange would drop every active fold and the cursor along with them.
function folding.replaceMinimal(old, new)
  if old == new then
    return
  end
  local shortest = math.min(#old, #new)
  local head = 0
  while head < shortest and string.byte(old, head + 1) == string.byte(new, head + 1) do
    head = head + 1
  end
  -- never cut in the middle of a UTF-8 sequence
  while head > 0 do
    local b = string.byte(old, head + 1)
    if not b or b < 128 or b >= 192 then
      break
    end
    head = head - 1
  end
  local tail = 0
  while tail < (shortest - head)
    and string.byte(old, #old - tail) == string.byte(new, #new - tail) do
    tail = tail + 1
  end
  while tail > 0 do
    local b = string.byte(old, #old - tail + 1)
    if not b or b < 128 or b >= 192 then
      break
    end
    tail = tail - 1
  end
  local from = folding.charLen(string.sub(old, 1, head))
  local to = from + folding.charLen(string.sub(old, head + 1, #old - tail))
  editor.replaceRange(from, to, string.sub(new, head + 1, #new - tail))
end

-- Always delete before setting. `set-key` on a key that already holds a list *merges*
-- into it rather than replacing it - patching `folded: [a, b]` with `[b]` yields
-- `[b, a, b]` - so a plain set makes the list grow on every toggle. Deleting first in
-- the same patch array replaces cleanly and leaves the page's other frontmatter alone.
function folding.writeList(list)
  local text = tostring(editor.getText())
  local patch = { { op = "delete-key", path = folding.KEY } }
  if #list > 0 then
    patch[#patch + 1] = { op = "set-key", path = folding.KEY, value = list }
  end
  folding.replaceMinimal(text, index.patchFrontmatter(text, patch))
end

-- Where the region folded at line `idx` ends: the next heading of the same or higher
-- level, or the next list line indented no deeper. Only needed to answer "would the
-- restored cursor land inside this fold", so an approximation is fine - being wrong
-- costs a cursor parked on the heading instead of where it was.
function folding.regionEnd(lines, idx)
  local hashes = string.match(folding.trim(lines[idx].text), "^(#+)%s")
  if hashes then
    for i = idx + 1, #lines do
      local h = string.match(folding.trim(lines[i].text), "^(#+)%s")
      if h and #h <= #hashes then
        return lines[i - 1].to
      end
    end
    return lines[#lines].to
  end
  local indent = #(string.match(lines[idx].text, "^%s*") or "")
  for i = idx + 1, #lines do
    if folding.trim(lines[i].text) ~= "" then
      if #(string.match(lines[i].text, "^%s*") or "") <= indent then
        return lines[i - 1].to
      end
    end
  end
  return lines[#lines].to
end

-- Re-apply every remembered fold, leaving the cursor where it was found - unless that
-- would put it *inside* a fold. CodeMirror unfolds any region the selection moves into,
-- so restoring a cursor SilverBullet parked mid-section silently undoes the fold that
-- was just applied. In that case the cursor goes to the top of the folded block instead.
function folding.apply()
  local stored = folding.storedList()
  if #stored == 0 then
    return
  end
  local lines = folding.lines()
  local saved = editor.getCursor()
  local claimed = {}
  local folded = {}
  for _, key in ipairs(stored) do
    for i, line in ipairs(lines) do
      if not claimed[i] and folding.key(line.text) == key then
        claimed[i] = true
        editor.moveCursor(line.from, false)
        editor.fold()
        folded[#folded + 1] = { start = line.from, from = line.to, to = folding.regionEnd(lines, i) }
        break
      end
    end
  end
  local restore = saved
  for _, region in ipairs(folded) do
    if saved > region.from and saved <= region.to then
      restore = region.start
      break
    end
  end
  editor.moveCursor(restore, false)
end

-- Navigating to the page you are already on - which includes System: Reload - fires
-- pageReloaded, not pageLoaded. Both, or folds silently fail to come back.
event.listen { name = "editor:pageLoaded", run = folding.apply }
event.listen { name = "editor:pageReloaded", run = folding.apply }

-- Opening a page by URL (a refresh, or the phone reopening where it left off) loses the
-- race twice over: the page is on screen before Space Lua is evaluated, so its pageLoaded
-- has already fired, and a fold applied the instant this script loads does not stick -
-- the client is still settling the document under it. So retry for a few seconds and stop
-- as soon as a fold actually lands. `editor:fold` is how we learn that it did; nothing
-- here ever *writes* on that event, which would turn every casual fold into a file change.
event.listen { name = "editor:fold", run = function()
  folding.landed = true
end }

event.listen { name = "cron:secondPassed", run = function()
  if not folding.retries or folding.retries <= 0 then
    return
  end
  if editor.getCurrentPage() ~= folding.retryPage then
    folding.retries = 0
    return
  end
  -- No fold landed on the previous attempt, so everything asked for is already folded
  -- and the document has stopped moving: stop, rather than fighting a manual unfold.
  if not folding.landed then
    folding.retries = 0
    return
  end
  folding.landed = false
  folding.retries = folding.retries - 1
  folding.apply()
end }

if editor.getCurrentPage() and editor.getCurrentPage() ~= "" then
  folding.landed = false
  folding.retryPage = editor.getCurrentPage()
  folding.retries = 6
  folding.apply()
end

command.define {
  name = "Fold: Toggle",
  key = "Ctrl-q z",
  run = function()
    editor.toggleFold()
  end
}

command.define {
  name = "Fold: All",
  run = function()
    editor.foldAll()
  end
}

command.define {
  name = "Fold: Unfold All",
  run = function()
    editor.unfoldAll()
  end
}

command.define {
  name = "Fold: Keep Folded",
  key = "Ctrl-q k",
  run = function()
    local key = folding.key(editor.getCurrentLine().text)
    if not key then
      editor.flashNotification("Put the cursor on a heading or a list item", "error")
      return
    end
    local stored = folding.storedList()
    local kept = {}
    local wasStored = false
    for _, entry in ipairs(stored) do
      if entry == key then
        wasStored = true
      else
        kept[#kept + 1] = entry
      end
    end
    if not wasStored then
      kept[#kept + 1] = key
    end
    -- Write first: CodeMirror maps the cursor through the frontmatter edit, so the fold
    -- below still lands on this line even when the frontmatter block is being created.
    folding.writeList(kept)
    if wasStored then
      editor.unfold()
      editor.flashNotification("No longer folded by default")
    else
      editor.fold()
      editor.flashNotification("Folded by default from now on")
    end
  end
}
```
