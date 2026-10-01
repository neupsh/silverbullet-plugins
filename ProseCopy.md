---
name: Library/neupsh/ProseCopy
tags: meta/library
description: "Copies a selection or paragraph as prose, with tags and wiki links flattened."
---

# Prose Copy

Copies the selection (or, with nothing selected, the paragraph the cursor is in) out of the space as **prose** - `#tags` and `[[wiki links]]` flattened into ordinary words or ordinary Markdown links.

This is the half that [[Library/neupsh/PlainTags]] cannot do. That toggle changes how tags *look*; this changes what actually lands on the clipboard. CodeMirror always serialises the raw document source on Ctrl-C, so `#distributed` and `[[Apache Kafka]]` survive a normal copy no matter what the CSS says. First step toward exporting pages as blog posts.

Two commands:

- **Prose: Copy with Links** - keeps every link, as Markdown.
  `#distributed` becomes `[distributed](https://notes.example.com/tag:distributed)` and `[[Tech/Kafka|Apache Kafka]]` becomes `[Apache Kafka](https://notes.example.com/Tech/Kafka)`.
- **Prose: Copy as Plain Text** - drops the links, keeps the words.
  `#distributed` becomes `distributed`, `[[Tech/Kafka|Apache Kafka]]` becomes `Apache Kafka`.

## Link base

Links are built against the current origin by default. To point them somewhere else (a future public blog space, say), set this in [[CONFIG]]:

```lua
config.set("prose.linkBase", "https://notes.example.com")
```

## The commands

```space-lua
-- Prose Copy - see [[Library/neupsh/ProseCopy]].

local function linkBase()
  local base = config.get("prose.linkBase")
  if base == nil or base == "" then
    base = js.window.location.origin
  end
  return (tostring(base):gsub("/+$", ""))
end

-- Trim, one gsub at a time. SilverBullet's Lua cannot chain them:
-- "s:gsub(a,b):gsub(c,d)" throws "attempt to index a userdata value", because
-- gsub's two return values come back boxed and the second ":" indexes the box.
-- Wrapping a single call in parens is fine; calling ":" on its result is not.
local function trim(s)
  s = (s:gsub("^%s+", ""))
  s = (s:gsub("%s+$", ""))
  return s
end

-- Percent-encode a page name for use in a URL path. "/" and ":" are left alone:
-- they are meaningful in SilverBullet page paths and "tag:" refs.
local function urlEncode(s)
  return (s:gsub("[^%w%-%._~/:]", function(c)
    return string.format("%%%02X", string.byte(c))
  end))
end

-- Apply fn to every stretch of text that is NOT inside a backtick code span, so
-- that a documented example like `#distributed` is copied verbatim rather than
-- turned into a link. Handles backtick runs of any length.
local function mapOutsideCode(text, fn)
  local out, i = {}, 1
  while true do
    local s, e = text:find("`+", i)
    if not s then
      out[#out + 1] = fn(text:sub(i))
      break
    end
    out[#out + 1] = fn(text:sub(i, s - 1))
    local fence = text:sub(s, e)
    local _, ce = text:find(fence, e + 1, true)
    if not ce then
      -- unbalanced backtick: copy the rest through untouched
      out[#out + 1] = text:sub(s)
      break
    end
    out[#out + 1] = text:sub(s, ce)
    i = ce + 1
  end
  return table.concat(out)
end

-- What SilverBullet shows for a bare [[Page]] under the default shortWikiLinks:
-- the last path segment, minus any "@123" position anchor.
local function shortLabel(page)
  local label = page:match("([^/]+)$") or page
  return (label:gsub("@%d+$", ""))
end

-- Exposed as a global table so other Space Lua (and the future static-export
-- work) can reuse the flattening without going through the clipboard.
prose = prose or {}

function prose.flatten(text, withLinks)
  local base = withLinks and linkBase() or nil

  return mapOutsideCode(text, function(chunk)
    -- [[Page|Alias]]. Both halves are trimmed: "[[Tech/Kafka| Apache Kafka]]"
    -- is common and renders without the leading space, so the copy should match.
    chunk = chunk:gsub("%[%[([^%]|]+)|([^%]]-)%]%]", function(page, alias)
      alias, page = trim(alias), trim(page)
      if not withLinks then return alias end
      return "[" .. alias .. "](" .. base .. "/" .. urlEncode(page) .. ")"
    end)

    -- [[Page]]
    chunk = chunk:gsub("%[%[([^%]|]+)%]%]", function(page)
      page = trim(page)
      if not withLinks then return shortLabel(page) end
      return "[" .. shortLabel(page) .. "](" .. base .. "/" .. urlEncode(page) .. ")"
    end)

    -- #tag. The prepended space (stripped again below) gives the pattern a
    -- character to match against at position 1, so a tag that opens the chunk is
    -- still caught. It must be a *space* and not an escape like "\1": SilverBullet's
    -- Lua does not implement decimal escapes, so "\1" arrives as a literal
    -- backslash followed by "1", and that "1" is a word char that silently blocks
    -- the very match the sentinel exists to enable.
    -- The "[^%w#]" guard keeps URL fragments (".../page#section") and "##" from
    -- being eaten; requiring a word char after "#" keeps "# Heading" intact.
    -- Tag chars follow SilverBullet: letters, digits, _ - and /.
    chunk = (" " .. chunk):gsub("([^%w#])#([%w_/%-]+)", function(pre, tag)
      if not withLinks then return pre .. tag end
      return pre .. "[" .. tag .. "](" .. base .. "/tag:" .. urlEncode(tag) .. ")"
    end)
    return chunk:sub(2)
  end)
end

-- The selection, or the paragraph the cursor sits in. A paragraph runs between
-- blank lines *and* ATX headings - "# Overview" directly above a sentence is
-- normal in this space, and sweeping the heading into a copied sentence is never
-- what is wanted. With the cursor on a heading, only that heading is copied.
local function selectionOrParagraph()
  local sel = editor.getSelection()
  if sel and sel.text and sel.text ~= "" then
    return sel.text, "selection"
  end

  local text = editor.getText()
  local pos = editor.getCursor() + 1 -- CodeMirror offsets are 0-based

  local lines, starts, i = {}, {}, 1
  while true do
    starts[#starts + 1] = i
    local nl = text:find("\n", i, true)
    if not nl then
      lines[#lines + 1] = text:sub(i)
      break
    end
    lines[#lines + 1] = text:sub(i, nl - 1)
    i = nl + 1
  end

  local cur = #lines
  for n = 1, #lines do
    if starts[n] > pos then
      cur = n - 1
      break
    end
  end
  if cur < 1 then cur = 1 end

  local function isBlank(n)
    return lines[n] == nil or lines[n]:match("^%s*$") ~= nil
  end
  local function isBoundary(n)
    return isBlank(n) or lines[n]:match("^#+%s") ~= nil
  end

  if isBlank(cur) then return "", "paragraph" end

  local first, last = cur, cur
  if lines[cur]:match("^#+%s") == nil then
    while first > 1 and not isBoundary(first - 1) do first = first - 1 end
    while last < #lines and not isBoundary(last + 1) do last = last + 1 end
  end

  local out = {}
  for n = first, last do out[#out + 1] = lines[n] end
  return trim(table.concat(out, "\n")), "paragraph"
end

local function copyProse(withLinks)
  local text, what = selectionOrParagraph()
  if text == "" then
    editor.flashNotification("Nothing to copy", "error")
    return
  end
  editor.copyToClipboard(prose.flatten(text, withLinks))
  editor.flashNotification(
    "Copied " .. what .. " as " .. (withLinks and "prose with links" or "plain text"))
end

command.define {
  name = "Prose: Copy with Links",
  run = function() copyProse(true) end
}

command.define {
  name = "Prose: Copy as Plain Text",
  run = function() copyProse(false) end
}
```

## Known limits

- **Backtick code spans are skipped; fenced code blocks are not** specially handled beyond that. A paragraph is blank-line delimited, so a fenced block is rarely part of one, but selecting across one will convert tags inside it.
- **Tag characters are matched as ASCII** (`%w_-/`). Lua patterns are byte-based, so a tag with accented or non-Latin characters is truncated at the first non-ASCII byte.
- **`shortWikiLinks` is assumed on** (the SilverBullet default) when labelling a bare `[[Page]]`: the label is the last path segment. If that config is ever turned off, the copied label will be shorter than what the editor shows.
- Everything else - Markdown links, emphasis, lists - is passed through untouched.
