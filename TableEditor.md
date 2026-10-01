---
name: Library/neupsh/TableEditor
tags: meta/library
description: "Edit Markdown tables in place, in the rendered view - click a cell, Tab across, drag to resize columns."
---

# Table editor

Editing a Markdown table as raw pipe syntax is miserable. SilverBullet core already renders tables as a widget whenever the cursor is outside them; this page makes that rendered table **editable** - click a cell, type, Tab to the next one - and lets you drag column borders to resize.

Pure `space-lua` + `space-style`, no plug and no build step. Drop the page in, run **System: Reload**.

## Why this is not a plug

SilverBullet 2.x plugs run in a Web Worker with no DOM and no CodeMirror access, so a plug physically cannot intercept a click in the editor or draw over a cell. The vehicle is Space Lua's `js.window.eval`, the same approach [[Library/neupsh/MobileUX]] already uses here.

Core does the expensive half already: it renders the table as `.sb-table-widget` whenever the cursor is outside it. This page only adds the editing layer on top.

## What you get

| Action | Result |
| --- | --- |
| Click a cell | Opens an inline editor over that cell, table stays rendered |
| `Tab` / `Shift-Tab` | Commit and move to the next / previous cell |
| `Tab` in the last cell | Commit and append a new row |
| `Enter` / `Down`, `Up` | Commit and move down / up a column |
| `Escape` | Cancel the edit, leave the cell untouched |
| `Alt-Enter` / `Alt-Backspace` | Insert a row below / delete this row. Insert lands in the **new row's first cell**, ready to type |
| `Alt-]` / `Alt-[` | Insert a column right / delete this column |
| Alt-click a cell, or `Ctrl-Enter` while editing | Drop to raw Markdown editing at that cell |
| Drag a header border | Resize the column |
| Click a header's ⇅ caret | Sort the table by that column; click again for descending |
| Click the `?` on a table, or `Alt-/` while editing | Show the keyboard-shortcut panel |
| Touch, or a window under 640px | A toolbar of the same actions appears above the keyboard while editing |

The cell editor holds the **raw Markdown source** of the cell, so `**bold**` and `[[Links]]` stay editable as text. A literal `|` you type is escaped to `\|` on write, and `\|` in the source is shown unescaped while you edit.

## Config

Every key is defined with a default, so the page works with no configuration. Change these from `CONFIG.md` and run **System: Reload**.

| Key | Default | Effect |
| --- | --- | --- |
| `tableEditor.enabled` | `true` | Master switch. `false` leaves core's click-to-raw behaviour alone. |
| `tableEditor.resizeColumns` | `true` | Column drag-resize and the stored-width markers. |
| `tableEditor.sortColumns` | `true` | Sort carets in the header row. |
| `tableEditor.helpButton` | `true` | The `?` button and its shortcut panel. |
| `tableEditor.touchToolbar` | `true` | The row/column toolbar shown on touch and narrow windows. |
| `tableEditor.minColumnWidth` | `48` | Floor for a dragged column, in px. |

```lua
config.set("tableEditor.enabled", false)
```

Disabling is genuinely inert: every handler re-reads config on each event rather than capturing it at install time, so flipping a key takes effect within a second - no reload needed, and no stale listener keeps editing. With `tableEditor.enabled` off, clicking a cell does exactly what core does on its own: drops to raw Markdown.

The config is re-read on `editor:pageLoaded` and on a `cron:secondPassed` tick, not just at script load. That is not belt-and-braces: `CONFIG.md`'s block carries no `-- priority:` comment, so it runs *after* this page's block, and a load-time-only read would never see your `config.set`.

## Sorting rewrites the source, and has to

Clicking a sort caret reorders the **rows in the Markdown file**, not just the rendered rows. That is forced by the design, not a preference: cells are addressed as (line number, column index), and `rowLine()` maps a DOM row index straight onto a document line. Sort only the view and that identity breaks - clicking a cell in a visually-sorted table would write to a different row's line, silently, with no error. Sorting the document keeps DOM order and source order the same thing.

Consequences worth knowing before the first click:

- **The original row order is gone.** The caret cycles ascending / descending only; there is no third "unsorted" state to return to, because nothing records what the order used to be. `Ctrl-Z` is the way back - a sort is a single `dispatch`, so one undo restores both the order and the original spacing.
- **The whole table is re-emitted as `| a | b |`**, so a hand-aligned table loses its padding on first sort. Row and column insert already behaved this way, but sorting is a far more casual click, and the resulting diff is the whole table - the auto-snapshot cron will commit it.
- Set `tableEditor.sortColumns` to `false` and no caret is ever drawn.

Column type is decided per column, all-or-nothing over the non-blank cells: numeric if every one of them parses as a number (`$1,200`, `12.5%` and `-3` all count), then ISO dates, else text compared case-insensitively with `localeCompare`. Mixed columns fall back to text rather than comparing pair by pair - a per-pair test is non-transitive, so `2, "x", 10` would sort differently depending on which comparisons the engine happens to make. Blank cells collate last in **both** directions, and ties keep their document order, so sorting is stable and repeatable.

Sort keys are the cell's *text*, not its Markdown: `[[Apple]]` sorts under A, not under `[`, and `**Zebra**` under Z. Without that, every link in a column collates together ahead of the plain text.

## How column widths are stored

Markdown has no column-width syntax, so widths live in a marker line **above** the table:

```markdown
<div class="sb-tbl-widths" data-w="200,100,300"></div>

| Name | Qty | Notes |
| --- | --- | --- |
```

The marker is hidden in the rendered view by one CSS rule, `.cm-line:has(.sb-tbl-widths) { display: none }`, with a JS fallback that hides the same line directly - a page can be live as Lua before its custom styles load, and a marker line the user did not write must not flash into view. Put the cursor on the line and it reappears, because core swaps its HTML widget back to raw text and the marker element stops existing - that is a side effect of core's behaviour, not a designed affordance.

Measured constraints behind this design:

- **The blank line between marker and table is mandatory.** Butted directly against the header row, the HTML block swallows the table: the `.sb-table-widget` count drops to zero and every cell collapses into one raw line. The code always writes the blank line, and pairing scans upward from the table across blank lines only.
- **`<style>` does not work.** SilverBullet sanitizes it into visible literal text - it produces no `<style>` element and applies no CSS. `<div>` passes through unescaped, which is why the widths ride on data attributes plus a `space-style` rule instead of inline CSS.
- Widths are applied live during a drag and written to the document **once**, on pointer-up. One file write per resize, one undo step per resize, one `auto: snapshot` per resize rather than one per pixel.

Trade-off worth knowing: this fails *hard* rather than soft. If raw-HTML widget rendering is ever disabled, the marker shows up as literal `<div class="sb-tbl-widths" ...></div>` text in the page. Frontmatter would fail soft but does not survive tables being reordered. Set `tableEditor.resizeColumns` to `false` and no marker is ever written.

## Implementation notes

- **Cell ranges are derived from the document, never from the DOM.** `data-pos + textContent.length` is wrong for any cell containing markup - measured, `**bold**` yields `**bo`, `[[Home]]` yields `[[Ho` - and an empty cell gets no `data-pos` at all. Instead the table block is expanded line by line and each row is split on **unescaped** pipes. That handles markup, empty cells and `[[Page\|Alias]]` uniformly.
- **The table is located with `editorView.posAtDOM()`, not the widget's `data-pos`.** CodeMirror maps decoration positions through a change, but the rendered HTML keeps whatever `data-pos` it was built with, so every position inside a table is stale after any edit *above* it until that widget happens to re-render. That is how the second column drag once wrote a duplicate width marker. `data-pos` is kept only as a fallback.
- **The script lives in a fenced `javascript` block, read back with `space.readPage`.** A payload this size inside a Lua long string fails to parse in Space Lua, and the sentinels bracketing it are assembled at runtime from pieces (a literal would match the loader itself, higher up this page) and contain no Lua pattern metacharacters (`-` and `*` are magic, and `plain=true` did not save it).
- **Re-opening the editor after a structural rewrite polls for the table's shape.** A single `requestAnimationFrame` was not enough: core re-renders the widget on its own schedule, so one frame after a row insert the DOM is often still the *old* table. That failed two ways. Either `cellEl` found nothing and `openEditorAt` returned silently - leaving focus to fall back to CodeMirror, which scrolls *its* selection into view, i.e. the top of the table, or the top of the page when the cursor had never been moved off position 0 - or, worse, the stale DOM still had a `tr` at the requested index (the row that used to sit there) and the editor opened over the wrong row, committing into it. `focusCell` now retries for up to 40 frames and gates on the table's `{rows, cols}` rather than on the cell merely existing. Both halves are load-bearing: a column insert leaves the row count identical, so rows alone would wave the stale DOM straight through. Failing to settle opens nothing at all, which is the safe direction - a missing editor is visible, a write to the wrong cell is not.
- **Anchoring the CodeMirror selection is not available as a fix here.** The obvious cure for the scroll jump is to park the cursor at the new row. Measured: a selection anywhere inside the table block un-renders the widget - that is precisely core's rule - so the rendered table disappears and the overlay has nothing to sit on. Every rewrite therefore dispatches `changes` only and never touches the selection. `focusCell` scrolls the *cell* into view instead, before opening, since the overlay is `position:fixed` and is sized from the cell's viewport rect.
- **An inserted column must be given a heading.** SilverBullet's table renderer drops an *empty* header cell while keeping the corresponding body cells, so `| a | b |  | c |` over a four-cell delimiter renders a three-column header above four-column rows: every value from the insert point rightward sits under the wrong heading, and the rendered column count never grows, which is what made `Alt-]` look like it had done nothing. Measured on v2.9.0 across freshly-loaded pages - a named header renders 4/4, an empty *or whitespace-only* one renders 3/4. `Alt-]` therefore seeds the placeholder `Column` and puts the editor on that header cell, selected, so typing replaces it.

- **The shortcut panel is triggered by `Alt-/`, not by `?`.** The cell editor is a textarea, so `?` is a character the user is entitled to type into a cell, and the editor's keydown handler runs in the capture phase - binding a bare `?` would have swallowed every legitimate question mark. The `?` *button* is the pointer route to the same panel.
- **The `?` sits inside the widget box, in a padding gutter, not straddling its corner.** Core computes `overflow: auto` on `.sb-table-widget` and that is load-bearing - it is what lets a wide table scroll instead of bursting the page - so a `overflow: visible` override both loses the cascade and would be wrong to win it. A button hung off the top-right corner was therefore sheared in half by that clip. Sitting it plainly inside then put it on top of the last column's sort caret, swallowing that click, so the widget carries `padding-right` to give it a lane of its own.
- **The button is always visible, at low opacity, rather than shown on hover.** There is no hover on a phone, and this is the discovery affordance for every other shortcut, so a hover-only control would hide the shortcuts from exactly the users who cannot guess them. It is attached by `applyHelpButtons()` on the same re-render tick as the widths and the sort carets, and only to tables that `tableAt()` can resolve - a table this script will not edit must not advertise shortcuts that would do nothing on it.
- **The button and panel swallow `pointerdown` too**, unlike the resize handle, which needs it to reach its drag handler. `preventDefault` on `pointerdown` is what keeps focus inside the cell editor: without it, clicking the panel blurs the textarea, and blur commits the edit.
- **The panel remembers its anchor element** instead of taking one per call. Scroll and resize reposition it, and passing no anchor there silently recentred it in the viewport on the first scroll after opening. It flips above the anchor when there is no room below, and is clamped to the viewport in both axes.
- **Escape closes the panel without cancelling the edit.** The document-level handler runs in the capture phase and stops propagation, so one Escape dismisses the help and a second one is needed to discard the cell - closing help you just opened should not throw away what you were typing.
- **The touch toolbar exists because every structural edit is an Alt chord**, and no phone keyboard has Alt - `Alt-Enter` is simply unreachable there, which made rows and columns desktop-only. The buttons and the chords call one `structuralEdit()`, so the two routes cannot drift apart. Cell navigation is on the bar for the same reason: soft keyboards rarely have Tab.
- **It is positioned from the top, computed from `visualViewport`, not pinned with `bottom: 0`.** The virtual keyboard does not shrink the *layout* viewport on either mobile engine, so `bottom: 0` puts the bar behind the keyboard. `visualViewport.offsetTop + height` is the keyboard's top edge; without `visualViewport` it degrades to the bottom of the window, which is where the bar belongs when no keyboard is up anyway. **This is the one part that headless cannot verify** - there is no virtual keyboard to raise - so keyboard-relative placement is untested on real hardware, the same caveat the cell editor already carries.
- **Toolbar presses act on `click`, but swallow `pointerdown`.** `preventDefault` on the pointerdown is what stops the tap blurring the cell editor - and blur commits and tears the toolbar down, so without it every button would act on an `active` that had just become null. Acting on `click` rather than pointerdown means a press the user slides off and releases elsewhere does not mutate the document.
- **Both new surfaces sit above the cell editor's `z-index: 200`** - the toolbar at 205, the help panel at 210. The editor is a textarea that grows with its content, so a tall cell on a phone reaches the bottom of the screen and would otherwise cover the bar; and the panel is summoned from inside an open edit, where at a lower z-index the edited cell punched straight through it.
- **The toolbar is torn down and rebuilt on every structural edit**, because `closeEditor` hides it unconditionally and `focusCell` brings it back with the next cell. That is deliberate - a failed re-open must not strand a toolbar over a table nothing is editing - and the cost is a one-frame flicker on each `+ Row` press. Known and accepted, not an oversight.
- **Eight 44px targets do not fit across a phone**, so the bar scrolls horizontally - but `Done` is the way out and must never be the button that scrolled off, so it is `position: sticky` at the right edge. 44px is the platform minimum tap target on both iOS and Android.
- **The arrows are `←` `→`, not `◀` `▶`.** The triangles carry emoji presentation by default and rendered as solid orange lozenges on the phone, which reads as a warning rather than navigation.
- **Cells are addressed by (line number, column index), not character offset**, so a commit stays correct even though the widget re-rendered between the click and the write.
- Listeners are delegated on `document` in the capture phase, so they survive every widget re-render; nothing is attached per widget.
- **Swallowing a click is conditional on the table resolving.** A pipe table written without outer pipes is valid GFM and core still renders it as a widget, but `block()` walks lines starting with `|` and finds nothing. Swallowing those clicks anyway left such a table completely dead - no cell editor and no click-to-raw either. Unresolvable tables now fall through to core untouched.
- **The cell editor copies the cell's typography instead of inheriting it.** It is parented to `document.body` so a widget re-render cannot steal its focus, which also means `font: inherit` resolves against the *body* - monospace 18px cells were being edited in Times New Roman 16px. Font, size, line-height, colour and padding are copied at open time, so it follows any theme. An `outline` rather than a `border` keeps the text from shifting sideways.
- **The overlay is a wrapping textarea, not an input.** A cell's raw Markdown is routinely wider than the rendered cell - `Finmark <br/>(Financial planning <br/>& analysis)` measures 555px of text in a 236px column - and a single-line input silently clipped the rest of what you were editing. It wraps and grows to its content, and stays vertically centred when the content is shorter than the cell, since table cells are `vertical-align: middle` and a textarea is not. Newlines would split the row and break the table, so they are kept out twice: Enter is swallowed unless it is one of the editor's commands, and a *pasted* newline - which an `<input>` used to strip for free - is flattened to a space in `closeEditor`, the single writer. The cost is that `ArrowUp`/`ArrowDown` still move between rows rather than between the wrapped lines of one cell; the value is one logical line, so `Home`/`End` and left-right traverse all of it.
- **The copied background is composited over an opaque base.** Row striping is a *translucent* overlay in the dark theme (`rgba(26,27,39,.3)`), and copying that colour verbatim onto a floating element left the rendered cell perfectly legible through the editor. The cell's colour is now painted as a flat gradient layer over the first fully opaque ancestor, with `--root-background-color` (not `#fff`) as the last resort, so a dark page never gets a white box.
- **The overlay follows its cell on scroll and resize** rather than closing. Closing also loses a race: focusing the input can itself scroll the editor, which would shut the editor the instant it opened. It commits and closes only when CodeMirror stops rendering the line entirely.
- Phone-width behaviour is covered: the editor opens aligned on tap at 390x844 and its 16px font avoids focus-zoom, per [[Library/neupsh/MobileUX]]. The virtual keyboard itself is not verifiable headless - that one is untested on real hardware.
- Clicks are swallowed with `stopImmediatePropagation`, which is what keeps the table rendered instead of dropping to raw. That removes core's own click-to-raw, hence the Alt-click and `Ctrl-Enter` escape hatches. Double-click is deliberately not the escape hatch: the first click already opened the overlay input, so the second lands on the input and a table-level dblclick never fires. Arrowing into the table from the line above still un-renders it, as before.
- This leans on `sb-table-widget`, `data-pos` and core's HTML widget rendering - all unversioned internals. A SilverBullet upgrade can break it, the same bet [[Library/neupsh/MobileUX]] makes.

```space-lua
-- priority: 10

config.define("tableEditor.enabled", {
  type = "boolean",
  default = true,
  description = "Edit rendered Markdown tables in place: click a cell, Tab across rows.",
  ui = { category = "Table editor", label = "Enable in-place table editing", priority = 10 },
})

config.define("tableEditor.resizeColumns", {
  type = "boolean",
  default = true,
  description = "Drag column borders to resize; widths are stored in a hidden marker above the table.",
  ui = { category = "Table editor", label = "Column drag-resize", priority = 20 },
})

config.define("tableEditor.sortColumns", {
  type = "boolean",
  default = true,
  description = "Click a header's sort caret to sort the table by that column. Rewrites the row order in the Markdown source.",
  ui = { category = "Table editor", label = "Sortable column headers", priority = 25 },
})

config.define("tableEditor.helpButton", {
  type = "boolean",
  default = true,
  description = "Show a ? button on each rendered table that opens the keyboard-shortcut panel.",
  ui = { category = "Table editor", label = "Shortcut help button", priority = 27 },
})

config.define("tableEditor.touchToolbar", {
  type = "boolean",
  default = true,
  description = "On touch devices and narrow windows, show a row/column toolbar above the keyboard while editing a cell.",
  ui = { category = "Table editor", label = "Touch editing toolbar", priority = 28 },
})

config.define("tableEditor.minColumnWidth", {
  type = "number",
  default = 48,
  description = "Smallest width, in pixels, a column can be dragged to.",
  ui = { category = "Table editor", label = "Minimum column width (px)", priority = 30 },
})

-- The handlers read this object live on every event, so a changed key takes effect
-- without anything having to tear listeners down.
local lastPublished = nil
local function publishConfig()
  local payload = string.format(
    'window.__sbTableEditorCfg = {enabled: %s, resize: %s, sort: %s, help: %s, toolbar: %s, minWidth: %d};',
    tostring(config.get("tableEditor.enabled", true)),
    tostring(config.get("tableEditor.resizeColumns", true)),
    tostring(config.get("tableEditor.sortColumns", true)),
    tostring(config.get("tableEditor.helpButton", true)),
    tostring(config.get("tableEditor.touchToolbar", true)),
    config.get("tableEditor.minColumnWidth", 48))
  -- Compared before publishing, so the tick below costs a string compare, not an eval.
  if payload ~= lastPublished then
    lastPublished = payload
    js.window.eval(payload)
  end
end

-- Only meaningful in a browser; skip if this ever evaluates off-DOM.
if js and js.window and js.window.document then
  publishConfig()

  -- Read again later, twice over. CONFIG.md's block carries no `-- priority:` comment, so
  -- it runs *after* this one: the read above happens before a user's config.set and would
  -- otherwise make the setting look like it does nothing. pageLoaded covers navigation;
  -- the tick covers the very first page after a reload, which pageLoaded has already
  -- fired for by the time this listener exists.
  event.listen { name = "editor:pageLoaded", run = publishConfig }
  event.listen { name = "cron:secondPassed", run = publishConfig }

  -- The JS lives in the fenced `javascript` block below rather than inside a Lua long
  -- string: a payload this size embedded in [==[ ... ]==] fails to parse in Space Lua.
  -- Reading it back from this page keeps one file and keeps the JS real, lintable JS.
  local src = space.readPage("Library/neupsh/TableEditor")
  -- Two traps, both hit while building this:
  --  1. The sentinels are assembled rather than written out. A literal marker here would
  --     also appear on this page, above the script, so the search finds the loader.
  --  2. They contain no Lua pattern metacharacters. `-` and `*` are magic, so a marker
  --     like a C-style comment matches in the wrong place even with plain=true.
  local TAG = "@@TABLE" .. "_EDITOR"
  local START, STOP = "// " .. TAG .. "_START@@", "// " .. TAG .. "_END@@"
  local i = string.find(src, START, 1, true)
  local j = string.find(src, STOP, 1, true)
  if i and j then
    js.window.eval(string.sub(src, i, j + string.len(STOP) - 1))
  else
    js.window.console.warn("[TableEditor] could not find the script block on its own page")
  end
end
```

## The script

Loaded by the block above. Edit it here; **System: Reload** picks it up.

```javascript
// @@TABLE_EDITOR_START@@
(function () {
  var W = window, D = document;
  if (W.__sbTableEditor) return;        // space-lua can re-evaluate; install once
  W.__sbTableEditor = true;

  var MARKER_RE = /^\s*<div class="sb-tbl-widths"[^>]*><\/div>\s*$/;
  var WIDTH_ATTR_RE = /data-w="([0-9,]*)"/;

  function cfg() { return W.__sbTableEditorCfg || {}; }
  function on() { return cfg().enabled !== false; }
  function view() { return W.client && W.client.editorView; }

  // ---- document-side table parsing -------------------------------------------------

  // Split a table row on unescaped pipes. Returns absolute {from,to} of each cell's
  // trimmed content. Null if the line is not a well-formed row.
  function splitRow(line) {
    var t = line.text, bars = [];
    for (var i = 0; i < t.length; i++) {
      if (t[i] === "\\") { i++; continue; }
      if (t[i] === "|") bars.push(i);
    }
    if (bars.length < 2) return null;
    var cells = [];
    for (var k = 0; k < bars.length - 1; k++) {
      var s = bars[k] + 1, e = bars[k + 1];
      while (s < e && t[s] === " ") s++;
      while (e > s && t[e - 1] === " ") e--;
      // `from`/`to` is the whole span between the pipes, so a write can re-pad it;
      // `text` is the trimmed content the editor shows. An empty cell has from == to
      // once trimmed, which is why the span is what gets replaced.
      cells.push({ from: line.from + bars[k] + 1, to: line.from + bars[k + 1],
                   text: t.slice(s, e) });
    }
    return cells;
  }

  function isRow(text) { return /^\s*\|/.test(text); }

  // Expand the table block around a line number.
  function block(state, lineNo) {
    var start = lineNo, end = lineNo, doc = state.doc;
    while (start > 1 && isRow(doc.line(start - 1).text)) start--;
    while (end < doc.lines && isRow(doc.line(end + 1).text)) end++;
    return { start: start, end: end, rows: end - start + 1 };
  }

  // DOM row index -> document line number. Row 0 is the header; the delimiter line is
  // skipped, so body row i lives at start + 2 + i.
  function rowLine(blk, domRow) {
    return domRow === 0 ? blk.start : blk.start + 1 + domRow;
  }

  // Locate the table a rendered <table> belongs to, purely from the document.
  //
  // Deliberately NOT the widget's data-pos attribute as the primary source: CodeMirror
  // maps decoration positions through a change, but the rendered HTML keeps whatever
  // data-pos it was built with. Edit anything above a table and every data-pos inside it
  // is stale until the widget happens to re-render - which is how a second column drag
  // ended up writing a duplicate width marker. posAtDOM always reflects current state.
  function tableAt(tableEl) {
    var v = view(); if (!v || !tableEl) return null;
    var pos;
    try { pos = v.posAtDOM(tableEl); } catch (e) { pos = null; }
    if (pos === null || pos === undefined || isNaN(pos)) {
      pos = +tableEl.getAttribute("data-pos");
      if (isNaN(pos)) return null;
    }
    if (pos < 0 || pos > v.state.doc.length) return null;
    var ln = v.state.doc.lineAt(pos).number;
    // posAtDOM resolves to the widget's own position, which can sit just before the
    // first row line; snap forward one line rather than parsing the wrong block.
    if (!isRow(v.state.doc.line(ln).text) && ln < v.state.doc.lines &&
        isRow(v.state.doc.line(ln + 1).text)) ln++;
    if (!isRow(v.state.doc.line(ln).text)) return null;
    return block(v.state, ln);
  }

  function cellRange(blk, domRow, col) {
    var v = view(); if (!v) return null;
    var ln = rowLine(blk, domRow);
    if (ln < 1 || ln > v.state.doc.lines) return null;
    var cells = splitRow(v.state.doc.line(ln));
    if (!cells || col >= cells.length) return null;
    return cells[col];
  }

  function unescapePipes(s) { return s.replace(/\\\|/g, "|"); }
  function escapePipes(s) { return s.replace(/\|/g, "\\|").replace(/\n/g, " "); }

  // ---- whole-block rewrites (row / column insert + delete) --------------------------

  function rowCells(state, ln) {
    var cells = splitRow(state.doc.line(ln));
    return cells ? cells.map(function (c) { return c.text; }) : null;
  }

  function emitRow(cells) { return "| " + cells.join(" | ") + " |"; }

  function rewriteBlock(blk, fn) {
    var v = view(); if (!v) return false;
    var st = v.state, rows = [];
    for (var ln = blk.start; ln <= blk.end; ln++) {
      var cells = rowCells(st, ln);
      if (!cells) return false;
      rows.push(cells);
    }
    if (!fn(rows)) return false;
    var text = rows.map(emitRow).join("\n");
    v.dispatch({
      changes: { from: st.doc.line(blk.start).from, to: st.doc.line(blk.end).to, insert: text },
    });
    return true;
  }

  // Placeholder heading for an inserted column - see the Alt-] branch for why it cannot be blank.
  var NEW_COLUMN_NAME = "Column";

  function blankRow(width) {
    var out = [];
    for (var i = 0; i < width; i++) out.push("");
    return out;
  }

  // ---- the cell editor -------------------------------------------------------------

  var active = null;   // {input, tableIdx, domRow, col, original}

  function tables() { return D.querySelectorAll(".sb-table-widget table"); }

  function blkFor(tableIdx) {
    var tbl = tables()[tableIdx];
    return tbl ? tableAt(tbl) : null;
  }

  function cellEl(tableIdx, domRow, col) {
    var tbl = tables()[tableIdx];
    if (!tbl) return null;
    var rows = [].concat(
      [].slice.call(tbl.querySelectorAll("thead tr")),
      [].slice.call(tbl.querySelectorAll("tbody tr")));
    var tr = rows[domRow];
    if (!tr) return null;
    return tr.children[Math.min(col, tr.children.length - 1)] || null;
  }

  function closeEditor(commit) {
    var a = active;
    if (!a) return;
    active = null;
    // Hidden unconditionally, including on the commit-then-reopen path: focusCell brings it
    // straight back when the next cell opens, and leaving it up in between meant a failed
    // re-open (see focusCell's give-up branch) stranded a toolbar over a table nothing was
    // editing.
    hideToolbar();
    // Newlines are flattened at the single writer, not at the keyboard. `<input>` used to
    // strip them from a paste for free; a textarea keeps them, and a newline written into
    // a row line splits it in two and structurally breaks the table. Paste, drag-drop and
    // anything else all funnel through here, so this is the one place it cannot be missed.
    var value = a.input.value.replace(/\s*\r?\n\s*/g, " ");
    a.input.remove();
    if (!commit || value === a.original) return;
    var v = view(); if (!v) return;
    // Recompute the block: the widget may have re-rendered since the editor opened.
    var blk = blkFor(a.tableIdx) || a.blk;
    var r = cellRange(blk, a.domRow, a.col);
    if (!r) return;
    v.dispatch({ changes: { from: r.from, to: r.to, insert: " " + escapePipes(value) + " " } });
  }

  // Alpha of a computed colour. getComputedStyle always resolves to rgb()/rgba(), but
  // `transparent` survives on some engines, so handle it explicitly.
  function alphaOf(c) {
    if (!c || c === "transparent") return 0;
    var m = /^rgba?\(([^)]+)\)$/.exec(c);
    if (!m) return 1;
    var parts = m[1].split(/[,\/]/);
    return parts.length > 3 ? parseFloat(parts[3]) : 1;
  }

  // Single writer for the overlay's box, used at open time, on every keystroke, and on
  // every scroll or resize. Height is the cell's, or the wrapped content's when that is
  // taller - never less, or a short cell's editor would be smaller than the cell it covers.
  function sizeEditor(a, rect) {
    var i = a.input;
    i.style.left = rect.left + "px";
    i.style.top = rect.top + "px";
    i.style.width = rect.width + "px";
    // Always measure from the cell's own padding, never from the padded-out value written
    // by the previous call - this runs on every keystroke and would otherwise compound.
    i.style.paddingTop = a.padTop + "px";
    i.style.paddingBottom = a.padBottom + "px";
    i.style.height = "0px";
    var natural = i.scrollHeight;          // wrapped content plus the cell's padding
    if (natural > rect.height) { i.style.height = natural + "px"; return; }
    // Table cells are `vertical-align: middle` and a textarea is top-aligned, so a short
    // cell in a tall row had its text jump upward the moment the editor opened. Give the
    // slack back as padding instead.
    var slack = (rect.height - natural) / 2;
    i.style.paddingTop = (a.padTop + slack) + "px";
    i.style.paddingBottom = (a.padBottom + slack) + "px";
    i.style.height = rect.height + "px";
  }

  // The only opener. Addresses a cell by index and reads the live DOM, so it is always
  // safe to call after a re-render.
  function openEditorAt(tableIdx, domRow, col) {
    var v = view(); if (!v) return;
    var td = cellEl(tableIdx, domRow, col);
    var blk = blkFor(tableIdx);
    if (!td || !blk) return;
    var r = cellRange(blk, domRow, col);
    if (!r) return;

    var rect = td.getBoundingClientRect();
    // A textarea, not an input: the raw Markdown of a cell is routinely wider than the
    // rendered cell (`Finmark <br/>(Financial planning <br/>& analysis)` is 555px of text
    // in a 236px column), and a single-line input clipped it to whatever happened to fit.
    // It wraps and auto-grows instead, so the whole cell source is visible while editing.
    // Newlines are kept out two ways: Enter is swallowed in onEditorKey, and a pasted or
    // dropped one is flattened in closeEditor, which is the only writer.
    var input = D.createElement("textarea");
    input.rows = 1;
    input.className = "sb-tbl-cell-input";
    input.value = unescapePipes(r.text);
    input.style.cssText =
      "position:fixed;left:" + rect.left + "px;top:" + rect.top + "px;" +
      "width:" + rect.width + "px;height:" + rect.height + "px;";
    // Typography is copied from the cell, not inherited. The input lives on document.body
    // so that a widget re-render cannot steal its focus, which also means `font: inherit`
    // gives it the *body* font - the editor was rendering the space's monospace cells in
    // Times New Roman. Copy rather than hardcode, so it follows any theme or custom style.
    // Vertical padding is copied too: a textarea is top-aligned, so without it the first
    // line sits above the rendered text it replaces.
    var cs = W.getComputedStyle(td);
    ["fontFamily", "fontSize", "fontWeight", "fontStyle", "fontVariant", "letterSpacing",
     "lineHeight", "color", "textAlign", "paddingLeft", "paddingRight",
     "paddingTop", "paddingBottom"].forEach(function (k) {
      input.style[k] = cs[k];
    });
    // Rows are striped, so a fixed white background would flash a different colour on
    // every other row - but the stripe is often *translucent* (rgba(26,27,39,.3) in the
    // dark theme), and copying it verbatim onto a floating element left the rendered cell
    // legible straight through the editor. Paint the cell's own colour as a layer over the
    // first fully opaque ancestor instead; background-color plus a flat linear-gradient is
    // the only way to stack two colours on one element.
    var base = "";
    for (var el = td; el; el = el.parentElement) {
      var c = W.getComputedStyle(el).backgroundColor;
      if (alphaOf(c) === 1) { base = c; break; }
    }
    // No opaque ancestor at all (the page background comes from the canvas). Theme variable
    // first - falling straight to #fff would put a white box on a dark page, which is worse
    // than the bug being fixed.
    if (!base) {
      base = W.getComputedStyle(D.documentElement)
              .getPropertyValue("--root-background-color").trim() || "#fff";
    }
    input.style.backgroundColor = base;
    var layer = cs.backgroundColor;
    if (alphaOf(layer) > 0) {
      input.style.backgroundImage = "linear-gradient(" + layer + "," + layer + ")";
    }

    // Below 16px mobile browsers zoom the page on focus and leave it zoomed - the same
    // floor [[Library/neupsh/MobileUX]] applies to every other input.
    if (parseFloat(cs.fontSize) < 16 && W.matchMedia("(pointer: coarse)").matches) {
      input.style.fontSize = "16px";
    }
    D.body.appendChild(input);
    active = { input: input, tableIdx: tableIdx, domRow: domRow, col: col, blk: blk,
               original: input.value,
               padTop: parseFloat(cs.paddingTop) || 0,
               padBottom: parseFloat(cs.paddingBottom) || 0 };
    sizeEditor(active, rect);   // needs to be in the document to have a scrollHeight
    showToolbar();
    input.focus({ preventScroll: true });
    input.select();

    input.addEventListener("blur", function () { closeEditor(true); });
    input.addEventListener("keydown", onEditorKey, true);
    // Typing past the current wrap grows the box; deleting shrinks it back to the cell.
    input.addEventListener("input", repositionEditor);
  }

  // Re-open the editor on a cell after the widget has re-rendered.
  //
  // A single rAF was not enough. Core re-renders the widget on its own schedule, so one frame
  // after a structural rewrite the DOM is often still the *old* table. That failed two ways:
  // `cellEl` returned null and `openEditorAt` bailed silently - leaving focus to fall back to
  // CodeMirror, which scrolls its selection into view, i.e. the top of the table or of the page -
  // or, worse, the stale DOM still had a `tr` at the requested index (the row that used to be
  // there) and the editor opened over the wrong row.
  //
  // So poll, and gate on the table's *shape* rather than on mere existence: `expect` is the
  // {rows, cols} the caller knows the table has after its rewrite, which is what distinguishes the
  // re-rendered table from the stale one. Both halves are needed - a column insert leaves the row
  // count identical, so rows alone would wave the stale DOM straight through, which is exactly how
  // Alt-] came to open its editor on the pre-insert table. Callers that change nothing structural
  // still pass the current shape; a null `expect` waits only for the cell to exist.
  var FOCUS_FRAMES = 40;   // ~650ms at 60fps; a re-render that slow has failed anyway

  function focusCell(tableIdx, domRow, col, expect) {
    var frames = 0;
    (function attempt() {
      requestAnimationFrame(function () {
        var blk = blkFor(tableIdx);
        var td = blk && cellEl(tableIdx, domRow, col);
        // Rows come from the document block, columns from the rendered row - the two together
        // pin down the DOM we are about to address.
        var settled = blk && td && (!expect ||
          (bodyRows(blk) === expect.rows && td.parentNode.children.length === expect.cols));
        if (!settled) {
          if (++frames < FOCUS_FRAMES) return attempt();
          // Give up rather than open on a table we cannot vouch for - but say so. This is the
          // branch that leaves you with no editor and the focus back in CodeMirror, i.e. the
          // original "it threw me to the top of the table" symptom. It should now be unreachable;
          // if it is ever hit, this line is the only evidence of why.
          if (W.console && W.console.warn) {
            W.console.warn("[tableEditor] gave up re-opening the cell editor after " +
              FOCUS_FRAMES + " frames", { tableIdx: tableIdx, domRow: domRow, col: col,
              expect: expect, sawRows: blk ? bodyRows(blk) : null,
              sawCols: td ? td.parentNode.children.length : null, haveCell: !!td });
          }
          return;
        }
        // The row we are about to edit may be below the fold - a new last row usually is. Bring it
        // into view *before* opening, because the overlay is `position:fixed` and is sized from the
        // cell's viewport rect; scrolling afterwards would leave it stranded.
        var r = td.getBoundingClientRect();
        if (r.top < 60 || r.bottom > (W.innerHeight || 0) - 60) {
          td.scrollIntoView({ block: "center", behavior: "auto" });
          return requestAnimationFrame(function () { openEditorAt(tableIdx, domRow, col); });
        }
        openEditorAt(tableIdx, domRow, col);
      });
    })();
  }

  // Where does a clicked cell live, in indices rather than element references?
  function addressOf(td) {
    var tableEl = td.closest("table");
    var tr = td.closest("tr");
    if (!tableEl || !tr) return null;
    return {
      tableIdx: [].indexOf.call(tables(), tableEl),
      domRow: td.closest("thead") ? 0 : [].indexOf.call(tr.parentNode.children, tr) + 1,
      col: td.cellIndex,
    };
  }

  function bodyRows(blk) { return blk.rows - 2; }   // minus header and delimiter

  // The four structural edits, shared by the Alt chords and by the touch toolbar's buttons.
  // Keyed by the chord that used to be their only trigger, so the mapping stays obvious.
  // Everything it needs is re-derived from `active`, because a toolbar button is pressed with
  // no keyboard event to read it off.
  function structuralEdit(key) {
    var a = active; if (!a) return;
    var v = view(); if (!v) return;
    var blk = blkFor(a.tableIdx) || a.blk;
    var lastRow = bodyRows(blk);
    var headerCells = splitRow(v.state.doc.line(blk.start));
    var width = headerCells ? headerCells.length : 1;
    var t = a.tableIdx, r = a.domRow, c = a.col;
    closeEditor(true);
    var blk3 = blkFor(t);
    if (!blk3) return;
    var ok = rewriteBlock(blk3, function (rows) {
      if (key === "Enter") { rows.splice(r + 2, 0, blankRow(width)); return true; }
      if (key === "Backspace") {
        if (r === 0 || rows.length <= 3) return false;   // never delete the header
        rows.splice(r + 1, 1); return true;
      }
      if (key === "]") {
        // The new header cell must not be blank. SilverBullet's renderer *drops* an empty header
        // cell entirely while keeping the body cell, so `| a | b |  | c |` renders a 3-column
        // header over 4-column rows - every value from the insert point rightward displays under
        // the wrong heading. Measured: a named header renders 4/4, an empty or whitespace-only
        // one renders 3/4. So seed a placeholder; it is selected on arrival, so typing replaces it.
        rows.forEach(function (row, i) {
          row.splice(c + 1, 0, i === 0 ? NEW_COLUMN_NAME : i === 1 ? "---" : "");
        });
        return true;
      }
      if (width <= 1) return false;
      rows.forEach(function (row) { row.splice(c, 1); });
      return true;
    });
    if (!ok) { focusCell(t, r, c, { rows: lastRow, cols: width }); return; }
    // Land on the *first* cell of the new row, not the column Alt-Enter was pressed in: the
    // point of inserting a row is to start filling it, and that starts at column 0. Matches
    // where Tab-off-the-last-cell lands.
    if (key === "Enter") return focusCell(t, r + 1, 0, { rows: lastRow + 1, cols: width });
    if (key === "Backspace") return focusCell(t, Math.min(r, lastRow - 1), c, { rows: lastRow - 1, cols: width });
    // Land on the new column's *header*, not its body cell: the placeholder heading is the one
    // thing that must be replaced, and it arrives selected so typing overwrites it.
    if (key === "]") return focusCell(t, 0, c + 1, { rows: lastRow, cols: width + 1 });
    return focusCell(t, r, Math.max(0, c - 1), { rows: lastRow, cols: width - 1 });
  }

  function onEditorKey(e) {
    var a = active; if (!a) return;
    var v = view(); if (!v) return;
    var blk = blkFor(a.tableIdx) || a.blk;
    var lastRow = bodyRows(blk);           // domRow of the final body row
    var headerCells = splitRow(v.state.doc.line(blk.start));
    var width = headerCells ? headerCells.length : 1;
    var t = a.tableIdx, r = a.domRow, c = a.col;

    // Navigation only - a commit rewrites one cell and never changes the row count, so the
    // expected count is simply the current one.
    function go(nr, nc) { e.preventDefault(); closeEditor(true); focusCell(t, nr, nc, { rows: lastRow, cols: width }); }

    // The overlay is a textarea, so every Enter that is not one of the commands below
    // would otherwise insert a literal newline. That newline survives escapePipes and
    // splits the row across two lines on commit, silently destroying the table - guard it
    // once here rather than trusting every branch to preventDefault (Shift-Enter did not).
    if (e.key === "Enter" && !e.altKey && !e.ctrlKey && !e.metaKey) e.preventDefault();

    if (e.key === "Escape") {
      e.preventDefault(); e.stopPropagation();
      closeEditor(false);
      return;
    }
    if (e.key === "Tab") {
      e.preventDefault(); e.stopPropagation();
      if (e.shiftKey) {
        if (c > 0) return go(r, c - 1);
        if (r > 0) return go(r - 1, width - 1);
        return closeEditor(true);
      }
      if (c < width - 1) return go(r, c + 1);
      if (r < lastRow) return go(r + 1, 0);
      // Tab out of the last cell appends a row.
      e.preventDefault();
      closeEditor(true);
      var blk2 = blkFor(t);
      if (blk2 && rewriteBlock(blk2, function (rows) { rows.push(blankRow(width)); return true; })) {
        focusCell(t, lastRow + 1, 0, { rows: lastRow + 1, cols: width });
      }
      return;
    }
    if (e.altKey && (e.key === "Enter" || e.key === "Backspace" ||
                     e.key === "]" || e.key === "[")) {
      e.preventDefault(); e.stopPropagation();
      structuralEdit(e.key);
      return;
    }
    // Alt-/ rather than a bare `?`: the overlay is a textarea, so `?` is a character the
    // user is entitled to type into a cell. Alt-/ prints nothing on the layouts here.
    if (e.altKey && (e.key === "/" || e.key === "?")) {
      e.preventDefault(); e.stopPropagation();
      toggleHelp(a.input);
      return;
    }
    if (e.key === "Enter" && (e.ctrlKey || e.metaKey)) {
      e.preventDefault(); e.stopPropagation();
      dropToRaw({ tableIdx: t, domRow: r, col: c });
      return;
    }
    if (e.key === "Enter" || e.key === "ArrowDown") {
      if (r < lastRow) return go(r + 1, c);
      e.preventDefault(); closeEditor(true); return;
    }
    if (e.key === "ArrowUp") {
      if (r > 0) return go(r - 1, c);
      e.preventDefault(); closeEditor(true); return;
    }
  }

  // ---- click interception ----------------------------------------------------------

  function cellFrom(e) {
    var t = e.target;
    if (!t || !t.closest) return null;
    if (t.closest(".sb-tbl-resize")) return null;
    return t.closest(".sb-table-widget td");
  }

  ["pointerdown", "mousedown", "mouseup", "click"].forEach(function (type) {
    D.addEventListener(type, function (e) {
      if (!on()) return;
      // The help button and the panel swallow the whole gesture, pointerdown included -
      // unlike the resize handle, which needs pointerdown to pass through to its drag
      // handler. preventDefault on pointerdown is what keeps focus in the cell editor:
      // without it, clicking the panel blurs the textarea, and blur commits.
      if (e.target.closest && e.target.closest(".sb-tbl-help, .sb-tbl-help-panel")) {
        e.preventDefault(); e.stopImmediatePropagation();
        if (type === "pointerdown") toggleHelp(e.target.closest(".sb-tbl-help"));
        return;
      }
      // Same deal for the touch toolbar. preventDefault on pointerdown is what stops the tap
      // from blurring the cell editor - and blur commits and tears the toolbar down, so
      // without it every button would act on an `active` that had just become null.
      var tool = e.target.closest && e.target.closest(".sb-tbl-toolbar");
      if (tool) {
        e.preventDefault(); e.stopImmediatePropagation();
        var btn = e.target.closest(".sb-tbl-tool");
        // On click, not pointerdown: these mutate the document, and acting on the first event
        // of the gesture would fire on a press the user then slid off and released elsewhere.
        if (type === "click" && btn && btn.__sbAction) btn.__sbAction(btn);
        return;
      }
      // A click anywhere else dismisses an open panel, before the cell handling below.
      if (type === "pointerdown" && helpOpen()) closeHelp();
      // A resize handle needs the same swallowing as a cell: left unhandled, CodeMirror
      // sees the mousedown, puts the cursor inside the table and un-renders it, so the
      // handle vanishes mid-drag and every later drag has nothing to grab. pointerdown
      // is deliberately let through - the drag handler below is registered after this
      // one in the same capture phase, so stopping it here would kill resizing outright.
      if (e.target.closest && e.target.closest(".sb-tbl-resize, .sb-tbl-sort")) {
        if (type !== "pointerdown") { e.preventDefault(); e.stopImmediatePropagation(); }
        return;
      }
      var td = cellFrom(e);
      if (!td) return;
      // Only swallow the event if the table actually resolves to a block we can edit.
      // GFM allows a pipe table with no outer pipes; core still renders it as a widget,
      // but `block()` walks on lines starting with `|` and returns nothing. Swallowing
      // those clicks anyway left such a table completely dead - no cell editor, and no
      // core click-to-raw either, i.e. worse than not installing this page at all.
      if (!tableAt(td.closest("table"))) return;
      e.preventDefault();
      e.stopImmediatePropagation();
      if (type !== "click" || e.detail >= 2) return;
      var at = addressOf(td);
      if (!at) return;
      if (e.altKey) return dropToRaw(at);
      if (active) {
        // Committing re-renders the widget, so re-address on the next frame rather
        // than holding on to a td that is about to be replaced.
        closeEditor(true);
        focusCell(at.tableIdx, at.domRow, at.col);
      } else {
        openEditorAt(at.tableIdx, at.domRow, at.col);
      }
    }, true);
  });

  // Escape hatch: Alt-click a cell, or Ctrl-Enter while editing, puts the cursor in the
  // raw Markdown. Not double-click: the first click opens the overlay input, so the
  // second one lands on the input and a table-level dblclick never fires - and keeping
  // dblclick free preserves word-select inside the cell editor.
  function dropToRaw(at) {
    closeEditor(true);
    if (!at) return;
    requestAnimationFrame(function () {
      var v = view(); if (!v) return;
      var blk = blkFor(at.tableIdx);
      if (!blk) return;
      var r = cellRange(blk, at.domRow, at.col);
      v.dispatch({ selection: { anchor: r ? r.from + 1 : v.state.doc.line(blk.start).to } });
      v.focus();
    });
  }

  // The overlay is fixed-positioned from a rect captured at open time, so a scroll or a
  // resize would otherwise leave it floating over the wrong cell and a later blur-commit
  // would write somewhere the user cannot see. Follow the cell instead of closing:
  // closing on scroll also loses races, because focusing the input can itself make the
  // browser scroll, which would shut the editor the instant it opened.
  function repositionEditor() {
    var a = active; if (!a) return;
    var td = cellEl(a.tableIdx, a.domRow, a.col);
    var rect = td && td.getBoundingClientRect();
    // Scrolled far enough that CodeMirror stopped rendering the line: commit and go.
    if (!rect || (rect.width === 0 && rect.height === 0)) { closeEditor(true); return; }
    sizeEditor(a, rect);
  }

  ["scroll", "resize"].forEach(function (type) {
    W.addEventListener(type, repositionEditor, true);
  });

  // ---- column widths ---------------------------------------------------------------

  // Nearest marker line above a table, crossing blank lines only.
  function markerLine(blk) {
    var v = view(); if (!v) return null;
    for (var ln = blk.start - 1; ln >= 1; ln--) {
      var text = v.state.doc.line(ln).text;
      if (MARKER_RE.test(text)) return ln;
      if (text.trim() !== "") return null;
    }
    return null;
  }

  function storedWidths(blk) {
    var ln = markerLine(blk); if (!ln) return null;
    var m = WIDTH_ATTR_RE.exec(view().state.doc.line(ln).text);
    if (!m || !m[1]) return null;
    return m[1].split(",").map(Number).filter(function (n) { return n > 0; });
  }

  function writeWidths(blk, widths) {
    var v = view(); if (!v) return;
    var line = '<div class="sb-tbl-widths" data-w="' + widths.map(Math.round).join(",") +
               '"></div>';
    var ln = markerLine(blk);
    if (ln) {
      var l = v.state.doc.line(ln);
      v.dispatch({ changes: { from: l.from, to: l.to, insert: line } });
    } else {
      // The blank line is mandatory: an HTML block butted against a table swallows it.
      var start = v.state.doc.line(blk.start);
      v.dispatch({ changes: { from: start.from, to: start.from, insert: line + "\n\n" } });
    }
  }

  function applyWidths() {
    // Belt-and-braces for the hiding the space-style rule also does. A freshly pulled
    // page can be live as Lua while custom styles have not loaded yet ("Not loading
    // custom styles, since no index is available yet"), and a visible marker line is
    // content the user did not write. Runs even when resizing is off, since documents
    // keep their markers either way.
    [].forEach.call(D.querySelectorAll(".sb-tbl-widths"), function (el) {
      var line = el.closest(".cm-line");
      if (line) line.style.display = "none";
    });
    if (!on() || cfg().resize === false) {
      // Turning either key off mid-session leaves handles behind otherwise. They are
      // inert (every handler re-checks config), but a stray grab affordance that does
      // nothing is worse than none.
      [].forEach.call(D.querySelectorAll(".sb-tbl-resize"), function (h) { h.remove(); });
      return;
    }
    [].forEach.call(tables(), function (tbl) {
      var blk = tableAt(tbl); if (!blk) return;
      var headCells = tbl.querySelectorAll("thead tr > td, thead tr > th");
      var widths = storedWidths(blk);
      if (widths) {
        tbl.style.tableLayout = "fixed";
        tbl.style.width = widths.reduce(function (a, b) { return a + b; }, 0) + "px";
        for (var i = 0; i < headCells.length && i < widths.length; i++) {
          // border-box, so the width we store is the width we measure on the next drag.
          // Without it each drag reads back padding + borders and the column creeps.
          headCells[i].style.boxSizing = "border-box";
          headCells[i].style.width = widths[i] + "px";
        }
      }
      // Resize handles live inside the widget and are re-added after every re-render.
      for (var j = 0; j < headCells.length; j++) {
        if (headCells[j].querySelector(".sb-tbl-resize")) continue;
        var h = D.createElement("span");
        h.className = "sb-tbl-resize";
        h.setAttribute("contenteditable", "false");
        headCells[j].appendChild(h);
      }
    });
  }

  // Drag to resize. Widths apply live to the DOM; the document is written once, on
  // pointer-up, so a drag costs one undo step and one sync rather than one per pixel.
  var drag = null;
  D.addEventListener("pointerdown", function (e) {
    if (!on() || cfg().resize === false) return;
    var h = e.target.closest && e.target.closest(".sb-tbl-resize");
    if (!h) return;
    e.preventDefault(); e.stopImmediatePropagation();
    var td = h.parentNode, tbl = td.closest("table");
    var heads = [].slice.call(tbl.querySelectorAll("thead tr > td, thead tr > th"));
    drag = {
      tbl: tbl, heads: heads, idx: heads.indexOf(td), x: e.clientX,
      widths: heads.map(function (c) { return c.getBoundingClientRect().width; }),
    };
    tbl.style.tableLayout = "fixed";
    heads.forEach(function (c) { c.style.boxSizing = "border-box"; });
    D.body.classList.add("sb-tbl-dragging");
  }, true);

  D.addEventListener("pointermove", function (e) {
    if (!drag) return;
    var min = cfg().minWidth || 48;
    var w = drag.widths.slice();
    w[drag.idx] = Math.max(min, drag.widths[drag.idx] + (e.clientX - drag.x));
    drag.current = w;
    for (var i = 0; i < drag.heads.length; i++) drag.heads[i].style.width = w[i] + "px";
    drag.tbl.style.width = w.reduce(function (a, b) { return a + b; }, 0) + "px";
  }, true);

  D.addEventListener("pointerup", function () {
    if (!drag) return;
    var d = drag; drag = null;
    D.body.classList.remove("sb-tbl-dragging");
    if (!d.current) return;
    var blk = tableAt(d.tbl);
    if (blk) writeWidths(blk, d.current);
  }, true);

  // ---- column sorting --------------------------------------------------------------

  // Sorting REWRITES THE SOURCE ROW ORDER rather than just reordering the DOM, and that
  // is forced, not stylistic: rowLine() maps a DOM row index straight onto a document
  // line (blk.start + 1 + domRow). Sort only the view and that identity breaks - click a
  // cell in a visually-sorted table and the cell editor writes to a different row's line.
  // Silent corruption with no error. Sorting the document keeps DOM order == source order.

  // Which column each table is sorted by, keyed by header text rather than by position:
  // a table's line numbers move when anything above it is edited, but its header does not,
  // so the caret survives both re-renders and the sort's own rewrite.
  var sortState = {};

  function headerKey(blk) {
    var v = view(); if (!v) return null;
    var cells = rowCells(v.state, blk.start);
    return cells ? cells.join("") : null;
  }

  // Sort by what a cell reads as, not by its Markdown. Without this "[[Apple]]" sorts
  // under "[", and every link in a column collates together ahead of the plain text.
  function sortText(s) {
    return unescapePipes(s)
      .replace(/!?\[\[([^\]|]*)(?:\|([^\]]*))?\]\]/g, function (m, page, alias) { return alias || page; })
      .replace(/!?\[([^\]]*)\]\([^)]*\)/g, "$1")
      .replace(/[*_`~]/g, "")
      .trim();
  }

  // Currency, thousands separators and a trailing % are all common in note tables and all
  // sort as text otherwise ("10" before "9"). Anything else is left to the string path.
  var NUM_RE = /^[-+]?[$€£¥]?\s*\d{1,3}(?:,\d{3})*(?:\.\d+)?\s*%?$|^[-+]?\d*\.?\d+(?:[eE][-+]?\d+)?$/;
  var DATE_RE = /^\d{4}-\d{2}-\d{2}([T ]\d{2}:\d{2}(:\d{2})?)?$/;

  function asNumber(s) {
    if (!NUM_RE.test(s)) return null;
    var n = parseFloat(s.replace(/[^0-9.eE+-]/g, ""));
    return isNaN(n) ? null : n;
  }

  // Column type is decided ALL-OR-NOTHING over the non-blank cells. A per-pair "numeric if
  // both look numeric" test is non-transitive - with 2, "x", 10 the result depends on the
  // comparison order the sort happens to pick, so the same table sorts differently on
  // different engines. Blanks are excluded from the vote and always collate last.
  function columnKeys(body, col) {
    var vals = body.map(function (r) { return sortText(r[col] || ""); });
    var filled = vals.filter(function (s) { return s !== ""; });
    var numeric = filled.length > 0 && filled.every(function (s) { return asNumber(s) !== null; });
    var dates = !numeric && filled.length > 0 && filled.every(function (s) { return DATE_RE.test(s); });
    return vals.map(function (s) {
      if (s === "") return null;
      if (numeric) return asNumber(s);
      return dates ? s : s.toLocaleLowerCase();   // ISO 8601 collates correctly as text
    });
  }

  function sortByColumn(blk, col, dir) {
    return rewriteBlock(blk, function (rows) {
      var body = rows.slice(2);                   // past the header and the delimiter
      if (body.length < 2) return false;
      var keys = columnKeys(body, col);
      var order = body.map(function (_, i) { return i; });
      order.sort(function (x, y) {
        var a = keys[x], b = keys[y];
        // Blanks last in BOTH directions - flipping them with the sort would hide the
        // rows you are least likely to want buried, and a descending sort that leads
        // with a screen of empty cells reads as broken.
        if (a === null || b === null) {
          if (a === null && b === null) return x - y;
          return a === null ? 1 : -1;
        }
        var c = (typeof a === "number") ? a - b : String(a).localeCompare(String(b));
        if (c === 0) return x - y;                // ties keep document order: stable
        return dir === "desc" ? -c : c;
      });
      var sorted = order.map(function (i) { return body[i]; });
      // Already in this order: skip the dispatch. rewriteBlock re-emits every row, so a
      // no-op sort would still reflow the whole table into one undo step and one diff.
      if (order.every(function (i, n) { return i === n; })) return false;
      rows.length = 2;
      for (var i = 0; i < sorted.length; i++) rows.push(sorted[i]);
      return true;
    });
  }

  // Carets are injected the same way resize handles are: the widget is rebuilt on every
  // edit, so anything added to it has to be re-added rather than bound once.
  function applySortCarets() {
    if (!on() || cfg().sort === false) {
      [].forEach.call(D.querySelectorAll(".sb-tbl-sort"), function (c) { c.remove(); });
      return;
    }
    [].forEach.call(tables(), function (tbl) {
      var blk = tableAt(tbl); if (!blk) return;
      // One body row cannot be out of order; a caret there is a button that does nothing.
      if (bodyRows(blk) < 2) return;
      var key = headerKey(blk), st = key ? sortState[key] : null;
      var heads = tbl.querySelectorAll("thead tr > td, thead tr > th");
      for (var i = 0; i < heads.length; i++) {
        var c = heads[i].querySelector(".sb-tbl-sort");
        if (!c) {
          c = D.createElement("span");
          c.className = "sb-tbl-sort";
          c.setAttribute("contenteditable", "false");
          // Before the resize handle, which has to stay flush against the cell edge.
          heads[i].insertBefore(c, heads[i].querySelector(".sb-tbl-resize"));
        }
        var dir = st && st.col === i ? st.dir : "";
        c.setAttribute("data-col", i);
        c.setAttribute("data-dir", dir);
        c.title = dir === "asc" ? "Sorted A-Z - click for Z-A"
                : dir === "desc" ? "Sorted Z-A - click for A-Z"
                : "Sort by this column";
      }
    });
  }

  // On pointerdown, not click. The caret sits inside a header cell whose own clicks open
  // the cell editor, and acting on the first event of the gesture means a pointerup that
  // drifts onto a caret - the tail of a resize drag - can never trigger a sort.
  D.addEventListener("pointerdown", function (e) {
    if (!on() || cfg().sort === false) return;
    var c = e.target.closest && e.target.closest(".sb-tbl-sort");
    if (!c) return;
    e.preventDefault(); e.stopImmediatePropagation();
    var tbl = c.closest("table"), blk = tableAt(tbl);
    if (!blk) return;
    var col = +c.getAttribute("data-col");
    var key = headerKey(blk); if (!key) return;
    var st = sortState[key];
    // Two states, not three: the source row order is overwritten by the first sort, so
    // "unsorted" no longer exists to cycle back to. Ctrl-Z is the way back - the rewrite
    // is a single dispatch, so one undo restores the original order and spacing.
    var dir = (st && st.col === col && st.dir === "asc") ? "desc" : "asc";
    if (active) closeEditor(true);
    sortByColumn(blk, col, dir);
    sortState[key] = { col: col, dir: dir };
    applySortCarets();
  }, true);

  // ---- touch toolbar ---------------------------------------------------------------

  // Every structural edit is an Alt chord, and a phone keyboard has no Alt. This is the same
  // set of actions as tap targets, shown while a cell is being edited. Cell navigation is in
  // here too: Tab is the desktop way across a row, and most soft keyboards have no Tab either.
  var TOOLBAR = [
    // Plain arrows, not ◀ ▶: those carry emoji presentation by default and rendered as solid
    // orange lozenges on the phone, which read as a warning rather than as navigation.
    ["←", "Previous cell", function () { moveCell(-1); }],
    ["→", "Next cell", function () { moveCell(1); }],
    ["+ Row", "Insert a row below", function () { structuralEdit("Enter"); }],
    ["− Row", "Delete this row", function () { structuralEdit("Backspace"); }],
    ["+ Col", "Insert a column right", function () { structuralEdit("]"); }],
    ["− Col", "Delete this column", function () { structuralEdit("["); }],
    ["?", "Table shortcuts", function (btn) { toggleHelp(btn); }],
    ["✓", "Done", function () { closeEditor(true); }],
  ];

  var toolbar = null;

  // Same traversal Tab performs, minus the append-a-row tail: a toolbar press should never
  // silently grow the table, since there is a +Row button right next to it that says so.
  function moveCell(delta) {
    var a = active; if (!a) return;
    var v = view(); if (!v) return;
    var blk = blkFor(a.tableIdx) || a.blk;
    var lastRow = bodyRows(blk);
    var headerCells = splitRow(v.state.doc.line(blk.start));
    var width = headerCells ? headerCells.length : 1;
    var t = a.tableIdx, r = a.domRow, c = a.col + delta;
    if (c >= width) { if (r >= lastRow) return; r++; c = 0; }
    else if (c < 0) { if (r <= 0) return; r--; c = width - 1; }
    closeEditor(true);
    focusCell(t, r, c, { rows: lastRow, cols: width });
  }

  // Shown on coarse pointers, and on any narrow window so it can be exercised on a desktop
  // without a touchscreen. `cfg().toolbar === false` turns it off entirely.
  function wantToolbar() {
    if (!on() || cfg().toolbar === false) return false;
    var coarse = W.matchMedia && W.matchMedia("(pointer: coarse)").matches;
    return !!coarse || (W.innerWidth || 0) <= 640;
  }

  function hideToolbar() {
    if (!toolbar) return;
    toolbar.remove();
    toolbar = null;
  }

  // The keyboard-relative placement. `visualViewport` is the only thing that reports the
  // keyboard: when it opens, the visual viewport shrinks while `innerHeight` does not, so
  // `offsetTop + height` is the top edge of the keyboard and the bar sits just above it.
  // Without visualViewport (older engines) this degrades to the bottom of the window, which
  // is where the bar belongs anyway when no keyboard is up.
  function placeToolbar() {
    if (!toolbar) return;
    var vv = W.visualViewport;
    var h = vv ? vv.height : (W.innerHeight || 0);
    var top = vv ? vv.offsetTop : 0;
    toolbar.style.top = Math.max(0, top + h - toolbar.offsetHeight) + "px";
  }

  function showToolbar() {
    if (toolbar || !wantToolbar()) return;
    var bar = D.createElement("div");
    bar.className = "sb-tbl-toolbar";
    TOOLBAR.forEach(function (spec) {
      var b = D.createElement("button");
      b.type = "button";
      b.className = "sb-tbl-tool";
      b.textContent = spec[0];
      b.title = spec[1];
      b.setAttribute("aria-label", spec[1]);
      b.__sbAction = spec[2];
      bar.appendChild(b);
    });
    toolbar = bar;
    D.body.appendChild(bar);
    placeToolbar();
  }

  // The bar tracks the keyboard opening and closing, and the page scrolling under it.
  if (W.visualViewport) {
    ["resize", "scroll"].forEach(function (type) {
      W.visualViewport.addEventListener(type, placeToolbar);
    });
  }
  W.addEventListener("resize", placeToolbar);

  // ---- shortcut help ---------------------------------------------------------------

  // The panel's content, grouped. Kept here rather than read off the page so the overlay
  // works before anything else has loaded; the `## What you get` table above is the prose
  // version of the same list, and the two are meant to be changed together.
  // Backticks mark the parts that are literally keys, so "`Tab` in the last cell" renders one
  // key cap followed by prose. Without that distinction every word of "Drag a header border"
  // came out as its own cap, which reads as a four-key chord.
  var HELP = [
    ["Editing", [
      ["Click a cell", "Edit it in place"],
      ["`Tab` / `Shift-Tab`", "Next / previous cell"],
      ["`Enter` / `↓` `↑`", "Move down / up a column"],
      ["`Esc`", "Cancel this edit"],
    ]],
    ["Rows and columns", [
      ["`Alt-Enter`", "Insert a row below, land in its first cell"],
      ["`Alt-Backspace`", "Delete this row"],
      ["`Alt-]`", "Insert a column to the right"],
      ["`Alt-[`", "Delete this column"],
      ["`Tab` in the last cell", "Append a row"],
    ]],
    ["Everything else", [
      ["`Alt`-click / `Ctrl-Enter`", "Drop to the raw Markdown"],
      ["Drag a header border", "Resize the column"],
      ["Click ⇅ in a header", "Sort by that column"],
      ["`Alt-/`", "Show or hide this panel"],
    ]],
  ];

  var helpPanel = null, helpAnchor = null;

  function helpOpen() { return !!helpPanel; }

  function closeHelp() {
    if (!helpPanel) return;
    helpPanel.remove();
    helpPanel = null;
    helpAnchor = null;
  }

  function buildHelp() {
    var box = D.createElement("div");
    box.className = "sb-tbl-help-panel";
    var h = D.createElement("div");
    h.className = "sb-tbl-help-title";
    h.textContent = "Table shortcuts";
    box.appendChild(h);
    HELP.forEach(function (group) {
      var g = D.createElement("div");
      g.className = "sb-tbl-help-group";
      g.textContent = group[0];
      box.appendChild(g);
      group[1].forEach(function (pair) {
        var row = D.createElement("div");
        row.className = "sb-tbl-help-row";
        var k = D.createElement("span");
        k.className = "sb-tbl-help-keys";
        // Odd indices of a backtick split are the quoted spans, i.e. the key caps; the even
        // ones are the connective prose and go in as plain text.
        pair[0].split("`").forEach(function (part, i) {
          if (!part) return;
          if (i % 2 === 0) { k.appendChild(D.createTextNode(part)); return; }
          var kbd = D.createElement("kbd");
          kbd.textContent = part;
          k.appendChild(kbd);
        });
        var d = D.createElement("span");
        d.className = "sb-tbl-help-desc";
        d.textContent = pair[1];
        row.appendChild(k); row.appendChild(d);
        box.appendChild(row);
      });
    });
    return box;
  }

  // Anchored to whatever opened it - the table's own button, or the cell editor when the
  // panel was summoned by Alt-/ - then clamped into the viewport, because a table near the
  // right edge or low on a long page would otherwise put half the panel off-screen.
  // The anchor is remembered rather than passed in on every call: scroll and resize
  // reposition the panel, and re-deriving "no anchor" there would silently recentre it in
  // the viewport on the first scroll after opening.
  function placeHelp() {
    if (!helpPanel) return;
    var pad = 8;
    var r = helpAnchor && helpAnchor.isConnected ? helpAnchor.getBoundingClientRect() : null;
    if (r && r.width === 0 && r.height === 0) r = null;   // anchor scrolled out of the DOM
    var pr = helpPanel.getBoundingClientRect();
    var vw = W.innerWidth || D.documentElement.clientWidth;
    var vh = W.innerHeight || D.documentElement.clientHeight;
    var left = r ? r.right - pr.width : (vw - pr.width) / 2;
    var top = r ? r.bottom + 6 : (vh - pr.height) / 2;
    // Not enough room below the anchor: flip above it rather than hang off the bottom.
    if (r && top + pr.height > vh - pad) {
      top = r.top - pr.height - 6;
      if (top < pad) top = Math.max(pad, vh - pr.height - pad);
    }
    helpPanel.style.left = Math.max(pad, Math.min(left, vw - pr.width - pad)) + "px";
    helpPanel.style.top = Math.max(pad, Math.min(top, vh - pr.height - pad)) + "px";
  }

  function openHelp(anchorEl) {
    closeHelp();
    helpAnchor = anchorEl || null;
    helpPanel = buildHelp();
    D.body.appendChild(helpPanel);   // needs to be in the document to have a size
    placeHelp();
  }

  function toggleHelp(anchorEl) {
    if (helpOpen()) return closeHelp();
    openHelp(anchorEl);
  }

  // One button per rendered table, re-attached after every widget re-render. Deliberately
  // not hover-only: there is no hover on a phone, and the mobile toolbar shares this
  // panel, so the affordance has to be visible without a pointer. It sits at low opacity
  // and comes up to full on hover or focus.
  function applyHelpButtons() {
    if (!on() || cfg().help === false) {
      D.querySelectorAll(".sb-tbl-help").forEach(function (b) { b.remove(); });
      closeHelp();
      return;
    }
    D.querySelectorAll(".sb-table-widget").forEach(function (w) {
      var tbl = w.querySelector("table");
      // Same guard the click interception uses: a table this script cannot resolve is left
      // entirely alone, so it does not advertise shortcuts that would not work on it.
      if (!tbl || !tableAt(tbl)) return;
      if (w.querySelector(":scope > .sb-tbl-help")) return;
      var b = D.createElement("button");
      b.className = "sb-tbl-help";
      b.type = "button";
      b.textContent = "?";
      b.title = "Table shortcuts";
      b.setAttribute("aria-label", "Table shortcuts");
      w.insertBefore(b, w.firstChild);
    });
  }

  // Follow the anchor rather than close, for the same reason the cell editor does: the
  // panel is fixed-positioned from a rect taken at open time.
  ["scroll", "resize"].forEach(function (type) {
    W.addEventListener(type, function () {
      if (helpOpen()) placeHelp();
    }, true);
  });

  // Escape closes the panel wherever focus happens to be. Registered in the capture phase
  // so it beats CodeMirror's own Escape handling, and it does *not* fall through to the
  // cell editor's cancel - closing the help you just opened should not also discard the
  // edit you were making.
  D.addEventListener("keydown", function (e) {
    if (e.key === "Escape" && helpOpen()) {
      e.preventDefault(); e.stopImmediatePropagation();
      closeHelp();
    }
  }, true);

  // ---- keeping up with re-renders --------------------------------------------------

  // The widget is replaced wholesale on every edit, taking widths and handles with it.
  // Observe the current .cm-content, re-attaching when the editor is recreated.
  var observed = null, obs = new MutationObserver(function () {
    if (pending) return;
    pending = true;
    requestAnimationFrame(function () {
      pending = false; applyWidths(); applySortCarets(); applyHelpButtons();
    });
  });
  var pending = false;

  // The observer catches re-renders promptly; the unconditional call on each tick is the
  // backstop for everything that changes without mutating .cm-content - a config flip, or
  // custom styles arriving late and needing the marker hidden by JS instead. Cost is a
  // querySelectorAll plus a few line reads per table, twice a second.
  setInterval(function () {
    var content = D.querySelector(".cm-content");
    if (content && content !== observed) {
      observed = content;
      obs.disconnect();
      obs.observe(content, { childList: true, subtree: true });
    }
    applyWidths();
    applySortCarets();
    applyHelpButtons();
  }, 500);
})();
// @@TABLE_EDITOR_END@@
```

```space-style
/* The width marker is an HTML block, so it renders as an empty div - hide its whole line.
   Core swaps the widget back to raw text when the cursor lands there, the :has() stops
   matching, and the line becomes visible and editable again. */
.cm-line:has(.sb-tbl-widths) {
  display: none;
}

/* The widget becomes the containing block for the help button. Its `overflow` is left
   alone deliberately: core sets `auto` there and that is what lets a wide table scroll
   horizontally instead of bursting the page. The button therefore has to sit *inside* the
   box - straddling the top edge, it was sheared in half by that same clip. */
.sb-table-widget {
  position: relative;
  /* A gutter for the help button. Without it the button floats over whatever header content
     sits at the right edge, and on a wide table that is a sort caret - whose click it then
     swallows, i.e. a control the user can no longer reach. A table narrower than the page
     now clears the button entirely. A wide one still scrolls its content underneath, which
     is cosmetic: measured, the carets stay clear and the button stays the topmost element
     at its own centre, so nothing becomes unclickable. */
  padding-right: 1.7em;
}

/* Deliberately always visible rather than hover-only - there is no hover on a phone, and
   this is the discovery affordance for every other shortcut. Low opacity keeps it from
   competing with the header text until it is looked for. */
.sb-tbl-help {
  position: absolute;
  top: 3px;
  right: 0.15em;
  z-index: 2;
  width: 1.5em;
  height: 1.5em;
  padding: 0;
  border: 1px solid var(--editor-table-head-color, currentColor);
  border-radius: 50%;
  background: var(--root-background-color, #fff);
  color: inherit;
  font-size: 0.75em;
  line-height: 1;
  font-family: inherit;
  cursor: pointer;
  opacity: 0.3;
  transition: opacity 0.12s ease-in-out;
}

.sb-tbl-help:hover,
.sb-tbl-help:focus-visible {
  opacity: 1;
}

/* Touch toolbar. Positioned from the top rather than pinned with `bottom: 0`, because the
   virtual keyboard does not shrink the layout viewport on either mobile engine - `bottom`
   would put the bar behind the keyboard. placeToolbar() computes the top edge from
   visualViewport on every keyboard and scroll event. */
.sb-tbl-toolbar {
  position: fixed;
  left: 0;
  right: 0;
  /* Above .sb-tbl-cell-input's 200: the editor is a textarea that grows with its content,
     and a tall cell on a phone reaches the bottom of the screen and would cover the bar. */
  z-index: 205;
  display: flex;
  gap: 0.25rem;
  padding: 0.35rem 0.4rem;
  /* The row is wider than a phone; let it scroll rather than shrink the targets. */
  overflow-x: auto;
  scrollbar-width: none;
  border-top: 1px solid rgba(128, 128, 128, 0.3);
  background: var(--root-background-color, #fff);
  box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.15);
}

.sb-tbl-toolbar::-webkit-scrollbar {
  display: none;
}

.sb-tbl-tool {
  flex: 0 0 auto;
  /* 44px is the platform minimum for a reliable tap target on both iOS and Android. */
  min-width: 44px;
  min-height: 44px;
  padding: 0 0.7rem;
  border: 1px solid rgba(128, 128, 128, 0.35);
  border-radius: 0.45rem;
  background: rgba(128, 128, 128, 0.1);
  color: inherit;
  font: inherit;
  font-size: 0.9rem;
  line-height: 1;
  white-space: nowrap;
  cursor: pointer;
  /* Suppresses the double-tap-zoom delay and the grey tap flash on mobile Safari/Chrome. */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.sb-tbl-tool:active {
  background: rgba(128, 128, 128, 0.3);
}

/* Eight 44px targets do not fit across a phone, so the row scrolls - but Done is the way
   out and must never be the button that scrolled off. Pin it to the right edge and let the
   rest pass underneath. */
.sb-tbl-tool:last-child {
  position: sticky;
  right: 0;
  background: var(--root-background-color, #fff);
  box-shadow: -6px 0 6px -4px rgba(0, 0, 0, 0.25);
}

/* Translucent, per the ask - the table stays readable underneath while the panel is up. */
.sb-tbl-help-panel {
  position: fixed;
  /* Above both the cell editor (200) and the toolbar (205) - the panel is summoned from
     inside an open edit, and at a lower z-index the edited cell punched through it. */
  z-index: 210;
  max-width: min(26rem, calc(100vw - 1rem));
  max-height: calc(100vh - 1rem);
  overflow-y: auto;
  padding: 0.7rem 0.9rem;
  border: 1px solid rgba(128, 128, 128, 0.35);
  border-radius: 0.5rem;
  background: color-mix(in srgb, var(--root-background-color, #fff) 88%, transparent);
  backdrop-filter: blur(6px);
  box-shadow: 0 6px 24px rgba(0, 0, 0, 0.22);
  font-size: 0.85em;
  line-height: 1.5;
}

.sb-tbl-help-title {
  font-weight: 600;
  margin-bottom: 0.35rem;
}

.sb-tbl-help-group {
  margin: 0.55rem 0 0.2rem;
  opacity: 0.6;
  font-size: 0.85em;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.sb-tbl-help-group:first-of-type {
  margin-top: 0.15rem;
}

/* Two columns, keys fixed so the descriptions line up down the panel. */
.sb-tbl-help-row {
  display: grid;
  grid-template-columns: 13em 1fr;
  gap: 0.6rem;
  align-items: baseline;
}

/* The column may wrap as a whole; an individual key cap never splits across lines. */
.sb-tbl-help-keys kbd {
  white-space: nowrap;
}

.sb-tbl-help-panel kbd {
  display: inline-block;
  padding: 0.05em 0.35em;
  border: 1px solid rgba(128, 128, 128, 0.5);
  border-bottom-width: 2px;
  border-radius: 0.25em;
  background: rgba(128, 128, 128, 0.12);
  font-family: var(--editor-font, monospace);
  font-size: 0.85em;
}

.sb-tbl-help-desc {
  opacity: 0.85;
}

/* Narrow screens: stack the two columns rather than squeeze the descriptions to threads. */
@media (max-width: 480px) {
  .sb-tbl-help-panel {
    /* Shrink-to-fit leaves a stacked panel unhelpfully narrow, so claim the screen width. */
    min-width: min(21rem, calc(100vw - 1rem));
  }

  .sb-tbl-help-row {
    grid-template-columns: 1fr;
    gap: 0;
    margin-bottom: 0.3rem;
  }
}

/* Sort caret. Inline-block with a fixed width so the header text never reflows between
   the neutral and sorted states - a caret that widens on click nudges the whole row. */
.sb-tbl-sort {
  display: inline-block;
  width: 1em;
  margin-left: 0.25em;
  cursor: pointer;
  user-select: none;
  text-align: center;
  font-size: 0.8em;
  line-height: 1;
  opacity: 0.25;
  vertical-align: middle;
}

.sb-tbl-sort::before {
  content: "⇅";
}

/* Sorted columns keep their caret visible; unsorted ones stay faint until hovered, so a
   wide table is not a row of icons competing with the header text. */
.sb-tbl-sort:hover {
  opacity: 0.9;
}

.sb-tbl-sort[data-dir="asc"],
.sb-tbl-sort[data-dir="desc"] {
  opacity: 0.85;
  color: var(--editor-command-button-color, #464cfc);
}

.sb-tbl-sort[data-dir="asc"]::before {
  content: "▲";
}

.sb-tbl-sort[data-dir="desc"]::before {
  content: "▼";
}

/* Mid-drag the pointer is nowhere near the caret it started from; leaving hover styling
   live makes it flicker as the column moves under the cursor. */
body.sb-tbl-dragging .sb-tbl-sort {
  pointer-events: none;
}

/* The inline cell editor. Fixed-positioned over the cell it replaces, and deliberately
   outside .cm-content: an input living inside the widget loses focus the moment the
   widget re-renders. */
.sb-tbl-cell-input {
  z-index: 200;
  box-sizing: border-box;
  background: var(--root-background-color, #fff);
  /* An outline rather than a border: it paints outside the box, so the cell's own padding
     can be copied across verbatim and the text does not shift sideways when the editor
     opens over it. Font, colour and padding are set inline from the cell - see the script. */
  border: none;
  outline: 2px solid var(--editor-command-button-color, #464cfc);
  border-radius: 2px;
  /* It is a textarea holding one logical line: wrap it, never scroll it (the script sizes
     it to its own content), and no drag-resize grip over the cell's bottom-right corner. */
  resize: none;
  overflow: hidden;
  white-space: pre-wrap;
  overflow-wrap: break-word;
  margin: 0;
}

/* Grab area on the right edge of each header cell. */
.sb-tbl-resize {
  position: absolute;
  top: 0;
  right: -3px;
  width: 7px;
  height: 100%;
  /* Without a z-index the handle is painted but not hit-testable: elementFromPoint at its
     centre returns the neighbouring td, and the drag never starts. */
  z-index: 5;
  cursor: col-resize;
  user-select: none;
}

.sb-table-widget thead td,
.sb-table-widget thead th {
  position: relative;
}

.sb-tbl-resize:hover {
  background: var(--editor-command-button-color, #464cfc);
  opacity: 0.4;
}

/* Keep the cursor consistent for the whole drag, not just over the 7px handle. */
body.sb-tbl-dragging,
body.sb-tbl-dragging * {
  cursor: col-resize !important;
  user-select: none !important;
}

/* Column resize is a pointer affordance; on a phone it only gets in the way. */
@media (max-width: 600px) and (pointer: coarse) {
  .sb-tbl-resize {
    display: none;
  }
}
```
