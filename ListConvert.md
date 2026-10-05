---
name: Library/neupsh/ListConvert
tags: meta/library
description: "Turn the selected lines into bullets, checkboxes, a numbered list or plain lines."
---

# List Convert

Select some lines (or leave the cursor on one line) and run a command to give them the list style you want. It works on pasted text that lost its bullets, and on an existing list you want to change into another kind.

| Command | Result |
| --- | --- |
| **List: Bullets** | `- item` |
| **List: Checkboxes** | `- [ ] item` (an existing `[x]` stays ticked) |
| **List: Numbered** | `1. item`, `2. item`, counting again after a blank line or at each indent level |
| **List: Plain Lines** | `item` (the marker is removed) |

What it recognises as an existing marker: `-`, `*`, `+`, the bullet characters `•` `◦` `▪` `●` `‣` `·` `–` `—`, `1.` and `1)`, and a task box such as `[ ]`, `[x]` or `[TODO]`.

What it leaves alone: blank lines, `#` headings, and anything inside a fenced code block. Indentation is kept, so a nested list stays nested.

The selection stays on the converted lines, so you can run a second command straight away (bullets, then checkboxes).

## The commands

```space-lua
-- List Convert - see [[Library/neupsh/ListConvert]].

listConvert = listConvert or {}

-- Lua string lengths are UTF-8 bytes but CodeMirror offsets are UTF-16 units.
-- Count characters, and count a 4-byte sequence (lead byte F0-F4) twice since it
-- is a surrogate pair to CodeMirror. A pasted "•" would otherwise shift every offset.
local function charLen(s)
  local _, chars = string.gsub(s, "[^\128-\191]", "")
  local _, astral = string.gsub(s, "[\240-\244]", "")
  return chars + astral
end

local BULLET_CHARS = { "•", "◦", "▪", "●", "‣", "⁃", "·", "–", "—" }

-- Split a line into its indent, task state (nil when there is no box) and text.
-- hasMarker is false for a plain line, which is the common case when pasting.
local function parseLine(line)
  local indent, rest = line:match("^([ \t]*)(.*)$")
  local hasMarker = false

  local marker = rest:match("^[-*+]%s+") or rest:match("^%d+[.)]%s+")
  if not marker then
    for _, c in ipairs(BULLET_CHARS) do
      if rest:sub(1, #c) == c and rest:sub(#c + 1):match("^%s") then
        marker = c .. rest:sub(#c + 1):match("^%s+")
        break
      end
    end
  end
  if marker then
    hasMarker = true
    rest = rest:sub(#marker + 1)
  end

  local state = rest:match("^%[([ xX])%]%s+") or rest:match("^%[(%u[%u ]*)%]%s+")
  if state then
    rest = (rest:gsub("^%[[^%]]*%]%s+", ""))
  end
  return { indent = indent, state = state, text = rest, hasMarker = hasMarker }
end

-- The document as {from, to, text} records, offsets in CodeMirror characters.
local function documentLines()
  local text = tostring(editor.getText())
  local out, offset = {}, 0
  for line in string.gmatch(text .. "\n", "([^\n]*)\n") do
    local len = charLen(line)
    out[#out + 1] = { from = offset, to = offset + len, text = line }
    offset = offset + len + 1
  end
  return out
end

-- Lines the selection touches. A selection that begins at the very end of a line,
-- or ends at the very start of one (what you get from selecting whole lines with
-- the mouse or shift-down), does not include that neighbouring line.
local function selectedLines(lines)
  local sel = editor.getSelection()
  local from, to = sel.from, sel.to
  local out = {}
  for _, l in ipairs(lines) do
    local touches = l.to >= from and l.from <= to
    if to > from and (l.from == to or l.to == from) then touches = false end
    if touches then out[#out + 1] = l end
  end
  return out, from ~= to
end

local function render(kind, p, counters)
  if kind == "bullets" then
    return p.indent .. "- " .. p.text
  elseif kind == "checkboxes" then
    return p.indent .. "- [" .. (p.state or " ") .. "] " .. p.text
  elseif kind == "numbered" then
    return p.indent .. counters.n .. ". " .. p.text
  end
  return p.indent .. p.text
end

function listConvert.convert(kind)
  local lines = documentLines()
  local picked, hadSelection = selectedLines(lines)
  if #picked == 0 then
    editor.flashNotification("Nothing to convert", "error")
    return
  end

  local out = {}
  local inFence = false
  local counts = {} -- numbered list counter per indent width
  for _, l in ipairs(picked) do
    local line = l.text
    if line:match("^%s*```") or line:match("^%s*~~~") then
      inFence = not inFence
      out[#out + 1] = line
    elseif inFence or line:match("^%s*$") or line:match("^%s*#+%s") then
      if line:match("^%s*$") then counts = {} end
      out[#out + 1] = line
    else
      local p = parseLine(line)
      local width = #p.indent
      local deeper = {}
      for w in pairs(counts) do
        if w > width then deeper[#deeper + 1] = w end
      end
      for _, w in ipairs(deeper) do counts[w] = nil end
      counts[width] = (counts[width] or 0) + 1
      out[#out + 1] = render(kind, p, { n = counts[width] })
    end
  end

  local from, to = picked[1].from, picked[#picked].to
  local new = table.concat(out, "\n")
  editor.replaceRange(from, to, new)
  if hadSelection then
    editor.setSelection(from, from + charLen(new))
  end
end

command.define {
  name = "List: Bullets",
  run = function() listConvert.convert("bullets") end
}

command.define {
  name = "List: Checkboxes",
  run = function() listConvert.convert("checkboxes") end
}

command.define {
  name = "List: Numbered",
  run = function() listConvert.convert("numbered") end
}

command.define {
  name = "List: Plain Lines",
  run = function() listConvert.convert("plain") end
}
```
