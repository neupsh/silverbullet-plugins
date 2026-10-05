---
name: Library/neupsh/SyncReload
tags: meta/library
description: "A Sync and Reload command (with a toolbar button) that reloads the whole client."
---
# Sync and Reload

Adds the command `Sync and Reload`, which reloads the browser tab so SilverBullet re-reads all Space Lua and styles and reindexes. Use it after installing or updating a page. It is the same as **System: Reload** but it is one click on the toolbar and it also works on a phone, where the keyboard shortcut cannot be typed.

```space-lua
command.define {
  name = "Sync and Reload",
  run = function()
    js.window.location.reload()
  end
}
```
