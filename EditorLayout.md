---
name: Library/neupsh/EditorLayout
tags: meta/library
description: "A wider editor (80%, or 95% on a phone) and headings that do not indent."
---
# Editor layout

Sets the editor to 80% of the window width (95% under 600px) and removes the text indent SilverBullet adds to heading lines.

```space-style
html {
  --editor-width: 80% !important;
}
.sb-header-inside.sb-line-h1 {
    text-indent: 0ch !important;
}
.sb-header-inside.sb-line-h2 {
    text-indent: 0ch !important;
}

.sb-header-inside.sb-line-h3 {
    text-indent: 0ch !important;
}

.sb-header-inside.sb-line-h4 {
    text-indent: 0ch !important;
}

.sb-header-inside.sb-line-h5 {
    text-indent: 0ch !important;
}

.sb-header-inside.sb-line-h6 {
    text-indent: 0ch !important;
}

@media only screen and (max-width: 600px) {
  html {
    --editor-width: 95% !important;
  }
}
```
