---
name: Library/neupsh/PageDates
tags: meta/library
description: "A small Created / Updated line at the top of every page, with the created date taken from git."
---
# Page dates

Shows a muted `Created … · Updated …` line at the top of every page.

SilverBullet's own `created` field is the file's modified time, so it cannot say when a page was made. The created date here comes from git instead: the commit that first added the page file (`git log --follow --diff-filter=A`), so a renamed page keeps its original date. The updated date is the file's modified time.

Needs a git repository as the space folder, and `shell.run` enabled on the server. Without either, only the Updated half shows.

- Pages added in the repo's very first commit show **In space since <date>**, because their real creation date is unknown.
- A page not committed yet shows **Created pending**. It is not cached, so the next visit picks the date up once it is committed.
- Built-in `Library/Std/*` pages are read-only and not in git, so they get no stamp.
- Each page's git lookup is cached until **System: Reload**.
- Uses the top widget slot. If you also use [[Library/neupsh/LinkedWidgets]], that page already leaves the top slot free.
- Git must trust the space folder. In Docker, set `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=safe.directory GIT_CONFIG_VALUE_0=/space`, or every lookup fails and every page reads "Created pending".

```space-lua
-- priority: 100
-- Registered early so the date line sorts above the journal prev/next nav,
-- which registers its own `hooks:renderTopWidgets` listener.

pageDates = pageDates or {}

-- pageName -> { iso = "2026-07-24T12:28:31-05:00", floored = false }
local createdCache = {}
local rootCommits = nil

-- Bridge strings (shell.run stdout, editor.getCurrentPage) report type=="string"
-- but carry no string metatable, so `string.*` is used in function form throughout
-- rather than as methods - see .agent/memory/silverbullet-pkim.md.
local function gitLines(args)
  local ok, r = pcall(shell.run, "git", args)
  if not ok or type(r) != "table" or r.code != 0 then
    return nil
  end
  local lines = {}
  for line in string.gmatch(r.stdout, "[^\n]+") do
    table.insert(lines, line)
  end
  return lines
end

local function isRootCommit(hash)
  if rootCommits == nil then
    rootCommits = {}
    local lines = gitLines { "rev-list", "--max-parents=0", "HEAD" }
    if lines then
      for _, h in ipairs(lines) do
        rootCommits[h] = true
      end
    end
  end
  return rootCommits[hash] == true
end

-- Returns isoDate, floored. `floored` means the page was added in the repo's
-- root commit and therefore genuinely predates the history - its real creation
-- date is unknowable, so callers must not present the root commit date as one.
local function gitCreated(pageName)
  local hit = createdCache[pageName]
  if hit then
    return hit.iso, hit.floored
  end
  local lines = gitLines {
    "log", "--follow", "--diff-filter=A", "--format=%H %aI", "--", pageName .. ".md"
  }
  if not lines or #lines == 0 then
    -- Either not committed yet (exit 0, empty stdout) or the shell is
    -- unreachable. Not cached either way, so this self-heals.
    return nil, false
  end
  -- --follow walks newest-first, so the page's own creation is the last line.
  local last = lines[#lines]
  local hash = string.sub(last, 1, 40)
  local iso = string.sub(last, 42)
  local floored = isRootCommit(hash)
  createdCache[pageName] = { iso = iso, floored = floored }
  return iso, floored
end

-- Both date sources are ISO strings in LOCAL time, differing only in what
-- trails the seconds - git's `%aI` carries an offset ("2026-07-24T12:28:31-05:00")
-- and `lastModified` carries milliseconds ("2026-08-11T12:35:45.045") - so the
-- same slice serves both. Do NOT treat `lastModified` as epoch milliseconds: it
-- is a string, and arithmetic on it throws "attempt to div a 'string' with a
-- 'number'", which takes the whole hooks:renderTopWidgets dispatch down with it.
local function fmtStamp(iso)
  if type(iso) != "string" or #iso < 16 then
    return nil
  end
  return string.sub(iso, 1, 10) .. " " .. string.sub(iso, 12, 16)
end

event.listen {
  name = "hooks:renderTopWidgets",
  run = function()
    local pageName = editor.getCurrentPage()
    if not pageName or pageName == "" then
      return
    end

    -- Bail out on a page that has no file yet - following a link to a page you
    -- have not created renders the editor for it, and stamping "Created pending"
    -- on a page that does not exist is noise. This also saves the git call.
    local okMeta, meta = pcall(space.getPageMeta, pageName)
    if not okMeta or not meta then
      return
    end

    local parts = {}
    local iso, floored = gitCreated(pageName)
    if iso and floored then
      parts[#parts + 1] = '<span title="Added in the repo's first commit - it predates the git history, so its real creation date is unknown">In space since <b>'
        .. string.sub(iso, 1, 10) .. "</b></span>"
    elseif iso then
      parts[#parts + 1] = '<span title="First commit that added this page (git log --follow --diff-filter=A)">Created <b>'
        .. (fmtStamp(iso) or string.sub(iso, 1, 10)) .. "</b></span>"
    elseif meta.perm == "ro" then
      -- A page with no git record that the space cannot write is one of the ~66
      -- `Library/Std/*` pages baked into the SilverBullet binary - they are not
      -- on disk and never enter the backup repo, so "pending" would be permanent.
      -- Pairing `perm` with the missing git record rather than testing `perm`
      -- alone keeps this immune to read-only mode: a real note still has history.
      return
    else
      parts[#parts + 1] = '<span title="Not committed to git yet">Created <b>pending</b></span>'
    end

    if meta.lastModified then
      local updated = fmtStamp(meta.lastModified)
      if updated then
        parts[#parts + 1] = '<span title="File mtime, as indexed by SilverBullet">Updated <b>'
          .. updated .. "</b></span>"
      end
    end

    return widget.htmlBlock(
      '<div class="sb-page-dates">'
        .. table.concat(parts, '<span class="sb-page-dates-sep">·</span>')
        .. "</div>"
    )
  end
}
```

```space-style
/* Strip std's widget chrome for this widget only. Std's own rules are
   `#sb-main .cm-editor .sb-lua-top-widget { border: 1px solid …; min-height: … }`
   and `… .content { max-height: 500px; padding: 10px }`, so the full selector is
   needed to outrank them on specificity. */
#sb-main .cm-editor .sb-lua-top-widget:has(.sb-page-dates) {
  border: none;
  background: none;
  min-height: 0;
  margin: 0 0 0.4rem 0;
  padding: 0;
}

#sb-main .cm-editor .sb-lua-top-widget:has(.sb-page-dates) .content {
  padding: 0;
  max-height: none;
}

.sb-page-dates {
  display: flex;
  flex-wrap: wrap;
  align-items: baseline;
  gap: 0.4rem;
  font-size: 0.8em;
  line-height: 1.4;
  opacity: 0.55;
}

.sb-page-dates b {
  font-weight: 600;
}

.sb-page-dates-sep {
  opacity: 0.5;
}
```
