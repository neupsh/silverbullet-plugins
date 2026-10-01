---
name: Library/neupsh/TextDensity
tags: meta/library
description: "Makes SilverBullet text smaller overall, with the sizes set under config."
---

# Text density

Makes SilverBullet's text smaller overall, and puts the sizes under `config` so they are set the same way as everything else in this space - no CSS editing.

## Configuring it

In `CONFIG.md` (or any `space-lua` block):

```lua
config.set("textDensity", {
  editorFontSize = "16px",       -- note body        (core: 18px)
  titleFontSize = "20px",        -- page title       (core: 28px)
  mobileTitleFontSize = "20px",  -- page title <=600px
})
```

Individual keys work too: `config.set("textDensity.editorFontSize", "15px")`. Any CSS length is valid - `px`, `rem`, `%`, `clamp(...)`. A bare number is read as `px`, so `16` and `"16px"` mean the same thing.

Setting nothing gives the defaults above. Core's own sizes are `18px` / `28px` if you want back to stock. Below ~14px the editor's checkbox and code-block chrome starts to look cramped, and the top-bar action icons are a fixed size regardless, so a very small title reads as misaligned against them.

Config changes need a `System: Reload` (Ctrl-Alt-r) to take effect, like any Space Lua change.

The tree view's size is **not** one of these keys - it lives in CSS, for the reason in [[#The tree view]] below.

## Why this needs CSS underneath

SilverBullet has **no built-in setting for font size**. The only font-related CSS variables core ships are `--editor-font`, `--editor-monospace-font` and `--ui-font` - all font *families*. Every size is hardcoded in `/.client/main.css`. Two declarations do almost all the work:

| Core rule | Default | Controls |
| --- | --- | --- |
| `#sb-main .cm-editor` | `18px` | the entire note body |
| `#sb-top .main .inner` | `28px` | the page title in the top bar |

The body is the important one: core sizes every heading in `em` (`h1` is `1.5em`, `h2` `1.2em`, `h3` `1.1em`, `h4`-`h6` `1em`), so shrinking the editor's base size scales the whole document - headings, lists, tables, code - in proportion. No per-heading rules needed.

So the shape is: **static CSS reads variables, Lua writes those variables from config.** The `space-style` block never changes; the `space-lua` block copies three config values onto `document.documentElement` as custom properties. `space-style` is plain CSS with no interpolation, which is why config cannot reach it directly.

Every `var()` carries its own fallback (`var(--sb-editor-font-size, 16px)`), so if the Lua half never runs - bridge unavailable, error during boot - the sizes are still the intended ones rather than unstyled. Failing back to "as designed" beats failing back to "core's 28px title".

An inline property on `documentElement` beats any stylesheet rule, so nothing can shadow a configured value, and `<html>` survives page navigation - it is never re-rendered - so this is set once at boot and re-asserted on page load, with no observer or resize handler.

## Two cascade landmines

**1. Both CSS rules are prefixed with `html`.** The core selectors `#sb-main .cm-editor` and `#sb-top .main .inner` are exactly as specific as anything written here, so an equal-specificity override wins only by source order - and source order across `space-style` blocks depends on page load order, which is not something to bet on. The leading `html` adds an element to the selector and settles it outright.

**2. The title rule is fenced above 600px on purpose.** [[Library/neupsh/MobileUX]] already sets `#sb-top .main .inner` inside a `max-width: 600px` block, reading `--mobile-title-font-size`. Two rules on the same selector, neither `!important`, is exactly the race above. Scoping this one to `min-width: 601px` means they never overlap - and `mobileTitleFontSize` here feeds MobileUX's existing variable rather than adding a second competing rule.

## The tree view

The Tree View panel (`Library/silverbullet-treeview/treeview.plug.js`) hardcodes `18px` on `.tree__label > span`, and it is not part of the note body, so shrinking the editor left the sidebar at full size. It is sized here too - but **through CSS only, not through config**.

The panel is a `srcdoc` iframe. The plug builds it as
`<style>${sortable-tree.css} ${treeview.css} ${customStyles}</style>`, where `customStyles` is this space's `space-style` text - which is why the rules above land inside it (and why the Tokyo Night theme colours the tree at all). What does **not** cross that boundary is the Lua half: custom properties set inline on the parent document's `<html>` are invisible to a child document, verified live - inside the panel `--sb-editor-font-size` reads as empty while the parent has `16px`.

So a `treeviewFontSize` config key would validate, parse, and change nothing. Rather than ship a setting that silently does nothing, the size lives in the `var()` fallback: **edit `var(--sb-treeview-font-size, 14px)` in the `space-style` block below.** The variable name is kept so that if a future SB version pipes CSS variables into panels, the Lua block starts driving it with no CSS change.

`14px` against the body's `16px` reads as secondary, which is what a navigation sidebar should be.

One companion rule: SortableTree's `--st-collapse-icon-height` is `2.1rem`, and `rem` inside the iframe is the browser default `16px`, not the tree's font size. Left alone it pins folder rows near their old height while page rows shrink, so the list comes out visibly ragged. It is scaled down with the text.

## What this deliberately does not touch

The UI chrome - modals, the command palette, buttons, panels - is a mix of `px` and `rem` and is already proportionate to the action icons. Shrinking it globally with `html { font-size: … }` would move the `rem`-based half and leave the `px`-based half behind, so the parts would drift out of proportion rather than get uniformly smaller. If the palette specifically feels too big, size that one component instead.

```space-lua
-- priority: 100
config.define("textDensity", {
  type = "object",
  properties = {
    editorFontSize = schema.string(),
    titleFontSize = schema.string(),
    mobileTitleFontSize = schema.string(),
  }
})
```

```space-lua
-- priority: -100
-- Copy config values onto <html> as CSS custom properties, which the
-- space-style block below reads. Static CSS, dynamic variables.
--
-- Negative priority so this evaluates AFTER the config.set blocks it reads.
-- At priority 0 it would race them and could apply defaults on the very first
-- paint (editor:pageLoaded would then correct it - a visible flicker rather
-- than a wrong result, but avoidable).
function applyTextDensity()
  -- Guard: only meaningful in a browser. Skip if this ever evaluates off-DOM.
  if not (js and js.window and js.window.document) then return end
  local root = js.window.document.documentElement

  local function set(property, key, default)
    local value = config.get("textDensity." .. key, default)
    -- A bare number is a CSS length only with a unit; 16 means 16px.
    if type(value) == "number" then value = tostring(value) .. "px" end
    root.style.setProperty(property, value)
  end

  set("--sb-editor-font-size", "editorFontSize", "16px")
  set("--sb-title-font-size", "titleFontSize", "20px")
  set("--mobile-title-font-size", "mobileTitleFontSize", "20px")
end

-- <html> exists before any page renders, so unlike the top-bar chrome this
-- can run at evaluation time - no waiting for a mount.
applyTextDensity()

event.listen {
  name = "editor:pageLoaded",
  run = applyTextDensity
}
```

```space-style
/* Note body. Core: 18px. Headings are em-relative, so this scales the
   whole document with it. `html` prefix - see landmine 1 above. */
html #sb-main .cm-editor {
  font-size: var(--sb-editor-font-size, 16px);
}

/* Page title. Core: 28px. Fenced above the 600px breakpoint so it cannot
   race MobileUX's --mobile-title-font-size rule - see landmine 2. */
@media only screen and (min-width: 601px) {
  html #sb-top .main .inner {
    font-size: var(--sb-title-font-size, 20px);
  }
}

/* Tree View panel. The plug hardcodes 18px here. This rule reaches inside the
   panel iframe, but the Lua half above does not - see "The tree view" section.
   Edit the fallback, not config, to change it. */
html .tree__label > span {
  font-size: var(--sb-treeview-font-size, 14px);
}

/* The collapse arrow's box is rem-sized (2.1rem = 33.6px), so it would hold
   folder rows at their old height while page rows shrank. */
html .treeview-root {
  --st-collapse-icon-height: 1.6rem;
}
```
