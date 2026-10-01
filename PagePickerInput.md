---
name: Library/neupsh/PagePickerInput
tags: meta/library
description: "Mirrors the highlighted page-picker result into the input so you can edit from it."
---

# Page Picker Input

SilverBullet's built-in Open page dialog filters by the text you typed, but arrowing through the result list does not copy the highlighted page name back into the input. That is awkward when creating a sibling page in a deep folder: you can find `Very/Deeply/Nested/OldFile`, but then you still have to type `Very/Deeply/Nested/InterestingFile` by hand.

This page mirrors the highlighted result into the picker input after navigation keys. It does **not** dispatch an `input` event while mirroring, so SilverBullet's own internal query and selected row stay intact: pressing `Enter` still opens the highlighted result. If you edit the mirrored text, the next real browser `input` event hands control back to SilverBullet with the edited value, so `Shift-Enter` creates the page from that path.

The basename is selected after mirroring. So from a result like `Very/Deeply/Nested/OldFile`, pressing `Backspace` leaves `Very/Deeply/Nested/`, and typing replaces just `OldFile`.

```space-lua
-- priority: 10

-- Mirror the selected Open-page result into the visible input after keyboard
-- navigation, without firing `input` and disturbing SilverBullet's own picker
-- state. Verified against SilverBullet 2.11's `.sb-nav-input` picker DOM (2.10 used `.sb-modal-box .sb-filter-input`).
if js and js.window and js.window.document then
  js.window.eval([==[
(function () {
  var W = window, D = document;
  if (W.__sbPagePickerInputMirror) return;
  W.__sbPagePickerInputMirror = true;

  function pickerFor(target) {
    if (!target || !target.closest) return null;
    var input = target.closest(".sb-nav-root-modal .sb-nav-input");
    if (!input) return null;
    var box = input.closest(".sb-nav-root-modal");
    if (!box) return null;
    var label = box.querySelector(".sb-nav-title");
    if (!label || label.textContent.trim() !== "Open") return null;
    return { box: box, input: input };
  }

  function isNavKey(e) {
    if (e.isComposing) return false;
    if ((e.ctrlKey || e.metaKey) && (e.key === "n" || e.key === "p")) return true;
    return e.key === "ArrowUp" || e.key === "ArrowDown" || e.key === "PageUp" || e.key === "PageDown" || e.key === "Home" || e.key === "End";
  }

  function selectedName(box) {
    // The "Create" row echoes what was typed, so mirroring it would do nothing useful.
    var selected = box.querySelector(".sb-nav-row.sb-nav-selected:not(.sb-nav-create) .sb-nav-primary");
    if (!selected) return "";
    return selected.textContent.trim();
  }

  function selectBasename(input) {
    var slash = input.value.lastIndexOf("/");
    var start = slash >= 0 ? slash + 1 : 0;
    try { input.setSelectionRange(start, input.value.length); } catch (e) {}
  }

  function mirror(picker) {
    if (!picker.input.isConnected || !picker.box.isConnected) return;
    if (D.activeElement !== picker.input) return;
    var name = selectedName(picker.box);
    if (!name || picker.input.value === name) return;
    picker.input.value = name;
    selectBasename(picker.input);
  }

  function mirrorAfterPickerSettles(picker) {
    W.setTimeout(function () {
      mirror(picker);
      W.requestAnimationFrame(function () { mirror(picker); });
    }, 0);
  }

  D.addEventListener("keydown", function (e) {
    if (!isNavKey(e)) return;
    var picker = pickerFor(e.target);
    if (!picker) return;
    mirrorAfterPickerSettles(picker);
  }, true);
})();
  ]==])
end
```
