# silverbullet-plugins

Small add-ons for [SilverBullet](https://silverbullet.md) 2.x, written as Space Lua library pages. Each one is a single Markdown page: no build step, no plug binary.

## What is here

| Page | What it does |
| --- | --- |
| `Folding` | Collapse a heading section or bullet subtree; optionally remember the fold per page. |
| `MobileUX` | Makes the command palette, page picker and other modals usable on a phone. |
| `TableEditor` | Edit Markdown tables in place in the rendered view. |
| `TextDensity` | Smaller text overall, with the sizes set under `config`. |
| `PagePickerInput` | Copies the highlighted page-picker result into the input. |
| `PlainTags` | Shows inline `#tags` as plain linked words (toggle command). |
| `ProseCopy` | Copies a selection as prose, with tags and wiki links flattened. |
| `MathDollarGuard` | Stops "$23B" from being read as inline LaTeX. Needs [Silverbullet-Math](https://github.com/Mr-xRed/Silverbullet-Math). |
| `VersionBadge` | Shows the SilverBullet version in the top bar. |
| `LinkedWidgets` | Moves the Linked Tasks widget below the page body. Changes the default layout. |
| `TreeViewLinks` | Makes [Tree View](https://github.com/joekrill/silverbullet-treeview) rows open in a new tab on Ctrl/middle-click. Needs that plug. |

Each page documents itself at the top. Read it before installing the ones that change layout (`LinkedWidgets`, `TextDensity`, `MobileUX`).

## Install a page

SilverBullet's built-in Share feature pulls a page from GitHub. Do this once per page:

1. Create a page named `Library/neupsh/<Name>`, for example `Library/neupsh/Folding`. Keep that exact path: `TableEditor` and a few others read their own page by name.
2. Put this at the top, with the page name in the URL:

   ```yaml
   ---
   share.uri: "github:neupsh/silverbullet-plugins/Folding.md"
   share.mode: pull
   ---
   ```
3. Run the command **Share: Page** from the command palette. SilverBullet asks whether to overwrite your local page with the remote one - choose **Ok**. The page is replaced with the published version and a `share.hash` line is filled in.
4. Run **System: Reload** (`Ctrl-Alt-r`).

## Update

Open an installed page and run **Share: Page** again. It pulls only when the published page changed, and asks before overwriting local edits. Run **System: Reload** afterwards.

## Install everything

Create each page above as in "Install a page", or script it against the space folder:

```bash
cd /path/to/your/space && mkdir -p Library/neupsh
for n in Folding MobileUX TableEditor TextDensity PagePickerInput PlainTags ProseCopy MathDollarGuard VersionBadge LinkedWidgets TreeViewLinks; do
  printf -- '---\nshare.uri: "github:neupsh/silverbullet-plugins/%s.md"\nshare.mode: pull\n---\n' "$n" > "Library/neupsh/$n.md"
done
```

Then open each page once, run **Share: Page** and choose **Ok**, then run **System: Reload**.

## Compatibility

Written and tested against SilverBullet 2.10 and 2.11. `MobileUX`, `TableEditor`, `PagePickerInput` and `TreeViewLinks` rely on internal editor markup that SilverBullet does not promise to keep, so a SilverBullet upgrade can break them.

## Licence

MIT.
