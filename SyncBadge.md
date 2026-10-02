---
name: Library/neupsh/SyncBadge
tags: meta/library
description: "A top-bar badge showing how long ago the git repository last fetched, red when overdue."
---
# Git sync badge

Shows how long ago the space's git repository last fetched from its remote, in the top bar: `↻ 4m ago`. It goes red when the last fetch is older than `syncBadge.staleMinutes` (default 25). Hover for the exact time plus the latest commit's short hash and subject. Tap to show them inline for a few seconds.

It reads `.git/FETCH_HEAD` (rewritten on every fetch) and `git log -1` through `shell.run`, in one `sh -c` call every two minutes at most. So it needs the space folder to be a git repository, something on a schedule that fetches or pulls (a cron job, for example), and shell enabled on the server with both `sh` and `git` allowed.

Written to sit after [[Library/neupsh/VersionBadge]] in the top bar. Both share a wrapper, `#sb-badge-stack`, created by whichever runs first. On a phone the two stack instead of sitting side by side.

All styling is inline, with no `space-style` block. A freshly pulled page can be live as Lua before the client has loaded custom styles, and a badge styled from CSS then renders as title-sized text.

Test any change to the top bar with a phone device profile, not a narrow desktop window. The mobile toolbar only switches on for `pointer: coarse`.

```space-lua
-- Survives re-evaluation of this block, so nothing is re-fetched needlessly on reload.
gitBadge = gitBadge or {}

local MINUTE, HOUR, DAY = 60, 3600, 86400
local REFRESH_EVERY = 2 * MINUTE   -- fetches are scheduled externally, so no need to poll faster
local STALE_AFTER = config.get("syncBadge.staleMinutes", 25) * MINUTE

local function relativeLabel(ts)
  local age = os.time() - ts
  if age < MINUTE then
    return "just now"
  elseif age < HOUR then
    return math.floor(age / MINUTE) .. "m ago"
  elseif age < DAY then
    return math.floor(age / HOUR) .. "h ago"
  else
    return math.floor(age / DAY) .. "d ago"
  end
end

-- One round trip: fetch mtime on line 1, "<commit epoch>\t<subject>" on line 2.
local function refresh()
  local ok, res = pcall(function()
    return shell.run("sh", { "-c",
      "stat -c %Y .git/FETCH_HEAD 2>/dev/null || echo 0; git log -1 --format=%ct%x09%h%x09%s" })
  end)
  -- Offline, or shell disabled: keep whatever we last knew.
  if not ok or not res or res.code ~= 0 then return end

  local lines = {}
  for line in string.gmatch(res.stdout, "[^\n]+") do
    lines[#lines + 1] = line
  end

  local fetchedAt = tonumber(lines[1] or "0") or 0
  if fetchedAt > 0 then
    gitBadge.fetchedAt = fetchedAt
  end

  local commitAt, sha, subject = string.match(lines[2] or "", "^(%d+)\t(%x+)\t(.*)$")
  if commitAt then
    gitBadge.commitAt = tonumber(commitAt)
    gitBadge.sha = sha
    gitBadge.subject = subject
  end

  gitBadge.checkedAt = os.time()
end

-- Create-or-find the shared badge column - the same helper as
-- [[Library/neupsh/VersionBadge]]'s, duplicated rather than shared because the
-- two pages have no guaranteed evaluation order. Inline styles only, for this
-- page's stated reason. On a phone the badges stack vertically, so the pair
-- costs one badge's width of the top bar instead of two.
local function badgeStack(doc)
  local el = doc.getElementById("sb-badge-stack")
  if el then return el end

  local actions = doc.querySelector("#sb-top .sb-actions")
  if not actions then return nil end

  el = doc.createElement("span")
  el.id = "sb-badge-stack"
  local s = el.style
  s.display = "flex"
  -- flex-start, not center: opening the hamburger grows that column to full
  -- screen height, and a centred sibling would ride halfway down the page with
  -- it. The margin does the vertical centring inside the 56px bar instead.
  s.alignSelf = "flex-start"
  s.flex = "0 0 auto"
  s.marginRight = "0.5rem"
  s.minWidth = "0"
  local narrow = js.window.matchMedia("(max-width: 600px)").matches
  s.flexDirection = narrow and "column" or "row"
  s.alignItems = narrow and "flex-end" or "center"
  s.gap = narrow and "2px" or "0.5rem"
  -- 5px/7px centre the stack in the 56px bar (29px tall stacked, 25px in a
  -- row, on top of the wrapper's own 8px offset). Re-derive if either badge's
  -- padding or line-height changes.
  s.marginTop = narrow and "5px" or "7px"
  actions.parentNode.insertBefore(el, actions)
  return el
end

-- Everything the badge needs to look right, without a stylesheet.
local function build(doc)
  local el = doc.createElement("span")
  el.id = "sb-gitsync-badge"

  local s = el.style
  s.fontSize = "0.7rem"
  s.lineHeight = "1"
  s.opacity = "0.55"
  s.whiteSpace = "nowrap"
  s.flex = "0 0 auto"        -- never grow at the page title's expense
  s.cursor = "pointer"
  s.order = "2"              -- under the version badge in the shared stack

  -- Tap/click appends the exact stamp for 4s - a phone's stand-in for hover.
  -- Toggles inline display, so this survives a client with no custom styles.
  el.setAttribute("onclick",
    "var f = this.querySelector('.sb-gitsync-full');" ..
    "var open = f.style.display !== 'inline';" ..
    "f.style.display = open ? 'inline' : 'none';" ..
    "clearTimeout(this._gitTimer);" ..
    "if (open) {" ..
    "  this._gitTimer = setTimeout(function (x) { x.style.display = 'none'; }, 4000, f);" ..
    "}")

  -- A real text node, not a ::before, for the same reason.
  local glyph = doc.createElement("span")
  glyph.textContent = "↻ "
  el.appendChild(glyph)

  local relEl = doc.createElement("span")
  relEl.className = "sb-gitsync-rel"
  el.appendChild(relEl)

  local fullEl = doc.createElement("span")
  fullEl.className = "sb-gitsync-full"
  fullEl.style.display = "none"
  el.appendChild(fullEl)

  return el
end

local function paint()
  if not gitBadge.fetchedAt then return end

  local doc = js.window.document
  -- Re-resolved on every paint, not cached: the stack is a foreign node inside
  -- a preact-managed subtree, so it can disappear on a re-render.
  local stack = badgeStack(doc)
  if not stack then return end

  local el = doc.getElementById("sb-gitsync-badge")
  if not el then
    el = build(doc)
    stack.appendChild(el)
  end

  local label = relativeLabel(gitBadge.fetchedAt)
  local relEl = el.querySelector(".sb-gitsync-rel")
  -- Unchanged since the last tick - don't touch the DOM at all.
  if relEl.textContent ~= label then
    relEl.textContent = label
  end

  local full = " · " .. os.date("%Y-%m-%d %H:%M", gitBadge.fetchedAt)
  if gitBadge.sha then
    full = full .. " · " .. gitBadge.sha
  end
  el.querySelector(".sb-gitsync-full").textContent = full

  local tooltip = "Last pulled from GitHub " ..
    os.date("%Y-%m-%d %H:%M:%S", gitBadge.fetchedAt)
  if gitBadge.commitAt then
    tooltip = tooltip .. "\nHEAD " .. (gitBadge.sha or "?") .. " " ..
      os.date("%Y-%m-%d %H:%M", gitBadge.commitAt) .. " - " .. (gitBadge.subject or "")
  end

  if os.time() - gitBadge.fetchedAt > STALE_AFTER then
    el.style.color = "var(--error-color, #c0392b)"
    el.style.opacity = "0.9"
    el.title = tooltip .. "\nThe scheduled fetch looks stuck."
  else
    el.style.color = "var(--action-button-color)"
    el.style.opacity = "0.55"
    el.title = tooltip
  end
end

local function tick()
  -- `not gitBadge.sha` forces one refetch after an upgrade that adds a field: the
  -- table survives a System: Reload, so otherwise the new field stays nil - and the
  -- tooltip reads "HEAD ?" - until the 2-minute window elapses.
  if not gitBadge.sha or os.time() - (gitBadge.checkedAt or 0) >= REFRESH_EVERY then
    refresh()
  end
  paint()
end

event.listen { name = "editor:pageLoaded", run = tick }

-- Age the label. The handle lives on `window` so a Space Lua reload can clear the
-- previous interval instead of stacking a second one on top of it.
if js.window.__sbGitTimer then
  js.window.clearInterval(js.window.__sbGitTimer)
end
js.window.__sbGitTimer = js.window.setInterval(tick, 30000)
```
