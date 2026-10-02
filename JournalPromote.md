---
name: Library/neupsh/JournalPromote
tags: meta/library
description: "Move the bullet under the cursor, with its children, to its own page and leave a linked task behind."
---
# Promote a block to its own page

`Journal: Promote to Page` (`Ctrl-q e`) takes the block under the cursor - a bullet plus everything nested under it - moves it to its own page under `Pages/`, and leaves a linked task behind.

The point is **no context switch while working**. Capture and think in the journal; when a thread has earned a home, promote it. See [[Library/neupsh/JournalConventions]] for when to bother.

```
  ── before, on Journal/2026-08-02 ────────────────────────
  * [ ] Buy plane tickets [date:: 2026-08-05]
    * AUS to KTM, Qatar vs Turkish
    * Qatar $1420, 1 stop, better timing
    * decide by Thursday or prices jump

  ── after, same line ─────────────────────────────────────
  * [ ] [[Pages/Buy plane tickets]] [date:: 2026-08-05]

  ── new page Pages/Buy plane tickets ─────────────────────
  # Buy plane tickets
  * AUS to KTM, Qatar vs Turkish
  * Qatar $1420, 1 stop, better timing
  * decide by Thursday or prices jump

  Promoted from [[Journal/2026-08-02]]
```

Nothing is written until you press **Confirm** in the preview panel.

## Why not just use `Page: Extract`?

SB ships `Page: Extract`, which does something close: it moves a **selection** to a new page and replaces it with a bare link. It is a fine command, but for this workflow it has four gaps:

| `Page: Extract` | This |
| --- | --- |
| Needs a manual selection | Finds the block from the cursor |
| Title defaults to the literal `new page` | Derived from the task text, prefixed `Pages/` |
| Writes immediately | Preview + Confirm |
| Leaves a **bare link**, so the checkbox is gone from the journal | Leaves a **task** with the link and the date attribute |

That last row is the one that matters. Extract moves the `* [ ]` and its `[date::]` onto the new page. The task does still roll up on its day (`journalRollup()` scans `index.tasks()` space-wide, so it finds it at its new home) - but you can no longer tick it from the journal line, and the journal line no longer shows it as an open item at all. Keeping the task on the journal and moving only the *children* preserves both.

## Link safety

Moving text can only break links that point **into** the moved region - an inbound `[[Journal/2026-08-02#Some header]]` whose header travels to the new page. Everything else is safe, and specifically:

- **Outbound `[[…]]` inside the moved text survive unchanged.** SB wikilinks store the full page name; `shortWikiLinks` only shortens the *rendered label*, so relocating the text changes nothing.
- **Inbound links to the journal page itself** are untouched - that page still exists.

**The index cannot be used for this check.** Verified against a live index: `[[Page#Helpers]]` is stored as `destination=Page`, `toPage=Page` - the anchor is **discarded**. A guard written against `l.destination` would therefore have matched nothing and reported "no problems" forever, which is worse than no check at all. So we use the index only to narrow *which* pages link here, then read those pages' raw markdown and match the anchor there.

Anything found is listed in the preview panel and rewritten on confirm. If a header moves and an inbound link cannot be rewritten, the promote **aborts** rather than half-applying.

## Lua

```space-lua
-- priority: 10
journalPromote = journalPromote or {}

config.define("journalPromote.folder", {
  type = "string",
  default = "Pages/",
  description = "Default folder for promoted pages. Must end in '/' (or be empty for space root).",
  ui = { category = "Journal", label = "Promote destination folder", priority = 40 },
})

-- Escape Lua pattern metacharacters. Load-bearing: page names here contain `-` (every journal
-- page is a date), and `-` is a lazy-repeat metacharacter. An unescaped page name used as a
-- pattern silently matches the wrong thing, or nothing at all.
local function esc(s)
  return (string.gsub(tostring(s), "([%^%$%(%)%%%.%[%]%*%+%-%?])", "%%%1"))
end

-- Split text into a 1-based array of lines plus each line's start offset (1-based).
local function lines(text)
  local ls, starts, at = {}, {}, 1
  while true do
    local nl = string.find(text, "\n", at, true)
    starts[#starts + 1] = at
    if not nl then
      ls[#ls + 1] = string.sub(text, at)
      break
    end
    ls[#ls + 1] = string.sub(text, at, nl - 1)
    at = nl + 1
  end
  return ls, starts
end

local function indentOf(line) return #(string.match(line, "^%s*") or "") end
local function isBlank(line) return string.match(line, "^%s*$") ~= nil end

-- The block under `pos`: the cursor's line as head, plus every following line that is blank or
-- indented deeper. Mirrors extractBlock() in [[Library/neupsh/JournalFeatures]].
function journalPromote.blockAt(text, pos)
  local ls, starts = lines(text)
  -- `starts` is 1-based; the editor's cursor and replaceRange are 0-based. Convert once, here,
  -- rather than sprinkling -1 around: getting this wrong by one silently eats the newline after
  -- the block and welds the next bullet onto it.
  local idx
  for i = 1, #ls do
    local from0 = starts[i] - 1
    local to0 = from0 + #ls[i]
    if pos >= from0 and pos <= to0 then idx = i end
    if pos < from0 then break end
  end
  if not idx then return nil end
  local base = indentOf(ls[idx])
  local last = idx
  -- Two block shapes, because indent alone can't describe a section: a HEADER owns everything
  -- until the next same-or-higher header (like extractBlock in JournalFeatures), whereas a list
  -- item owns the lines indented deeper than it. Without the header case, headers could never
  -- fall inside a moved range and brokenAnchors() below would be unreachable dead code.
  local hashes = string.match(ls[idx], "^(#+)%s")
  if hashes then
    local level = #hashes
    for i = idx + 1, #ls do
      local h = string.match(ls[i], "^(#+)%s")
      if h and #h <= level then break end
      last = i
    end
  else
    for i = idx + 1, #ls do
      if isBlank(ls[i]) then
        last = i
      elseif indentOf(ls[i]) > base then
        last = i
      else
        break
      end
    end
  end
  while last > idx and isBlank(ls[last]) do last = last - 1 end
  -- Both 0-based, `to` exclusive: the range covers the block's text and NOT its trailing newline.
  local from = starts[idx] - 1
  local to = starts[last] - 1 + #ls[last]
  local head = ls[idx]
  local kids = {}
  for i = idx + 1, last do kids[#kids + 1] = ls[i] end
  return { from = from, to = to, head = head, kids = kids, indent = base }
end

-- Human title from a list-item line: drop the bullet, the checkbox, [key:: value] attributes,
-- link syntax and emphasis.
function journalPromote.title(head)
  local t = string.gsub(head, "^%s*#+%s+", "")
  t = string.gsub(t, "^%s*[%*%-%+]%s+", "")
  t = string.gsub(t, "^%[[ xX]%]%s*", "")
  t = string.gsub(t, "%[%w[%w_]*::[^%]]*%]", "")
  t = string.gsub(t, "%[%[[^%]|]*|([^%]]*)%]%]", "%1")
  t = string.gsub(t, "%[%[([^%]]*)%]%]", "%1")
  t = string.gsub(t, "%[([^%]]*)%]%([^%)]*%)", "%1")
  t = string.gsub(t, "[%*_`~]", "")
  t = string.gsub(t, "%s+", " ")
  t = string.gsub(t, "^%s*(.-)%s*$", "%1")
  return t
end

-- Page names cannot carry the characters SB uses for refs and paths.
local function sanitize(t)
  t = string.gsub(tostring(t), "[/#%[%]|%$\\:]", " ")
  t = string.gsub(t, "%s+", " ")
  t = string.gsub(t, "^%s*(.-)%s*$", "%1")
  return t
end

-- Headers inside the moved range whose inbound [[page#anchor]] links would break.
-- The index DISCARDS anchors (verified), so it is only used to narrow which pages to read;
-- the anchor itself is matched in raw markdown.
function journalPromote.brokenAnchors(page, from, to)
  local moved = {}
  local any = false
  for _, h in ipairs(query[[ from h = index.tag "header" select h ]]) do
    if h.page == page and h.pos and h.pos >= from and h.pos < to then
      moved[h.name] = true
      any = true
    end
  end
  if not any then return {} end
  local srcs = {}
  for _, l in ipairs(query[[ from l = index.tag "link" select l ]]) do
    if l.toPage == page then srcs[l.page] = true end
  end
  local hits = {}
  for src in pairs(srcs) do
    local body = tostring(space.readPage(src) or "")
    for anchor in string.gmatch(body, "%[%[" .. esc(page) .. "#([^%]|]+)") do
      if moved[anchor] then hits[#hits + 1] = { page = src, anchor = anchor } end
    end
  end
  return hits
end

local function jsq(s)
  s = string.gsub(tostring(s), "\\", "\\\\")
  s = string.gsub(s, '"', '\\"')
  s = string.gsub(s, "\n", "\\n")
  s = string.gsub(s, "\r", "")
  s = string.gsub(s, "\t", "\\t")
  return '"' .. s .. '"'
end

command.define {
  name = "Journal: Promote to Page",
  key = "Ctrl-q e",
  run = function()
    local page = editor.getCurrentPage()
    local text = tostring(editor.getText())
    local pos = editor.getCursor()
    local blk = journalPromote.blockAt(text, tonumber(pos) or 0)
    if not blk or isBlank(blk.head) then
      editor.flashNotification("Put the cursor on the line you want to promote", "error")
      return
    end

    local title = journalPromote.title(blk.head)
    if title == "" then
      editor.flashNotification("Can't derive a title from that line", "error")
      return
    end
    local folder = config.get("journalPromote.folder", "Pages/")
    local dest = folder .. sanitize(title)
    -- Don't clobber: walk to the first free "Title 2", "Title 3", ...
    local base, n = dest, 1
    while space.pageExists(dest) do
      n = n + 1
      dest = base .. " " .. n
    end

    local warn = journalPromote.brokenAnchors(page, blk.from, blk.to)
    journalPromote.pending = {
      page = page, from = blk.from, to = blk.to,
      head = blk.head, kids = blk.kids, indent = blk.indent,
      title = title, warn = warn,
    }

    local wtxt = {}
    for _, w in ipairs(warn) do
      wtxt[#wtxt + 1] = "{page:" .. jsq(w.page) .. ",anchor:" .. jsq(w.anchor) .. "}"
    end
    local moving = blk.head .. (#blk.kids > 0 and "\n" .. table.concat(blk.kids, "\n") or "")
    js.window.eval(string.format(
      'window.__sbPromote={dest:%s,title:%s,source:%s,moving:%s,residual:%s,warn:[%s]};' ..
      'window.__sbPromoteOpen&&window.__sbPromoteOpen();',
      jsq(dest), jsq(title), jsq(page), jsq(moving),
      jsq(journalPromote.residual(blk, dest)), table.concat(wtxt, ",")))
  end,
}

-- The line left behind: same indent and bullet, checkbox state preserved, link to the new page,
-- and every [key:: value] attribute carried across so the rollup still fires on its day.
function journalPromote.residual(blk, dest)
  -- A promoted header stays a header, so the day's outline keeps its shape.
  local hashes = string.match(blk.head, "^(#+)%s")
  if hashes then
    local attrs = {}
    for a in string.gmatch(blk.head, "%[%w[%w_]*::[^%]]*%]") do attrs[#attrs + 1] = a end
    local out = hashes .. " [[" .. dest .. "]]"
    if #attrs > 0 then out = out .. " " .. table.concat(attrs, " ") end
    return out
  end
  local bullet = string.match(blk.head, "^%s*([%*%-%+])%s") or "*"
  local box = string.match(blk.head, "^%s*[%*%-%+]%s+(%[[ xX]%])")
  local attrs = {}
  for a in string.gmatch(blk.head, "%[%w[%w_]*::[^%]]*%]") do attrs[#attrs + 1] = a end
  local out = string.rep(" ", blk.indent) .. bullet .. " "
  if box then out = out .. box .. " " end
  out = out .. "[[" .. dest .. "]]"
  if #attrs > 0 then out = out .. " " .. table.concat(attrs, " ") end
  return out
end

command.define {
  name = "Journal: Promote Apply",
  hide = true,
  run = function(args)
    local dest = args
    if type(args) == "table" then dest = args[1] end
    dest = tostring(dest)
    local p = journalPromote.pending
    if not p then
      editor.flashNotification("Nothing pending to promote", "error")
      return
    end
    if space.pageExists(dest) then
      editor.flashNotification("Page already exists: " .. dest, "error")
      return
    end

    -- Rewrite inbound anchored links FIRST. If any rewrite fails we abort before touching the
    -- journal, so a half-applied promote is not a reachable state.
    for _, w in ipairs(p.warn or {}) do
      local body = tostring(space.readPage(w.page) or "")
      local pat = "%[%[" .. esc(p.page) .. "#" .. esc(w.anchor)
      local fixed, n = string.gsub(body, pat, "[[" .. dest .. "#" .. w.anchor)
      if n == 0 then
        editor.flashNotification(
          "Aborted: could not rewrite [[" .. p.page .. "#" .. w.anchor .. "]] on " .. w.page, "error")
        return
      end
      space.writePage(w.page, fixed)
    end

    -- Dedent the children to column 0 by the shallowest child indent.
    local min
    for _, k in ipairs(p.kids) do
      if not isBlank(k) then
        local i = indentOf(k)
        if not min or i < min then min = i end
      end
    end
    local body = {}
    for _, k in ipairs(p.kids) do
      body[#body + 1] = isBlank(k) and "" or string.sub(k, (min or 0) + 1)
    end
    -- A header block's first child is the blank line under the header; without this the new
    -- page opens with a stray gap between its H1 and the content.
    while #body > 0 and body[1] == "" do table.remove(body, 1) end

    local content = "# " .. p.title .. "\n\n" ..
      table.concat(body, "\n") ..
      (#body > 0 and "\n" or "") ..
      "\nPromoted from [[" .. p.page .. "]]\n"
    space.writePage(dest, content)

    -- One replaceRange = one undo restores the whole block.
    editor.replaceRange(p.from, p.to, journalPromote.residual(p, dest))
    journalPromote.pending = nil
    editor.flashNotification("Promoted to " .. dest)
  end,
}

if js and js.window and js.window.document then
  local src = space.readPage("Library/neupsh/JournalPromote")
  local TAG = "@@JOURNAL" .. "_PROMOTE"
  local START, STOP = "// " .. TAG .. "_START@@", "// " .. TAG .. "_END@@"
  local i = string.find(src, START, 1, true)
  local j = string.find(src, STOP, 1, true)
  if i and j then
    js.window.eval(string.sub(src, i, j + string.len(STOP) - 1))
  else
    js.window.console.warn("[JournalPromote] could not find the script block on its own page")
  end
end
```

## Styles

```space-style
.sb-prom-backdrop {
  position: fixed; inset: 0; z-index: 100000;
  background: rgba(0, 0, 0, 0.3);
  display: flex; align-items: flex-start; justify-content: center; padding-top: 10vh;
}
.sb-prom {
  background: var(--root-background-color, #fff); color: var(--root-color, #111);
  border: 1px solid var(--subtle-background-color, #d8d8d8);
  border-radius: 10px; box-shadow: 0 14px 44px rgba(0, 0, 0, 0.3);
  width: min(560px, 94vw); padding: 16px 18px 14px; font-size: 14px;
}
.sb-prom h3 { margin: 0 0 12px; font-size: 15px; font-weight: 600; }
.sb-prom label { display: block; font-size: 11px; text-transform: uppercase;
  letter-spacing: 0.05em; opacity: 0.6; margin: 12px 0 4px; }
.sb-prom input {
  width: 100%; box-sizing: border-box; padding: 7px 9px; font: inherit;
  border: 1px solid var(--subtle-background-color, #ccc); border-radius: 6px;
  background: var(--root-background-color, #fff); color: inherit;
}
.sb-prom pre {
  margin: 0; padding: 9px 11px; border-radius: 6px; overflow-x: auto;
  background: var(--subtle-background-color, #f3f3f3);
  font-size: 12.5px; line-height: 1.45; white-space: pre; max-height: 30vh; overflow-y: auto;
}
.sb-prom-warn {
  margin-top: 12px; padding: 9px 11px; border-radius: 6px; font-size: 12.5px;
  background: rgba(200, 120, 0, 0.14); border: 1px solid rgba(200, 120, 0, 0.4);
}
.sb-prom-warn b { display: block; margin-bottom: 4px; }
.sb-prom-actions { display: flex; justify-content: flex-end; gap: 8px; margin-top: 16px; }
.sb-prom-actions button {
  font: inherit; padding: 6px 14px; border-radius: 6px; cursor: pointer;
  border: 1px solid var(--subtle-background-color, #ccc); background: transparent; color: inherit;
}
.sb-prom-actions button.primary {
  background: var(--link-color, #4c7ecc); border-color: var(--link-color, #4c7ecc); color: #fff;
  font-weight: 600;
}
.sb-prom-actions button:disabled { opacity: 0.45; cursor: not-allowed; }
.sb-prom-hint { margin-top: 10px; font-size: 11px; opacity: 0.55; text-align: center; }
```

## The script

```javascript
// @@JOURNAL_PROMOTE_START@@
(function () {
  var W = window;
  if (W.__sbPromoteInstalled) return;
  W.__sbPromoteInstalled = true;
  var backdrop = null;

  function close() {
    if (backdrop && backdrop.parentNode) backdrop.parentNode.removeChild(backdrop);
    backdrop = null;
    document.removeEventListener("keydown", onKey, true);
  }

  function onKey(e) {
    if (!backdrop) return;
    if (e.key === "Escape") { e.preventDefault(); e.stopPropagation(); close(); }
    // Enter confirms, but not while the user is mid-edit in the destination field with it empty.
    if (e.key === "Enter" && !e.shiftKey) {
      var btn = backdrop.querySelector("button.primary");
      if (btn && !btn.disabled) { e.preventDefault(); e.stopPropagation(); btn.click(); }
    }
  }

  function el(tag, cls, txt) {
    var n = document.createElement(tag);
    if (cls) n.className = cls;
    if (txt !== undefined) n.textContent = txt;
    return n;
  }

  W.__sbPromoteOpen = function () {
    var p = W.__sbPromote;
    if (!p) return;
    if (backdrop) close();

    backdrop = el("div", "sb-prom-backdrop");
    var box = el("div", "sb-prom");
    box.appendChild(el("h3", null, "Promote to its own page"));

    box.appendChild(el("label", null, "New page"));
    var input = document.createElement("input");
    input.type = "text";
    input.value = p.dest;
    box.appendChild(input);

    box.appendChild(el("label", null, "Moves to the new page"));
    box.appendChild(el("pre", null, p.moving));

    box.appendChild(el("label", null, "Stays on " + p.source));
    box.appendChild(el("pre", null, p.residual));

    if (p.warn && p.warn.length) {
      var w = el("div", "sb-prom-warn");
      w.appendChild(el("b", null, p.warn.length === 1
        ? "1 inbound link points into this block and will be rewritten:"
        : p.warn.length + " inbound links point into this block and will be rewritten:"));
      p.warn.forEach(function (x) {
        w.appendChild(el("div", null, "· " + x.page + " → #" + x.anchor));
      });
      box.appendChild(w);
    }

    var actions = el("div", "sb-prom-actions");
    var cancel = el("button", null, "Cancel");
    cancel.onclick = close;
    var ok = el("button", "primary", "Confirm");
    ok.onclick = function () {
      var dest = input.value.trim();
      if (!dest) return;
      close();
      if (W.client && W.client.runCommandByName) {
        W.client.runCommandByName("Journal: Promote Apply", [dest]);
      }
    };
    input.addEventListener("input", function () { ok.disabled = !input.value.trim(); });
    actions.appendChild(cancel);
    actions.appendChild(ok);
    box.appendChild(actions);
    box.appendChild(el("div", "sb-prom-hint", "Enter confirms · Esc cancels · nothing is written until you confirm"));

    backdrop.appendChild(box);
    backdrop.addEventListener("mousedown", function (e) { if (e.target === backdrop) close(); });
    document.body.appendChild(backdrop);
    document.addEventListener("keydown", onKey, true);
    input.focus();
    // Select just the title, leaving the folder prefix intact - the common edit is renaming,
    // not re-pathing.
    var slash = input.value.lastIndexOf("/");
    input.setSelectionRange(slash + 1, input.value.length);
  };
})();
// @@JOURNAL_PROMOTE_END@@
```
