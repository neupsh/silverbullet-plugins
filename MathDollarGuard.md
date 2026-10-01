---
name: Library/neupsh/MathDollarGuard
tags: meta/library
description: "Stops a plain dollar amount ("$23B in 2021") from being swallowed as inline LaTeX."
---

# Math dollar guard

[[Library/mrmugame/Silverbullet-Math]] defines inline math as `$ ... $`, with a start marker of `\$(?!\{)` - that is, **any** `$` not followed by `{`. Prose about money has dollars in it, so a sentence like

    TPV of $23B in 2021 and $5M later.

pairs the two dollar signs and renders `23B in 2021 and` as an italic KaTeX expression. The words disappear into a formula. A lone `$` is worse: inline mode does not stop at the end of the line, so one unmatched dollar sign reaches forward and eats the *next* line's math delimiter along with everything in between.

This page redefines `LatexInline` with the standard dollar-math heuristic instead: **an opening `$` must not be followed by whitespace or a digit.** `$23B` and `$5M` are then plain text, while `$E = mc^2$` still renders.

The upstream library is `share.mode: pull`, so it is left untouched - overriding by name from a lower-priority block is what keeps the next pull from reverting this.

## Cost of the rule

A formula that genuinely *starts* with a digit - `$2x + 1$` - no longer renders inline. Write it as `$x \cdot 2 + 1$`, or use a `$$` block, which is unaffected. That is the trade for having prices work in prose, and prices are far more common here than digit-initial formulas.

`\$` still works as a per-instance escape if you want one. SilverBullet colours the escaped `$` as an escape mark, which looks like it means something - it does not, it is only syntax highlighting.

```space-lua
-- priority: -10
-- Lower priority than the library's own block, so this definition is the one
-- that lands. Same name, so it replaces rather than competes.

syntax.define {
  name = "LatexInline",
  -- Upstream: "\\$(?!\\{)". Added \s and \d to the lookahead.
  startMarker = "\\$(?![\\s\\d{])",
  endMarker = "\\$(?!\\{)",
  mode = "inline",
  startMarkerClass = "sb-latex-mark",
  bodyClass = "sb-latex-body",
  endMarkerClass = "sb-latex-mark",
  render = function(body)
    return latex.inline(body)
  end
}
```

## Notes

- Only the inline rule is touched. `LatexBlock` (`$$` on its own line) is anchored to a whole line and was never ambiguous.
- Related: [[Library/mrmugame/Silverbullet-Math]].
