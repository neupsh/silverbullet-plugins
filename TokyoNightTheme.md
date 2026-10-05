---
name: Library/neupsh/TokyoNightTheme
tags: meta/library
description: "A Tokyo Night inspired dark theme: colours, scrollbars, tags and modals."
---
# Tokyo Night theme

A dark theme based on [this community theme](https://community.silverbullet.md/t/tokyo-night-inspired-theme/3469/10). It only applies in dark mode (`html[data-theme="dark"]`) and sets the colour variables for the top bar, modals, editor, tables, code and tags.

```space-style
::-webkit-scrollbar {
  width: 15px;
  background: rgba(26, 27, 38, 0.4);
}

::-webkit-scrollbar-track {
  background: transparent;
  border-radius: 15px;
}

::-webkit-scrollbar-thumb {
  -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,0.2);
  background: rgba(122, 162, 247, 0.3);
}

html[data-theme="dark"] {
  color-scheme: dark;
  --ui-accent-color: #7aa2f7;
  --highlight-color: rgba(224, 175, 104, 0.35);
  --link-color: #7aa2f7;
  --link-missing-color: #f7768e;
  --meta-color: #e0af68;
  --meta-subtle-color: #565f89;
  --subtle-color: #565f89;
  /* code block background, etc. */
  /* --subtle-background-color: rgba(30, 40, 59, 0.4); */
  --subtle-background-color: rgba(26, 27, 39, 0.4);
  --root-background-color: #16161f;
  --root-color: #c0caf5;
  --top-color: #c0caf5;
  --top-background-color: #1a1b26;
  --top-border-color: #24283b;
  --top-sync-error-color: var(--top-color);
  --top-sync-error-background-color: #f7768e;
  --top-saved-color: #9ece6a;
  --top-unsaved-color: #565f89;
  --top-loading-color: #565f89;

  --panel-background-color: #24283b;
  --panel-border-color: #24283b;
  --bhs-background-color: #24283b;
  --bhs-border-color: #565f89;

  --modal-color: #c0caf5;
  --modal-background-color: #1a1b26;
  --modal-border-color: #24283b;
  --modal-header-label-color: var(--ui-accent-color);
  --modal-help-background-color: #181926;
  --modal-help-color: #565f89;
  --modal-selected-option-background-color: var(--ui-accent-color);
  --modal-selected-option-color: #1a1b26;
  --modal-hint-background-color: #24283b;
  --modal-hint-color: #c0caf5;
  --modal-description-color: #565f89;

  --notifications-background-color: #181926;
  --notifications-border-color: #24283b;
  --notification-info-background-color: #7dcfff;
  --notification-error-background-color: #f7768e;

  --action-button-background-color: transparent;
  --action-button-color: #565f89;
  --action-button-hover-color: #7aa2f7;
  --action-button-active-color: #bb9af7;

  --editor-caret-color: #c0caf5;
  --editor-selection-background-color: rgba(122, 162, 247, 0.25);
  --editor-panels-bottom-color: #9ece6a;
  --editor-panels-bottom-background-color: #1a1b26;
  --editor-panels-bottom-border-color: #24283b;

  --editor-completion-detail-color: #565f89;
  --editor-completion-detail-selected-color: #c0caf5;
  --editor-list-bullet-color: #565f89;
  --editor-heading-color: #bb9af7;
  --editor-heading-meta-color: var(--meta-subtle-color);
  --editor-hashtag-background-color: rgba(122, 162, 247, 0.2);
  --editor-hashtag-color: #c0caf5;
  --editor-hashtag-border-color: rgba(122, 162, 247, 0.4);
  --editor-ruler-color: #24283b;

  --editor-naked-url-color: var(--link-color);
  --editor-link-color: var(--link-color);
  --editor-link-url-color: var(--link-color);
  --editor-link-meta-color: var(--meta-subtle-color);
  --editor-wiki-link-page-background-color: rgba(122, 162, 247, 0.08);
  --editor-wiki-link-page-color: var(--link-color);
  --editor-wiki-link-page-missing-color: var(--link-missing-color);
  --editor-wiki-link-color: #bb9af7;
  --editor-named-anchor-color: var(--meta-subtle-color);

  --editor-command-button-color: #c0caf5;
  --editor-command-button-background-color: #24283b;
  --editor-command-button-hover-background-color: #565f89;
  --editor-command-button-meta-color: var(--meta-subtle-color);
  --editor-command-button-border-color: #565f89;

  --editor-line-meta-color: var(--meta-subtle-color);
  --editor-meta-color: var(--meta-color);

  --editor-table-head-background-color: rgba(26, 27, 39, 0.5);
  --editor-table-head-color: #c0caf5;
  --editor-table-even-background-color: rgba(26, 27, 39, 0.3);

  --editor-blockquote-background-color: var(--subtle-background-color);
  --editor-blockquote-color: var(--subtle-color);
  --editor-blockquote-border-color: #565f89;

  --editor-code-background-color: var(--subtle-background-color);
  --editor-struct-color: #f7768e;
  --editor-highlight-background-color: var(--highlight-color);
  --editor-code-comment-color: var(--meta-subtle-color);
  --editor-code-variable-color: #7aa2f7;
  --editor-code-typename-color: #9ece6a;
  --editor-code-string-color: #bb9af7;
  --editor-code-number-color: #bb9af7;
  --editor-code-info-color: var(--subtle-color);
  --editor-code-atom-color: #7dcfff;

  --editor-admonition-note-border-color: #7dcfff;
  --editor-admonition-note-background-color: rgba(125, 207, 255, 0.2);
  --editor-admonition-warning-border-color: #e0af68;
  --editor-admonition-warning-background-color: rgba(224, 175, 104, 0.2);

  --editor-frontmatter-background-color: rgba(26, 27, 39, 0.5);
  --editor-frontmatter-color: var(--subtle-color);
  --editor-frontmatter-marker-color: #c0caf5;
  --editor-widget-background-color: rgba(26, 27, 39, 0.5);
  --editor-task-marker-color: var(--subtle-color);

  --action-button-color: #9d7cd8;
  --treeview-folder-color: var(--action-button-color);
  --notification-info-background-color: var(--action-button-color);
}

#sb-main .cm-textfield {
  background-color: #181926;
}

#sb-main .cm-button {
  background-image: linear-gradient(#24283b, #565f89);
}

#sb-current-page .cm-line {
  color: var(--action-button-color);
  background-color: transparent;
}

.cm-line .sb-hashtag-text {
  color: var(--action-button-color);
}

#sb-main ::-webkit-scrollbar {
  background-color: #1a1b26;
  color: #565f89;
}

.sb-quote {
  color: var(--root-color);
}

/* Make the x-ray tooltip readable on the dark background. */
.cm-tooltip {
  background-color: var(--modal-background-color);
}
```
