---
name: Library/neupsh/FoldingRegression
tags: meta/library
---
# Folding regression checks

Install this test page alongside `Folding.md`, reload SilverBullet, and run
`Folding: Run regression checks`. It exercises the installed `folding` functions
in the real Space Lua runtime. No build tool or additional dependency is needed.

The eight checks use only synthetic text. They intercept `editor.replaceRange`,
compare exact strings, and restore it even on failure; no open page is edited.
For UI verification, use a disposable Chinese parent/child list with `123` and
`😀` before it. Remember the child and parent folds, reload, expand their markers,
then cancel both defaults and compare the entire file with its original copy.

```space-lua
function foldingRunRegressionChecks()
  assert(folding and folding.replaceMinimal, "Install Folding first")
  local passed = 0
  local previousReplace = editor.replaceRange
  local edits = {}
  editor.replaceRange = function(from, to, text)
    edits[#edits + 1] = { from = from, to = to, text = text }
  end
  local ok, failure = pcall(function()
    local function checkEdit(old, new, label)
      edits = {}
      folding.replaceMinimal(old, new)
      local result = old
      for _, edit in ipairs(edits) do
        assert(edit.from >= 0 and edit.to >= edit.from and edit.to <= #result,
          label .. ": edit range is outside the document")
        result = string.sub(result, 1, edit.from) .. edit.text
          .. string.sub(result, edit.to + 1)
      end
      assert(result == new, label .. ": edit did not produce the exact expected text")
      passed = passed + 1
    end

    local old = "---\ntitle: 测试\n---\n# 中文标题\n\n示例 123 😀\n- 中文父项\n    - 中文子项\n        - 中文孙项\n"
    local first, second = "li-1234567890", "li-2345678901"
    local added = index.patchFrontmatter(old, {
      { op = "set-key", path = "folded", value = { first } },
    })
    local changed = index.patchFrontmatter(added, {
      { op = "delete-key", path = "folded" },
      { op = "set-key", path = "folded", value = { first, second } },
    })
    local removed = index.patchFrontmatter(changed, {
      { op = "delete-key", path = "folded" },
    })
    checkEdit(old, added, "add fold")
    checkEdit(added, changed, "add nested fold after numeric ID")
    checkEdit(changed, removed, "remove folds")
    assert(removed == old, "round trip must preserve the entire original document")
    passed = passed + 1

    -- CodeMirror positions use UTF-16; cover both surrogate-boundary cases.
    for number, pair in ipairs({
      { "1234中文ABC", "1234中文XYZ" },
      { "😀前缀", "😁前缀" },
      { "𐀀尾部", "😀尾部" },
    }) do
      checkEdit(pair[1], pair[2], "boundary " .. number)
    end
    assert(folding.charLen("1234567890") == 10, "digits must count as 10 UTF-16 units")
    assert(folding.charLen("中文😀") == 4, "Chinese plus emoji must count as 4 UTF-16 units")
    passed = passed + 1
  end)
  editor.replaceRange = previousReplace
  if not ok then error(failure) end
  return "Folding regression checks passed: " .. passed
end

command.define {
  name = "Folding: Run regression checks",
  run = function()
    editor.flashNotification(foldingRunRegressionChecks())
  end,
}
```
