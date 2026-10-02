---
name: Library/neupsh/Plugins
tags: meta/library
description: "A plugin manager page: lists the pages in silverbullet-plugins and installs, updates or removes each one with a button."
---
# Plugins

Install, update and remove the add-on pages from [silverbullet-plugins](https://github.com/neupsh/silverbullet-plugins). Each add-on is one page under `Library/neupsh/`. Installing writes that page with `share.*` frontmatter, so it also works with SilverBullet's own **Share: Page** command.

## Set up once

Create a page named `Library/neupsh/Plugins` containing only this, then run **Share: Page** and choose **Ok**:

```yaml
---
share.uri: "github:neupsh/silverbullet-plugins/Plugins.md"
share.mode: pull
---
```

After that, this page lists everything and keeps itself current: **Update** on the `Plugins` row pulls the newest list.

## The list

${plugins.panel()}

- **Install** writes the page. **Update** pulls the published version, and asks first if you edited your copy. **Remove** deletes the page.
- Every change ends with a reload so new commands and styles take effect. Commands are also in the palette: `Plugins: Install`, `Plugins: Update`, `Plugins: Remove`, `Plugins: Update All`.
- A page marked **edited** differs from what was installed. Update replaces your edits after a confirm.
- Keep the page path `Library/neupsh/<Name>`. Several add-ons read their own page by that name.

```space-lua
plugins = plugins or {}

local REPO = "github:neupsh/silverbullet-plugins/"
local FOLDER = "Library/neupsh/"

-- name, group, what it does. The one list this page keeps; add a row when the repo gains a page.
plugins.catalog = {
  { "Plugins", "Manager", "This page. Update it to get the newest list." },
  { "Folding", "Editing", "Collapse a heading section or bullet subtree, optionally remembered per page." },
  { "TableEditor", "Editing", "Edit Markdown tables in place in the rendered view." },
  { "PlainTags", "Editing", "Show inline #tags as plain linked words (toggle command)." },
  { "ProseCopy", "Editing", "Copy a selection as prose, with tags and wiki links flattened." },
  { "MathDollarGuard", "Editing", "Stop \"$23B\" being read as inline LaTeX. Needs Silverbullet-Math." },
  { "PagePickerInput", "Editing", "Copy the highlighted page-picker result into the input." },
  { "MobileUX", "Layout", "Make the command palette, page picker and other modals usable on a phone." },
  { "TextDensity", "Layout", "Smaller text overall, sizes set under config." },
  { "LinkedWidgets", "Layout", "Move the Linked Tasks widget below the page body." },
  { "TreeViewLinks", "Layout", "Open Tree View rows in a new tab on Ctrl/middle-click. Needs the Tree View plug." },
  { "VersionBadge", "Top bar and page info", "Show the SilverBullet version in the top bar." },
  { "SyncBadge", "Top bar and page info", "Show how long ago git last fetched. Needs a git space and shell access." },
  { "PageDates", "Top bar and page info", "Created / Updated line on every page, created date from git." },
  { "JournalFeatures", "Journal", "Prev/next day links, a rollup of everything scheduled for the day, a stream page." },
  { "JournalCalendar", "Journal", "Month-grid picker that opens or creates any journal day." },
  { "JournalPromote", "Journal", "Move a bullet and its children to their own page, leaving a linked task." },
  { "JournalConventions", "Journal", "A short note on working in the journal first and promoting later." },
}

local function entry(name)
  for _, e in ipairs(plugins.catalog) do
    if e[1] == name then return e end
  end
  error("Unknown plugin: " .. tostring(name))
end

local function path(name)
  entry(name)
  return FOLDER .. name
end

-- "missing", "ok" or "edited" (local text no longer matches the hash recorded at the last pull).
local function state(name)
  local p = path(name)
  if not space.pageExists(p) then return "missing" end
  local text = space.readPage(p)
  local m = index.extractFrontmatter(text).frontmatter.share
  if type(m) == "table" and m.hash != nil and m.hash != share.contentHash(text) then
    return "edited"
  end
  return "ok"
end

local function readRemote(name)
  local uri = REPO .. name .. ".md"
  local text = net.readURI(uri, { encoding = "text/markdown" })
  if not text then error("Could not read " .. uri) end
  return uri, text
end

-- Writes the published page and records the hash, so the first Share: Page has nothing to ask about.
function plugins.install(name)
  local uri, text = readRemote(name)
  local stamped = share.setFrontmatter({ uri = uri, hash = share.contentHash(text), mode = "pull" }, text)
  space.writePage(path(name), stamped)
end

-- True if the page changed. Share asks before overwriting local edits.
function plugins.update(name)
  return share.sharePage(path(name))
end

function plugins.remove(name)
  space.deletePage(path(name))
end

local function reload()
  system.reloadConfig()
  editor.reloadUI()
end

-- Runs one action, tells the user what happened, and reloads when something changed.
local function run(verb, name, fn)
  local ok, res = pcall(fn)
  if not ok then
    editor.flashNotification(verb .. " " .. name .. " failed: " .. tostring(res), "error")
    return
  end
  if res == false then
    editor.flashNotification(name .. ": already up to date")
    return
  end
  editor.flashNotification(name .. ": " .. verb .. " done")
  reload()
end

command.define {
  name = "Plugins: Install",
  run = function(name)
    if not name then
      local options = {}
      for _, e in ipairs(plugins.catalog) do
        if state(e[1]) == "missing" then
          options[#options + 1] = { name = e[1], description = e[3] }
        end
      end
      local pick = editor.filterBox("Install plugin", options)
      if not pick then return end
      name = pick.name
    end
    run("Install", name, function() plugins.install(name) return true end)
  end
}

command.define {
  name = "Plugins: Update",
  run = function(name)
    if not name then
      local options = {}
      for _, e in ipairs(plugins.catalog) do
        if state(e[1]) != "missing" then
          options[#options + 1] = { name = e[1], description = e[3] }
        end
      end
      local pick = editor.filterBox("Update plugin", options)
      if not pick then return end
      name = pick.name
    end
    run("Update", name, function() return plugins.update(name) end)
  end
}

command.define {
  name = "Plugins: Remove",
  run = function(name)
    if not name then
      local options = {}
      for _, e in ipairs(plugins.catalog) do
        if state(e[1]) != "missing" then
          options[#options + 1] = { name = e[1], description = e[3] }
        end
      end
      local pick = editor.filterBox("Remove plugin", options)
      if not pick then return end
      name = pick.name
    end
    if not editor.confirm("Delete " .. path(name) .. "? Reinstall later from the Plugins page.") then return end
    run("Remove", name, function() plugins.remove(name) return true end)
  end
}

command.define {
  name = "Plugins: Update All",
  run = function()
    local changed, failed = {}, {}
    for _, e in ipairs(plugins.catalog) do
      if state(e[1]) != "missing" then
        local ok, res = pcall(plugins.update, e[1])
        if not ok then failed[#failed + 1] = e[1] .. " (" .. tostring(res) .. ")"
        elseif res then changed[#changed + 1] = e[1] end
      end
    end
    local msg = #changed .. " updated"
    if #failed > 0 then msg = msg .. ", failed: " .. table.concat(failed, "; ") end
    editor.flashNotification(msg, #failed > 0 and "error" or "info")
    if #changed > 0 then reload() end
  end
}

command.define {
  name = "Plugins: Install All",
  run = function()
    local n = 0
    for _, e in ipairs(plugins.catalog) do
      if state(e[1]) == "missing" then
        local ok, res = pcall(plugins.install, e[1])
        if ok then n = n + 1 else
          editor.flashNotification("Install " .. e[1] .. " failed: " .. tostring(res), "error")
        end
      end
    end
    editor.flashNotification(n .. " installed")
    if n > 0 then reload() end
  end
}

-- A button that runs a palette command for one plugin. The page name goes in as an argument,
-- never into the handler text, so no quoting can break it.
local function button(label, command, name)
  return '<button class="sb-plugins-btn" onclick="window.client.runCommandByName(\'' .. command
    .. '\',[\'' .. name .. '\']);return false">' .. label .. '</button>'
end

function plugins.panel()
  local rows, group = {}, nil
  for _, e in ipairs(plugins.catalog) do
    local name, grp, desc = e[1], e[2], e[3]
    if grp != group then
      group = grp
      rows[#rows + 1] = '<tr class="sb-plugins-group"><th colspan="3">' .. grp .. '</th></tr>'
    end
    local s = state(name)
    local label, buttons
    if s == "missing" then
      label = '<span class="sb-plugins-off">not installed</span>'
      buttons = button("Install", "Plugins: Install", name)
    else
      label = s == "edited" and '<span class="sb-plugins-edited">edited</span>' or "installed"
      buttons = button("Update", "Plugins: Update", name) .. button("Remove", "Plugins: Remove", name)
    end
    rows[#rows + 1] = '<tr><td class="sb-plugins-name">' .. name .. '</td><td>' .. desc
      .. '</td><td class="sb-plugins-act"><div>' .. label .. '</div>' .. buttons .. '</td></tr>'
  end
  local all = '<p class="sb-plugins-all">'
    .. '<button class="sb-plugins-btn" onclick="window.client.runCommandByName(\'Plugins: Install All\');return false">Install all</button>'
    .. '<button class="sb-plugins-btn" onclick="window.client.runCommandByName(\'Plugins: Update All\');return false">Update all</button></p>'
  return widget.htmlBlock(all .. '<table class="sb-plugins">' .. table.concat(rows) .. '</table>')
end
```

```space-style
.sb-plugins { width: 100%; border-collapse: collapse; }
.sb-plugins td, .sb-plugins th { padding: 6px 8px; text-align: left; vertical-align: top; border-bottom: 1px solid var(--subtle-color, #ddd); }
.sb-plugins-group th { font-size: 0.85em; text-transform: uppercase; opacity: 0.7; padding-top: 14px; }
.sb-plugins-name { font-weight: 600; white-space: nowrap; }
.sb-plugins-act { white-space: nowrap; text-align: right; }
.sb-plugins-act div { font-size: 0.8em; opacity: 0.75; margin-bottom: 2px; }
.sb-plugins-edited { color: var(--editor-warning-color, #b8860b); font-weight: 600; opacity: 1; }
.sb-plugins-btn { margin-left: 6px; padding: 2px 10px; cursor: pointer; }
.sb-plugins-all { margin: 0 0 8px; }
.sb-plugins-all .sb-plugins-btn { margin: 0 8px 0 0; }
@media (max-width: 600px) {
  .sb-plugins td:nth-child(2) { display: none; }
}
```
