---
name: Library/neupsh/LinkedWidgets
tags: meta/library
description: "Moves the Linked Tasks widget below the page body so it no longer sandwiches what you type."
---

# Linked widgets placement

Std ships two page widgets: **Linked Tasks** on `hooks:renderTopWidgets` and **Linked Mentions** on `hooks:renderBottomWidgets`. That sandwiches the page body between them, so anything typed while adding objects that link to a concept lands *between* two widget boxes. This page moves Linked Tasks down to the bottom slot, directly above Linked Mentions, and puts breathing room above the pair.

How it works:

- `config.set("std.widgets.linkedTasks.enabled", false)` at `-- priority: 100` kills std's top-slot registration - std reads that key from a block at `-- priority: -1`, i.e. much later, so the disable always wins the race.
- A replacement listener on `hooks:renderBottomWidgets`, registered from the same priority-100 block, therefore runs *before* std's linked-mentions listener. `hooks:renderBottomWidgets` is dispatched into an `ArrayWidget` that renders every listener's result in dispatch order, so tasks render above mentions.
- The task query is copied from std rather than calling `widgets.linkedTasks()`, for one reason: std's version returns `widget.new{markdown = ""}` even when there are no tasks. The bottom container has `border: 1px solid` and `min-height: 48px`, so delegating would paint an empty bordered box on every page. Returning nothing keeps the slot clean.

Linked Mentions itself is left entirely alone - std still owns it.

```space-lua
-- priority: 100
-- Move std's "Linked Tasks" out of the top slot (see the prose above).
config.set("std.widgets.linkedTasks.enabled", false)

event.listen {
  name = "hooks:renderBottomWidgets",
  run = function()
    local pageName = editor.getCurrentPage()
    local tasks = query[[
      from t = index.tasks()
      where not t.done and table.includes(t.ilinks, pageName)
      order by t.page
      select templates.taskItem(t)
    ]]
    if #tasks == 0 then
      -- No empty bordered box on the vast majority of pages.
      return
    end
    return widget.new {
      markdown = "# Linked Tasks\n" .. table.concat(tasks)
    }
  end
}
```

```space-style
/* Std's own rule is `#sb-main .cm-editor .sb-lua-bottom-widget { margin: 10px 0 0 0 }` - the full
   selector is needed to outrank it on specificity. Both widgets share this one container, so this
   is the gap between the page body and the Linked Tasks heading. */
#sb-main .cm-editor .sb-lua-bottom-widget {
  margin-top: 2.5rem;
}
```
