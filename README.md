# silverbullet-plugins

Small add-ons for [SilverBullet](https://silverbullet.md) 2.x, written as Space Lua library pages. Each one is a single Markdown page: no build step, no plug binary.

## Quick start

Create one page, `Library/neupsh/Plugins`, containing only this:

```yaml
---
share.uri: "github:neupsh/silverbullet-plugins/Plugins.md"
share.mode: pull
---
```

Run **Share: Page** (`Ctrl-p`) and choose **Ok** on the overwrite prompt. The page becomes a list of every add-on with **Install**, **Update** and **Remove** buttons, plus **Install all** and **Update all**. The same actions are in the command palette as `Plugins: Install`, `Plugins: Update`, `Plugins: Remove`, `Plugins: Install All` and `Plugins: Update All`. Each change reloads SilverBullet for you.

Keep the page path `Library/neupsh/<Name>`: several add-ons read their own page by name.

## What is here

| Page | What it does |
| --- | --- |
| `Plugins` | The manager page above. |
| `Folding` | Collapse a heading section or bullet subtree; optionally remember the fold per page. |
| `TableEditor` | Edit Markdown tables in place in the rendered view. |
| `PlainTags` | Shows inline `#tags` as plain linked words (toggle command). |
| `ProseCopy` | Copies a selection as prose, with tags and wiki links flattened. |
| `MathDollarGuard` | Stops "$23B" from being read as inline LaTeX. Needs [Silverbullet-Math](https://github.com/Mr-xRed/Silverbullet-Math). |
| `PagePickerInput` | Copies the highlighted page-picker result into the input. |
| `MobileUX` | Makes the command palette, page picker and other modals usable on a phone. |
| `TextDensity` | Smaller text overall, with the sizes set under `config`. |
| `LinkedWidgets` | Moves the Linked Tasks widget below the page body. Changes the default layout. |
| `TreeViewLinks` | Makes [Tree View](https://github.com/joekrill/silverbullet-treeview) rows open in a new tab on Ctrl/middle-click. Needs that plug. |
| `VersionBadge` | Shows the SilverBullet version in the top bar. |
| `SyncBadge` | Shows how long ago git last fetched, in the top bar. Needs a git space and shell access. |
| `PageDates` | A Created / Updated line on every page, with the created date from git. Needs a git space and shell access. |
| `JournalFeatures` | Prev/next day links, a rollup of everything scheduled for the day, and a stream page. Follows `journal.prefix`. |
| `JournalCalendar` | A month-grid picker that opens or creates any journal day. |
| `JournalPromote` | Moves a bullet and its children to their own page, leaving a linked task. |
| `JournalConventions` | A short note on working in the journal first and promoting later. |

Each page documents itself at the top. Read it before installing the ones that change layout (`LinkedWidgets`, `TextDensity`, `MobileUX`).

GitHub serves published files with a cache of about five minutes, so a push can take that long to show up as an update.

## Without the manager

Any single page installs by hand with SilverBullet's built-in Share feature: create `Library/neupsh/<Name>` with the `share.uri` and `share.mode: pull` lines above (pointing at that page's file), run **Share: Page**, choose **Ok**, then **System: Reload**.

To add a page to the repo, add its file and one row to the `plugins.catalog` list at the top of `Plugins.md`.

## Other add-ons I use

These are other people's work, not in this repo. Each installs the same way: create the page at the path shown with the two `share.*` lines, run **Share: Page**, choose **Ok**, then **System: Reload**.

| Page to create | `share.uri` | What it is |
| --- | --- | --- |
| `Library/mrmugame/Silversearch` | `ghr:MrMugame/silversearch/PLUG.md` | Full-text search. |
| `Library/mrmugame/Silverbullet-Math` | `https://github.com/MrMugame/silverbullet-math/blob/main/Math.md` | LaTeX maths. `MathDollarGuard` needs it. |
| `Library/mrmugame/Silverbullet-PDF` | `ghr:MrMugame/silverbullet-pdf/PLUG.md` | PDF viewer. |
| `Library/LogeshG5/silverbullet-excalidraw` | `https://github.com/LogeshG5/silverbullet-excalidraw/blob/main/PLUG.md` | Excalidraw drawings. |
| `Library/silverbullet-mermaid` | `https://github.com/silverbulletmd/silverbullet-mermaid/blob/main/PLUG.md` | Mermaid diagrams. |
| `Library/zefhemel/Git` | `https://github.com/zefhemel/silverbullet-libraries/blob/main/Git.md` | Git commands. |
| `Library/silverbulletmd/markdown-prettify/Prettify` | `ghr:silverbulletmd/silverbullet-markdown-prettify/Prettify.md` | Prettier rendering of Markdown. |
| `Library/Mr-xRed/DocumentExplorer` | `https://github.com/Mr-xRed/silverbullet-libraries/blob/main/DocumentExplorer.md` | Browse documents. |
| `Library/jagwarrx/Flashcards` | `github:jagwarrx/styles/flashcards.md` | Flashcards styling. |
| `Library/thepaperpilot/Copyable` | `github:thepaperpilot/silverbullet-libraries/Library/thepaperpilot/Copyable.md` | Text that copies itself on click. |
| `Library/Mr-xRed/AdvancedPanelControl` | `github:Mr-xRed/silverbullet-libraries/AdvancedPanelControl.md` | Side-panel control. Also needs `UnifiedAdvancedPanelControl.js` from the same repo saved as `Library/Mr-xRed/UnifiedAdvancedPanelControl.js`. |

Example, for Silversearch:

```yaml
---
share.uri: "ghr:MrMugame/silversearch/PLUG.md"
share.mode: pull
---
```

**Tree View** is a compiled plug, not a library page. Add this to the Space Lua block in `CONFIG.md`, then run **Plugs: Update** and reload:

```lua
config.set {
  plugs = {
    "github:joekrill/silverbullet-treeview/treeview.plug.js"
  }
}
```

The last two `github:` addresses in the table are my best reading of where those pages live and were not run end to end. If one fails, open its repo and copy the file path.

## Compatibility

Written and tested against SilverBullet 2.10 and 2.11. `MobileUX`, `TableEditor`, `PagePickerInput` and `TreeViewLinks` rely on internal editor markup that SilverBullet does not promise to keep, so a SilverBullet upgrade can break them.

## Licence

MIT.
