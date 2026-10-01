---
name: Library/neupsh/TreeViewLinks
tags: meta/library
description: "Gives Tree View plug rows a real anchor so pages open in a new tab on Ctrl/middle-click."
---

# Tree View - open pages in a new tab

The Tree View panel's rows are not links. Each page row is a bare `<span data-node-type="page" title="Some/Page">`, so there is nothing for the browser to act on: no Ctrl/Cmd-click, no middle-click, no mobile long-press "Open in new tab". The note body does not have this problem - SilverBullet renders wiki links as real `<a class="sb-wiki-link" href="Some/Page">`, which is why Ctrl-click already works there.

This page fixes the tree by giving each page row a real anchor, which is the one change that solves desktop and mobile at once: modifier-click and middle-click are native browser behaviour on an `<a href>`, and so is the long-press context menu on a phone. Nothing here reimplements `window.open`.

## How it works

The panel is a `srcdoc` iframe, but it is **same-origin**, so the parent document can reach `iframe.contentDocument` and mutate it directly - verified live. No fork of `treeview.plug.js` is needed, and nothing is written into the plug's own file.

The anchor is inserted **inside** the existing `<span>`, wrapping its text, rather than replacing or wrapping the span. That is deliberate: the plug's click handler and SortableTree's drag machinery are bound to the span and the `<sortable-tree-node>` around it, and moving them would break navigation and drag-to-move. Wrapping the text leaves both untouched.

Click routing:

- **Plain left click** - the anchor calls `preventDefault()` and lets the event bubble to the plug's own handler, so navigation stays in-app (no full page reload, no losing editor state).
- **Ctrl / Cmd / Shift / Alt click, or middle click** - nothing is prevented, so the browser does what it always does with an anchor: new tab, new window, background tab.
- **Long press on touch** - the native context menu appears because the target is a real anchor. `-webkit-touch-callout` is re-enabled on it, since panel CSS suppresses callouts elsewhere.

The `href` is absolute (`location.origin + "/" + encodeURI(path))`, not relative. A cold URL like `http://host/Projects/Some/Page` boots straight to that page, which is what makes a new tab actually useful.

`draggable="false"` on the anchor matters: anchors are natively draggable, and without it the browser's link-drag would win over SortableTree's row drag and silently break reordering.

## Re-install and re-render

Two things churn underneath this, and both are handled:

- **The panel iframe is destroyed and rebuilt** when the tree is toggled, or on `System: Reload`. The installer therefore polls for a treeview iframe rather than binding once, and marks the *panel document* (not the parent window) as done - a rebuilt panel gets a fresh document and so is re-decorated automatically.
- **Rows re-render** on expand/collapse, filter, and page create/rename. A `MutationObserver` inside the panel document re-decorates new rows; already-decorated rows are skipped by a marker class, so nothing double-wraps. The observer is bound to `documentElement`, **not** `body` - the plug replaces the whole body on re-render, and an observer bound to `body` silently detaches when that happens. That failure was real, not theoretical: it decorated all 165 rows on one run and zero on the next, purely on timing. The poll also re-scans on every tick, so a paint the observer missed is still caught within a second.

Folder rows are skipped - `data-node-type="folder"` has no page to open.

```space-lua
-- priority: 10

-- Give Tree View page rows a real anchor, so Ctrl/Cmd-click, middle-click and
-- mobile long-press can open a page in a new tab. See the prose above.
if js and js.window and js.window.document then
  js.window.eval([==[
(function () {
  var W = window, D = document;
  if (W.__sbTreeViewLinks) return;   // space-lua re-evaluates; install once
  W.__sbTreeViewLinks = true;

  var MARK = "sb-treelink";

  function pathOf(span) {
    var t = span.getAttribute("title");
    return t && t.trim() ? t.trim() : null;
  }

  function decorate(span) {
    if (span.querySelector("a." + MARK)) return;         // already done
    var path = pathOf(span);
    if (!path) return;

    var a = D.createElement("a");
    a.className = MARK;
    a.href = W.location.origin + "/" + encodeURI(path);
    a.draggable = false;                                  // let SortableTree drag win
    a.style.cssText = "color:inherit;text-decoration:inherit;font:inherit;" +
      "-webkit-touch-callout:default;";

    // Move the row's existing children into the anchor: keeps the span, its
    // handlers, and any icon markup the plug put there.
    while (span.firstChild) a.appendChild(span.firstChild);
    span.appendChild(a);

    a.addEventListener("click", function (e) {
      // Modifier or middle click: do nothing, the browser opens a new tab.
      if (e.metaKey || e.ctrlKey || e.shiftKey || e.altKey || e.button !== 0) return;
      // Plain click: suppress navigation, let the plug handle it in-app.
      e.preventDefault();
    });
  }

  function scan(doc) {
    var rows = doc.querySelectorAll('span[data-node-type="page"]');
    for (var i = 0; i < rows.length; i++) decorate(rows[i]);
  }

  function install(doc) {
    if (!doc) return;
    if (!doc.__sbTreeViewLinks) {
      doc.__sbTreeViewLinks = true;
      // documentElement, not body: the plug replaces body wholesale on
      // re-render, which would silently detach an observer bound to it.
      new MutationObserver(function () { scan(doc); })
        .observe(doc.documentElement, { childList: true, subtree: true });
    }
    scan(doc);   // also every tick, so a paint we missed still gets caught
  }

  function findPanelDoc() {
    var frames = D.querySelectorAll("iframe");
    for (var i = 0; i < frames.length; i++) {
      var doc = null;
      try { doc = frames[i].contentDocument; } catch (err) { continue; }  // cross-origin panel
      if (doc && doc.querySelector(".treeview-root")) return doc;
    }
    return null;
  }

  // The panel is created, destroyed and rebuilt outside our control, so poll.
  // install() is idempotent per document; a rebuilt panel brings a fresh one.
  W.setInterval(function () { install(findPanelDoc()); }, 700);
})();
]==])
end
```
