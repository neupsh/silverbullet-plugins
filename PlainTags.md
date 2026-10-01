---
name: Library/neupsh/PlainTags
tags: meta/library
description: "Renders inline #tags as plain linked words, toggled by a command."
---

# Plain Tags

Renders inline `#tags` as ordinary words - no `#`, no pill, no border - so a sentence full of tags reads as prose while every tag stays a live link to its `tag:` page. Toggle it with the **Plain Tags: Toggle** command; the setting is remembered per browser.

Given the source `[[Tech/Kafka|Apache Kafka]] is a #distributed #event-streaming platform.`

With it **off** (SilverBullet default), the two tags render as bordered pills reading `#distributed` and `#event-streaming`. With it **on**, they render as plain underlined words - *distributed* and *event-streaming* - still linking to `tag:distributed` and `tag:event-streaming`.

(The example above is inside a code span on purpose, so this meta page does not add itself to `Tech/Kafka`'s linked mentions or to the tag index.)

## How it works

Both of SilverBullet's tag renderers put the tag name in a `data-tag-name` attribute on the anchor, without the `#`:

- **Editor** (CodeMirror): `<a class="sb-hashtag" data-tag-name="distributed"><span class="sb-hashtag-text">#distributed</span></a>`
- **Rendered markdown** (linked mentions, widgets): `<a class="hashtag sb-hashtag" data-tag-name="distributed">#distributed</a>`

So the CSS hides the literal `#tag` text and re-prints the name from `attr(data-tag-name)` in an `::after`. Verified identical on 2.9.0 and 2.10.0.

The style is always loaded and gated on `html[data-plain-tags="on"]`. The Lua command flips that attribute through the `js.window` bridge, so toggling is instant - no `System: Reload`.

## The style

```space-style
/* Plain Tags - see [[Library/neupsh/PlainTags]]. Inert unless html[data-plain-tags="on"]. */

/* Strip the pill. #sb-root is needed to outrank SilverBullet's own
   "#sb-main .cm-editor .sb-hashtag" rule, which is id-specific. */
html[data-plain-tags="on"] #sb-root a.sb-hashtag {
  background-color: transparent;
  border-color: transparent;
  border-radius: 0;
  padding: 0;
  margin: 0;
  font-size: inherit;
  color: var(--editor-wiki-link-page-color);
  text-decoration: underline;
  text-decoration-thickness: 1px;
  text-underline-offset: 2px;
}

/* Re-print the name without the "#", for both renderers. */
html[data-plain-tags="on"] #sb-root a.sb-hashtag::after {
  content: attr(data-tag-name);
}

/* Editor: the "#tag" text sits in a child span, so it can simply be removed. */
html[data-plain-tags="on"] #sb-root a.sb-hashtag > .sb-hashtag-text {
  display: none;
}

/* Rendered markdown: no child span, the text is a direct child of the <a>,
   so collapse the anchor's own font instead and restore it on the pseudo. */
html[data-plain-tags="on"] #sb-root a.sb-hashtag:not(:has(> .sb-hashtag-text)) {
  font-size: 0;
}
html[data-plain-tags="on"] #sb-root a.sb-hashtag:not(:has(> .sb-hashtag-text))::after {
  font-size: 1rem;
}
```

## The toggle

```space-lua
-- Plain Tags - render inline #tags as plain linked words. See [[Library/neupsh/PlainTags]].
local PLAIN_TAGS_KEY = "neupsh.plainTags"

-- Flip the attribute the space-style above is gated on.
-- Note: a *data attribute* rather than a class, so it can never clobber classes
-- SilverBullet or a theme puts on <html>. setAttribute via the js bridge is the
-- proven-working call shape here; "root.classList:add(...)" throws
-- "attempt to index a userdata value".
local function applyPlainTags(on)
  js.window.document.documentElement.setAttribute("data-plain-tags", on and "on" or "off")
end

command.define {
  name = "Plain Tags: Toggle",
  run = function()
    local on = not (clientStore.get(PLAIN_TAGS_KEY) == true)
    clientStore.set(PLAIN_TAGS_KEY, on)
    applyPlainTags(on)
    editor.flashNotification(on and "Plain tags on - # hidden" or "Plain tags off - # shown")
  end
}

-- Re-apply the saved choice on every client load, otherwise it resets on reload.
applyPlainTags(clientStore.get(PLAIN_TAGS_KEY) == true)
```

## Optional: a toolbar button

The toolbar in [[CONFIG]] is already crowded, so this is not wired up by default. To add it, drop this into the `actionButtons` list there:

```lua
{
  icon = 'hash',
  command = 'Plain Tags: Toggle',
  description = 'Toggle plain (#-less) tag rendering'
}
```

## Known limits

- **Copy/paste still yields `#distributed`.** This is the one thing the style cannot fix. CodeMirror serialises the *document source* to the clipboard, not the rendered DOM, so selecting a sentence in the editor and pressing Ctrl-C gives you the raw Markdown - `#tags`, `[[wiki links]]` and all - no matter what the CSS does. Verified directly. Getting prose-with-links out of a page for a blog post is an **export** problem, not a styling one.
- **The `#` becomes a zero-width character in the editor.** It still occupies a position in the document, so arrowing across a tag takes one invisible extra step, and typing at the very start of a tag name is not visible until you move off. Clicking a tag navigates to its `tag:` page anyway (it is an anchor), so this was never a place you could click the caret into. Toggle the feature off while doing heavy tag editing.
- **Side-panel iframes do not follow the toggle.** `space-style` reaches them (see [[Library/neupsh/MobileUX]]), but `data-plain-tags` is set on the *main* document's `<html>`, and an iframe has its own. Linked mentions render inline in the main document, so those do follow.
- **A line that is only a tag** (SilverBullet styles `#tag`-only H1 lines specially) loses its `#` too, which is usually what you want but is a visible change.
- The setting lives in `clientStore`, which is per browser profile - it does not sync across devices.
