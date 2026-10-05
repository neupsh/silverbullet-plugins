---
name: Library/neupsh/PageCreate
tags: meta/library
description: "A Page: Create Page command, with toolbar buttons to create and delete a page."
---
# Page: Create Page

SilverBullet ships no command to create a page from scratch. It is otherwise only possible by typing a name that does not exist into the page picker. This adds `Page: Create Page`, which asks for a name and opens it.

The prompt starts with the current page's folder, since creating a page next to the one you are on is the common case. Clear it to create at the top level. The new page is an empty editor and the file is written on your first keystroke.

Installing this page also adds a delete button (the built-in `Page: Delete`) next to it.

```space-lua
command.define {
  name = "Page: Create Page",
  run = function()
    -- Strings from the bridge carry no string metatable, so the function forms of
    -- `string.*` are used rather than method calls.
    local current = editor.getCurrentPage() or ""
    local folder = string.match(current, "^(.*/)") or ""
    local name = string.trim(some(editor.prompt("Page name", folder)) or "")
    if name == "" then
      return
    end
    editor.navigate(name)
  end
}
```
