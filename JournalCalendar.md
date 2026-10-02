---
name: Library/neupsh/JournalCalendar
tags: meta/library
description: "A month-grid date picker that opens or creates any journal day, past or future."
---
# Journal calendar

A month-grid date picker for opening **any** journal day - past or future - creating it if it doesn't exist yet. Fills the gap left by the built-in commands: `Journal: Today` only ever opens today, and `Journal: Previous/Next Day` + `Journal: Picker` only walk days that *already* exist.

| Do this | Get this |
| --- | --- |
| Calendar action button, or `Ctrl-q d` | Month grid overlay, current month |
| Click a day | Opens it, creating the page if needed |
| `‹` / `›`, or `PageUp` / `PageDown` | Previous / next month |
| `t` | Jump the grid back to today |
| `Escape`, or click outside | Close |

Days that already have an entry carry a dot. Today is ringed; the day you're currently viewing is filled.

## The bug this had to work around

`journal.openOrCreate(dateStr)` (in the built-in `Library/Std/Journal/Journal`) uses `dateStr` for the **page name** but renders `Library/Std/Journal/Template`, whose frontmatter is hardcoded:

```yaml
date: ${date.today()}
```

So creating `Journal/2026-08-05` on 2026-08-02 yields a page whose frontmatter says `date: 2026-08-02`. That is not cosmetic - **everything sorts on the frontmatter date, not the filename**: `journal.entries()` (`order by j.date desc`) drives Prev/Next Day, and `journalStream()` orders by `p.date`. A wrong date here misroutes navigation *silently*, weeks later.

The built-in is only correct because it is only ever called with `date.today()`.

We do **not** fix this by swapping `journal.template` globally - that would change what `Journal: Today` produces, which is the command actually used every day. Instead `journalCal.openOrCreate` calls the built-in and then patches the frontmatter `date:` line to the target date. A custom template still works, and today's behaviour is untouched.

## Lua

```space-lua
-- priority: 10
journalCal = journalCal or {}

config.define("journalCalendar.enabled", {
  type = "boolean",
  default = true,
  description = "Month-grid picker for opening or creating any journal day.",
  ui = { category = "Journal", label = "Enable journal calendar", priority = 30 },
})

config.define("journalCalendar.weekStartsMonday", {
  type = "boolean",
  default = true,
  description = "Start the calendar week on Monday rather than Sunday.",
  ui = { category = "Journal", label = "Week starts Monday", priority = 31 },
})

-- Open a journal day, creating it if absent, with the frontmatter date the target day
-- (NOT the creation day - see the note above).
function journalCal.openOrCreate(dateStr)
  if not string.match(tostring(dateStr), "^%d%d%d%d%-%d%d%-%d%d$") then
    editor.flashNotification("Not a date: " .. tostring(dateStr), "error")
    return
  end
  local pageName = config.get("journal.prefix") .. dateStr
  if space.pageExists(pageName) then
    editor.navigate(pageName)
    return
  end
  -- Write the file FIRST, then navigate. We deliberately do NOT use
  -- `template.createPageFromTemplate` here, and the reason is load-bearing:
  --
  --   1. Its template hardcodes `date: ${date.today()}`, so a future page gets today's date.
  --   2. It also NAVIGATES. Patching the file afterwards loses a race with the live editor
  --      buffer, which still holds the stale frontmatter and flushes over our write. Verified
  --      empirically: the patched date came back as today's twice, silently.
  --
  -- Writing before navigating means there is exactly one writer and no buffer to fight.
  -- Trade-off, stated loudly: a customised `journal.template` does NOT apply to days created
  -- here - it still applies to `Journal: Today`. Keep the seed below in sync if you change it.
  space.writePage(pageName, string.format(
    "---\ntags: %s\ndate: %s\n---\n* ", config.get("journal.tag"), dateStr))
  editor.navigate(pageName)
end

-- Publish state the overlay reads: existing days (for the dots), today, and the page prefix.
-- Re-published on navigation so a day created just now gets its dot without a reload.
local lastPublished = nil
local function publishState()
  local days = {}
  for _, p in ipairs(query[[ from p = index.pages(config.get("journal.tag")) select p ]]) do
    local d = string.match(p.name, "(%d%d%d%d%-%d%d%-%d%d)$")
    if d then days[#days + 1] = '"' .. d .. '"' end
  end
  table.sort(days)
  local payload = string.format(
    'window.__sbJournalCal = {days: [%s], today: "%s", current: "%s", mondayFirst: %s, enabled: %s};',
    table.concat(days, ","),
    date.today(),
    tostring(editor.getCurrentPage() or ""),
    tostring(config.get("journalCalendar.weekStartsMonday", true)),
    tostring(config.get("journalCalendar.enabled", true)))
  if payload ~= lastPublished then
    lastPublished = payload
    js.window.eval(payload)
  end
end

command.define {
  name = "Journal: Open Day",
  key = "Ctrl-q d",
  run = function()
    if not config.get("journalCalendar.enabled", true) then
      editor.flashNotification("Journal calendar is disabled")
      return
    end
    publishState()
    js.window.eval("window.__sbJournalCalOpen && window.__sbJournalCalOpen();")
  end,
}

-- Invoked from the overlay. runCommandByName(name, [iso]) delivers the array as a SINGLE
-- table argument, not spread - so it's args[1], not the first parameter. (Verified against
-- the client: `a.run(r)` passes the args array through untouched.)
command.define {
  name = "Journal: Go To Date",
  hide = true,
  run = function(args)
    local d = args
    if type(args) == "table" then d = args[1] end
    journalCal.openOrCreate(tostring(d))
  end,
}

if js and js.window and js.window.document then
  publishState()
  event.listen { name = "editor:pageLoaded", run = publishState }

  -- Same loader shape as [[Library/neupsh/TableEditor]]: the JS lives in a real fenced
  -- `javascript` block below (a payload this size inside a Lua long string fails to parse),
  -- and the sentinels are ASSEMBLED so this loader doesn't match itself.
  local src = space.readPage("Library/neupsh/JournalCalendar")
  local TAG = "@@JOURNAL" .. "_CALENDAR"
  local START, STOP = "// " .. TAG .. "_START@@", "// " .. TAG .. "_END@@"
  local i = string.find(src, START, 1, true)
  local j = string.find(src, STOP, 1, true)
  if i and j then
    js.window.eval(string.sub(src, i, j + string.len(STOP) - 1))
  else
    js.window.console.warn("[JournalCalendar] could not find the script block on its own page")
  end
end
```

## Styles

```space-style
.sb-jcal-backdrop {
  position: fixed; inset: 0; z-index: 100000;
  background: rgba(0, 0, 0, 0.28);
  display: flex; align-items: flex-start; justify-content: center;
  padding-top: 12vh;
}
.sb-jcal {
  background: var(--root-background-color, #fff);
  color: var(--root-color, #111);
  border: 1px solid var(--subtle-background-color, #d8d8d8);
  border-radius: 10px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.28);
  padding: 14px 16px 16px;
  width: min(340px, 92vw);
  font-size: 14px;
  user-select: none;
}
.sb-jcal-head {
  display: flex; align-items: center; justify-content: space-between;
  margin-bottom: 10px; gap: 8px;
}
.sb-jcal-title { font-weight: 600; }
.sb-jcal-nav { display: flex; gap: 4px; }
.sb-jcal-nav button, .sb-jcal-today {
  background: transparent; border: 1px solid transparent; color: inherit;
  border-radius: 6px; cursor: pointer; padding: 3px 8px; font-size: 14px; line-height: 1.2;
}
.sb-jcal-nav button:hover, .sb-jcal-today:hover {
  background: var(--subtle-background-color, #eee);
}
.sb-jcal-today { font-size: 12px; opacity: 0.75; }
.sb-jcal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 2px; }
.sb-jcal-dow {
  text-align: center; font-size: 11px; opacity: 0.55;
  padding-bottom: 4px; text-transform: uppercase; letter-spacing: 0.04em;
}
.sb-jcal-day {
  position: relative; aspect-ratio: 1 / 1;
  display: flex; align-items: center; justify-content: center;
  border: none; background: transparent; color: inherit;
  border-radius: 8px; cursor: pointer; font-size: 13px; font-variant-numeric: tabular-nums;
}
.sb-jcal-day:hover { background: var(--subtle-background-color, #ececec); }
.sb-jcal-day.other { opacity: 0.3; }
.sb-jcal-day.today { box-shadow: inset 0 0 0 1.5px var(--link-color, #4c7ecc); font-weight: 600; }
.sb-jcal-day.current { background: var(--link-color, #4c7ecc); color: #fff; }
.sb-jcal-day.has-entry::after {
  content: ""; position: absolute; bottom: 5px; left: 50%; transform: translateX(-50%);
  width: 4px; height: 4px; border-radius: 50%;
  background: var(--link-color, #4c7ecc);
}
.sb-jcal-day.current.has-entry::after { background: #fff; }
.sb-jcal-hint { margin-top: 10px; font-size: 11px; opacity: 0.55; text-align: center; }
@media (max-width: 480px) { .sb-jcal { font-size: 15px; } .sb-jcal-day { font-size: 14px; } }
```

## The script

Loaded by the Lua block above. Edit it here; **System: Reload** picks it up.

```javascript
// @@JOURNAL_CALENDAR_START@@
(function () {
  var W = window;
  if (W.__sbJournalCalInstalled) return;   // space-lua re-evaluates; install once
  W.__sbJournalCalInstalled = true;

  var DOW_MON = ["Mo", "Tu", "We", "Th", "Fr", "Sa", "Su"];
  var DOW_SUN = ["Su", "Mo", "Tu", "We", "Th", "Fr", "Sa"];
  var MONTHS = ["January", "February", "March", "April", "May", "June",
                "July", "August", "September", "October", "November", "December"];

  function state() { return W.__sbJournalCal || { days: [], today: "", current: "", mondayFirst: true }; }

  function iso(y, m, d) {
    return y + "-" + String(m + 1).padStart(2, "0") + "-" + String(d).padStart(2, "0");
  }
  // Parse YYYY-MM-DD as a LOCAL date. `new Date("2026-08-05")` parses as UTC and can land on
  // the previous day west of Greenwich, which would ring the wrong cell as "today".
  function parseIso(s) {
    var m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(s || "");
    return m ? new Date(+m[1], +m[2] - 1, +m[3]) : null;
  }

  var backdrop = null, viewY = 0, viewM = 0;

  function close() {
    if (backdrop && backdrop.parentNode) backdrop.parentNode.removeChild(backdrop);
    backdrop = null;
    document.removeEventListener("keydown", onKey, true);
  }

  function pick(dateStr) {
    close();
    // Hand back to Space Lua, which owns page creation + the frontmatter date fix.
    if (W.client && W.client.runCommandByName) {
      W.client.runCommandByName("Journal: Go To Date", [dateStr]);
    }
  }

  function onKey(e) {
    if (!backdrop) return;
    if (e.key === "Escape") { e.preventDefault(); e.stopPropagation(); close(); }
    else if (e.key === "PageUp") { e.preventDefault(); shift(-1); }
    else if (e.key === "PageDown") { e.preventDefault(); shift(1); }
    else if (e.key === "t" || e.key === "T") {
      e.preventDefault();
      var t = parseIso(state().today) || new Date();
      viewY = t.getFullYear(); viewM = t.getMonth(); render();
    }
  }

  function shift(n) {
    viewM += n;
    while (viewM < 0) { viewM += 12; viewY--; }
    while (viewM > 11) { viewM -= 12; viewY++; }
    render();
  }

  function render() {
    var st = state();
    var entries = {};
    (st.days || []).forEach(function (d) { entries[d] = true; });
    var todayStr = st.today;
    var currentStr = (/(\d{4}-\d{2}-\d{2})$/.exec(st.current || "") || [])[1] || "";
    var mondayFirst = st.mondayFirst !== false;
    var dow = mondayFirst ? DOW_MON : DOW_SUN;

    var panel = backdrop.firstChild;
    panel.innerHTML = "";

    var head = document.createElement("div");
    head.className = "sb-jcal-head";
    var title = document.createElement("div");
    title.className = "sb-jcal-title";
    title.textContent = MONTHS[viewM] + " " + viewY;
    var nav = document.createElement("div");
    nav.className = "sb-jcal-nav";
    var todayBtn = document.createElement("button");
    todayBtn.className = "sb-jcal-today";
    todayBtn.textContent = "Today";
    todayBtn.onclick = function () {
      var t = parseIso(todayStr) || new Date();
      viewY = t.getFullYear(); viewM = t.getMonth(); render();
    };
    ["‹", "›"].forEach(function (label, idx) {
      var b = document.createElement("button");
      b.textContent = label;
      b.setAttribute("aria-label", idx === 0 ? "Previous month" : "Next month");
      b.onclick = function () { shift(idx === 0 ? -1 : 1); };
      nav.appendChild(b);
    });
    head.appendChild(title);
    var right = document.createElement("div");
    right.className = "sb-jcal-nav";
    right.appendChild(todayBtn);
    right.appendChild(nav);
    head.appendChild(right);
    panel.appendChild(head);

    var grid = document.createElement("div");
    grid.className = "sb-jcal-grid";
    dow.forEach(function (d) {
      var c = document.createElement("div");
      c.className = "sb-jcal-dow";
      c.textContent = d;
      grid.appendChild(c);
    });

    var first = new Date(viewY, viewM, 1);
    // Day-of-week offset of the 1st, rebased so the configured start-of-week is column 0.
    var offset = (first.getDay() - (mondayFirst ? 1 : 0) + 7) % 7;
    var daysInMonth = new Date(viewY, viewM + 1, 0).getDate();
    var prevDays = new Date(viewY, viewM, 0).getDate();

    function cell(y, m, d, other) {
      var s = iso(y, m, d);
      var b = document.createElement("button");
      b.className = "sb-jcal-day" + (other ? " other" : "") +
        (s === todayStr ? " today" : "") +
        (s === currentStr ? " current" : "") +
        (entries[s] ? " has-entry" : "");
      b.textContent = String(d);
      b.title = s + (entries[s] ? " · has an entry" : " · will be created");
      b.onclick = function () { pick(s); };
      grid.appendChild(b);
    }

    for (var i = offset - 1; i >= 0; i--) {
      var pm = viewM - 1, py = viewY;
      if (pm < 0) { pm = 11; py--; }
      cell(py, pm, prevDays - i, true);
    }
    for (var d = 1; d <= daysInMonth; d++) cell(viewY, viewM, d, false);
    var used = offset + daysInMonth;
    var trail = (7 - (used % 7)) % 7;
    for (var k = 1; k <= trail; k++) {
      var nm = viewM + 1, ny = viewY;
      if (nm > 11) { nm = 0; ny++; }
      cell(ny, nm, k, true);
    }
    panel.appendChild(grid);

    var hint = document.createElement("div");
    hint.className = "sb-jcal-hint";
    hint.textContent = "PgUp/PgDn month · t today · Esc close";
    panel.appendChild(hint);
  }

  W.__sbJournalCalOpen = function () {
    if (backdrop) { close(); return; }          // re-invoking the command toggles
    var st = state();
    if (st.enabled === false) return;
    var start = parseIso((/(\d{4}-\d{2}-\d{2})$/.exec(st.current || "") || [])[1]) ||
                parseIso(st.today) || new Date();
    viewY = start.getFullYear();
    viewM = start.getMonth();

    backdrop = document.createElement("div");
    backdrop.className = "sb-jcal-backdrop";
    var panel = document.createElement("div");
    panel.className = "sb-jcal";
    backdrop.appendChild(panel);
    backdrop.addEventListener("mousedown", function (e) {
      if (e.target === backdrop) close();       // click-outside, but not click-inside-drag
    });
    document.body.appendChild(backdrop);
    // Capture phase: SB binds its own document-level key handlers, and Escape would otherwise
    // reach the editor before us.
    document.addEventListener("keydown", onKey, true);
    render();
  };
})();
// @@JOURNAL_CALENDAR_END@@
```
