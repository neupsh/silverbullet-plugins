---
name: Library/neupsh/VersionBadge
tags: meta/library
description: "Shows the running SilverBullet version as small text in the top bar."
---

# Version badge

Shows the running SilverBullet version as small text in the top bar, just before the action buttons - not an action button itself (no icon, not clickable, nothing to run).

There's no Space Lua slot for the top bar chrome (`#sb-top`) the way `actionButtons` is a slot for the button row - it's plain DOM outside any widget/panel API. So this reaches into the DOM directly via the `js.*` bridge (same technique already used for the Sync and Reload button's `js.window.location.reload()`): `#sb-top .sb-actions` is the button row (confirmed via `Library/Styles/UI.md`'s `#sb-top .sb-actions` mobile-toolbar rule), and the badge is inserted as its previous sibling. `js.document` is NOT valid (indexes nil) - only `js.window` is bound, so DOM access has to go through `js.window.document` (verified live: a bare `js.document.getElementById` threw "attempt to index a nil value" on every `editor:pageLoaded` dispatch until switched to `js.window.document`).

Runs on `editor:pageLoaded` rather than at top-level evaluation, because the top bar isn't guaranteed to be mounted yet when Library pages first evaluate during boot - a loaded page guarantees the shell exists. Gated on the badge's id already being present so repeat navigations don't insert it again.

## Always collapsed, click to expand

The full version string is long - `2.10.0-0-g2b2a7c71-2026-07-28T12-40-06Z` is 39 characters and measures ~210px, which on a 412px phone squeezed the page title down to 56px. A narrow *desktop* window (split-screen laptop) had the same problem. The build hash and timestamp are also rarely what you want to read - the `X.Y.Z` is.

So the badge is **collapsed at every width**: it renders only its `X.Y.Z` prefix (~30px), and **clicking it expands the full build string** for 4 seconds before collapsing again. There is no breakpoint. Earlier versions of this page expanded above 600px, then 1000px, then 1400px; each of those was still too eager, and the responsive rule bought nothing over just clicking when you actually want the hash.

The split is done in the markup rather than by measuring: the badge holds two spans, `.sb-version-short` (always shown) and `.sb-version-rest` (shown only while `.is-expanded`). No resize handler, no re-render, nothing to go stale.

The full version is also on the badge as a `title` attribute, which is a native hover tooltip on desktop - so on a mouse you can read it without clicking at all.

The toggle is an inline `onclick` attribute rather than an `addEventListener` callback, because that would require handing a Lua function across the `js.*` bridge; an attribute is just a string and sidesteps the question entirely.

## Stacked with the git-sync badge on a phone

This badge and any other top-bar badge page (a sync-status badge, say) sit between the page title and the action buttons, and side by side they cost ~100px of a 412px bar. They now share a wrapper, `#sb-badge-stack`, which is a flex **column** under 600px and a row above it - so on a phone the pair costs one badge's width (46px measured on a Pixel 7) and the title keeps the rest. Each page creates-or-finds the wrapper with its own copy of the same tiny helper rather than sharing one, because the pages have no guaranteed evaluation order; order inside is fixed by inline `order` (version 1), not by who ran first. The wrapper is styled inline because a client can have Space Lua live and custom styles not yet loaded - and the version badge drops to `padding: 1px 4px` under 600px so two rows fit inside the 56px bar without growing it. The wrapper aligns to the **top** of the bar with a margin rather than `align-self: center`: opening the hamburger turns that button column into a full-screen-height flex item, and a centred sibling rides halfway down the page with it - which is exactly what the first version did.

**Known nit:** the badge only appears after the first in-app navigation. On a cold load `editor:pageLoaded` fires before Space Lua has finished registering listeners, so the very first page render has no badge. Navigating anywhere inserts it.

```space-lua
-- Create-or-find the shared badge column that this badge and any sibling badge
-- both live in. Styled inline because a client can have Space Lua live and
-- custom styles not yet loaded.
-- On a phone the badges stack, so they cost one badge's width instead of two.
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
  s.flex = "0 0 auto"          -- never grow at the page title's expense
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

event.listen {
  name = "editor:pageLoaded",
  run = function()
    local doc = js.window.document
    if doc.getElementById("sb-version-badge") then return end
    local stack = badgeStack(doc)
    if not stack then return end

    local full = tostring(system.getVersion())
    -- Leading semver, e.g. "2.9.0" out of "2.9.0-0-g72bba941-2026-06-11T07-04-08Z".
    local short = string.match(full, "^%d+%.%d+%.%d+") or full
    local rest = string.sub(full, #short + 1)

    local badge = doc.createElement("span")
    badge.id = "sb-version-badge"
    badge.title = full

    local shortEl = doc.createElement("span")
    shortEl.className = "sb-version-short"
    shortEl.textContent = short
    badge.appendChild(shortEl)

    if rest ~= "" then
      local restEl = doc.createElement("span")
      restEl.className = "sb-version-rest"
      restEl.textContent = rest
      badge.appendChild(restEl)

      -- Tap to reveal the full string, auto-collapsing after 4s.
      badge.setAttribute("onclick",
        "this.classList.toggle('is-expanded');" ..
        "clearTimeout(this._verTimer);" ..
        "if (this.classList.contains('is-expanded')) {" ..
        "  this._verTimer = setTimeout(function (s) {" ..
        "    s.classList.remove('is-expanded');" ..
        "  }, 4000, this);" ..
        "}")
    end

    badge.style.order = "1"
    stack.appendChild(badge)
  end
}
```

```space-style
#sb-version-badge {
  font-size: 0.7rem;
  opacity: 0.55;
  align-self: center;
  white-space: nowrap;
  color: var(--action-button-color);
  flex: 0 0 auto;          /* never grow at the page title's expense */

  /* Collapsed is the default at every width - it is tap/clickable, so give it
     a real tap target. */
  cursor: pointer;
  padding: 6px 2px;
}

/* Stacked over the git-sync badge on a phone: the 56px top bar only fits two
   rows if each stays short, so trade the tall tap padding for a wider one. */
@media only screen and (max-width: 600px) {
  #sb-version-badge {
    padding: 1px 4px;
    line-height: 1.2;
  }
}

/* Collapse to the semver prefix so the page title keeps its width. */
#sb-version-badge .sb-version-rest {
  display: none;
}

#sb-version-badge.is-expanded .sb-version-rest {
  display: inline;
}

/* Expanding needs more room than a narrow bar has, so the page title clips
   behind it. Give the expanded badge the bar's own background (the box-shadow
   widens that opaque area without taking part in layout) so it reads as a
   chip sitting over the title instead of colliding with it. */
#sb-version-badge.is-expanded {
  position: relative;
  z-index: 1;
  opacity: 0.95;
  border-radius: 3px;
  background-color: var(--top-background-color);
  box-shadow: 0 0 0 5px var(--top-background-color);
}
```
