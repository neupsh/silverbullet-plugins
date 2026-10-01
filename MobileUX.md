---
name: Library/neupsh/MobileUX
tags: meta/library
description: "Makes SilverBullet 2.x modals (command palette, page picker, Silversearch) usable on a phone."
---

# Mobile UX

Fixes for SilverBullet 2.x on phones. Drop this page into a space, run **System: Reload**, done - it is pure `space-style`, no plug and no build step.

## What it fixes

**1. Modals hidden underneath the top bar.** SilverBullet core gives `#sb-top` `z-index: 105`, but gives `.sb-modal-backdrop` only `z-index: 100`. On a phone the top bar is 56px tall and modals are pinned to `inset: 8px`, so the top bar paints *over* the first row of every panel-based modal - which is exactly where the query input lives.

This is why [[Library/mrmugame/Silversearch]] looks broken on mobile: you see the help text and the tips box but never the search field, and tapping where the field should be focuses the top bar's page-name editor instead. That misdirected tap is also why the keyboard never comes up - you are focusing the wrong input.

Measured on a Pixel-class viewport, before and after: `document.elementFromPoint()` at the search input's own coordinates returned `INPUT.sb-page-name-editor` before, and the Silversearch `IFRAME` after.

**2. Result lists capped at 250px.** Core sets `.sb-result-list { max-height: 250px }` regardless of screen size, so a phone with 800px of vertical space shows three or four results and wastes the rest. Here the list grows to fill the screen and scrolls.

**3. Modals stretched to full height even when nearly empty.** The dialog is `position: fixed` with `top: 0; bottom: 0`, so it has to be released with `bottom: auto` before `height: auto` will size it to content. Without that, a palette with three matches still occupies the whole screen.

**4. Touch targets and input zoom.** Rows go to ~45px. Inputs go to 16px, below which mobile browsers zoom the page on focus and leave it zoomed.

**5. The keyboard not opening when a search/command dialog does.** Separate bug from #1, and it survives after the input is visible. Android Chrome (and iOS Safari) only raise the virtual keyboard when an editable element is focused *inside a user gesture*. SilverBullet focuses the dialog's input asynchronously, after the modal has rendered, which is outside that window - so you get a blinking cursor in a field that is genuinely focused, and no keyboard. Tapping the field by hand works, because that *is* a gesture.

The fix is the standard focus-keeper handoff, in the `space-lua` block below: when you tap an action button that is about to open a text dialog, focus a 1px offscreen input **synchronously during the tap** - which is what makes the keyboard open - then hand focus to the dialog's real input as soon as it exists. The keyboard never closes in between, so it looks like it simply opened with the dialog.

**6. A long page name you cannot read.** Core renders the title at 28px, so on a 412px phone a path like `Library/author/some-long-page-name` is cut off. The title is an `<input>`, and an input only scrolls its text once it has focus - so reading the tail meant tapping the field, which opens the keyboard and starts a rename you did not ask for. Worse, an ellipsis stays painted at the edge while you scroll, so it reads as "this is all there is". Fixed by making the *container* scroll instead of the input: a touch drag now pans the full name with nothing focused, and the edge fade disappears once you reach the end.

## What it does not fix

Modifier chords (`Ctrl-Alt-r` and friends) cannot be typed on a phone keyboard - that is a hardware limitation, not a SilverBullet bug. Reach commands through the **Open Command Palette** action button instead, and give the commands you use most an `actionButtons` entry in `CONFIG.md`. With the palette's input now visible and tappable, typing a command name works.

## The styles

```space-style
/* priority: 10 */

/* ---- Mobile modals: palette, page picker, prompts, Silversearch --------- */
/* Scoped to core's own 600px phone breakpoint, so desktop is untouched.     */
@media only screen and (max-width: 600px) {

  /* 1. Lift modals above the 56px top bar (core: #sb-top is z-index 105,
        .sb-modal-backdrop is only 100). Without this the query input of any
        panel-based modal is covered, and taps land on the page-name editor. */
  .sb-modal-backdrop {
    z-index: 110;
  }

  /* 2. Size modals to their content, capped at the visible viewport.
        `bottom: auto` is load-bearing: the dialog is position:fixed with
        top:0/bottom:0, so height:auto alone still stretches it full-height.
        100dvh (not 100vh) so the box shrinks when the keyboard opens. */
  .sb-modal-box {
    display: flex;
    flex-direction: column;
    height: auto;
    bottom: auto;
    max-height: calc(100dvh - 16px);
  }

  .sb-modal-box .sb-header,
  .sb-modal-box .sb-prompt,
  .sb-modal-box .sb-help-text {
    flex: 0 0 auto;
  }

  /* 3. Results fill the phone screen instead of core's flat 250px cap. */
  .sb-modal-box .sb-result-list {
    flex: 0 1 auto;
    min-height: 0;
    max-height: none;
    overflow-y: auto;
    overscroll-behavior: contain;
    -webkit-overflow-scrolling: touch;
  }

  /* 4. Touch targets ~45px, and 16px inputs so focusing does not zoom. */
  .sb-modal-box .sb-option,
  .sb-modal-box .sb-selected-option {
    padding: 12px 10px;
    line-height: 21px;
  }

  .sb-modal-box .sb-header {
    padding: 10px;
  }

  .sb-modal-box .sb-header input,
  .sb-modal-box .sb-header .sb-input,
  .sb-modal-box .sb-prompt .sb-input,
  .sb-modal-box .sb-mini-editor,
  .sb-modal-box .sb-mini-editor .cm-content {
    font-size: 16px;
  }

  /* Keyboard-shortcut hints cannot be used on a phone and crowd the row. */
  .sb-modal-box .sb-result-list .sb-hint {
    display: none;
  }

  /* Prompt dialogs (Create Page, rename, ...) get the same treatment. */
  .sb-modal-box .sb-prompt .sb-prompt-buttons button {
    min-height: 44px;
    padding: 0 16px;
    font-size: 16px;
  }

  /* 5. Page title. Core renders it at 28px, so on a 412px phone a path like
        "Library/author/some-long-page-name" showed about half of its characters.
        Override the size with --mobile-title-font-size if 14px is too small
        for you. Note 16px is the threshold below which mobile browsers zoom
        the page when you focus an input, and this title IS an input - the
        zoom is suppressed here by SilverBullet's own viewport meta
        (user-scalable=no, maximum-scale=1.0), but raise this to 16px if you
        ever hit a browser that ignores that.

        Set on .inner rather than on the input itself: core declares
        #sb-top .main #sb-current-page .sb-input { font-size: inherit }, and
        two IDs outrank anything reasonable you can write for the input, so
        the only way to move the size is to change what it inherits from. */
  #sb-top .main .inner {
    font-size: var(--mobile-title-font-size, 20px);
  }

  /* At 20px most everyday page names fit outright. The ones that don't get
     an ellipsis rather than a hard clip mid-glyph - but on its own that is a
     dead end, because the title is an <input> and an input only scrolls its
     text once it has focus. Reading the tail of a long path therefore meant
     tapping the field, which opens the keyboard and drops you into a rename
     you did not ask for.

     So the ellipsis is the no-JS fallback only. When the script below has
     run it marks the box `sb-title-scroll`, and from there the *container*
     scrolls instead of the input: the input is sized to its own content, so
     a plain touch drag pans the name with no focus and no keyboard. The
     "there is more" hint becomes a mask fade that disappears once you reach
     the end - an ellipsis cannot do that, it stays put while you scroll. */
  #sb-top .main #sb-current-page .sb-input {
    text-overflow: ellipsis;
  }

  #sb-top .main #sb-current-page.sb-title-scroll {
    display: block;
    overflow-x: auto;
    overscroll-behavior-x: contain;
    scrollbar-width: none;
    -webkit-overflow-scrolling: touch;
  }

  #sb-top .main #sb-current-page.sb-title-scroll::-webkit-scrollbar {
    display: none;
  }

  /* Only while there is something still off to the right. */
  #sb-top .main #sb-current-page.sb-title-scroll.sb-title-more {
    -webkit-mask-image: linear-gradient(to right, #000 calc(100% - 28px), transparent 100%);
    mask-image: linear-gradient(to right, #000 calc(100% - 28px), transparent 100%);
  }

  /* min-width keeps a short name filling the bar, so tapping anywhere in the
     row still focuses the editor exactly as it did before. */
  #sb-top .main #sb-current-page.sb-title-scroll .sb-input {
    text-overflow: clip;
    min-width: 100%;
  }

  /* 6. Roomier action buttons in the top-bar hamburger, and keep the last
        one clear of the home-gesture bar on gesture-nav phones. */
  #sb-top .sb-actions.hamburger button:not(.sb-code-copy-button) {
    height: 2.1rem;
    padding: 5px 0;
  }

  #sb-top .sb-actions.hamburger.open {
    padding-bottom: env(safe-area-inset-bottom, 0px);
  }
}
```

## The keyboard assist

Which buttons arm the keyboard is matched against the button's **tooltip** (SilverBullet renders an action button's `description` as its `title`; the command name is not exposed in the DOM). Override the pattern for your own button labels with `config.set("mobileUX.inputButtons", "...")` - it is a JavaScript regex source string, matched case-insensitively.

Arming only happens on phone-sized, coarse-pointer viewports, so desktop is never touched.

```space-lua
-- priority: 10

-- Hand the virtual keyboard from the tap to the dialog it opens.
-- Only meaningful in a browser; skip if this ever evaluates off-DOM.
if js and js.window and js.window.document then
  local pattern = config.get("mobileUX.inputButtons",
    "search|command|palette|open page|create|rename|find|picker|prompt")

  -- Injected as a global first, so the body below stays a plain JS literal.
  -- The pattern must not contain a double quote.
  js.window.eval('window.__sbMobileKbPattern = "' .. pattern .. '";')

  js.window.eval([==[
(function () {
  var W = window, D = document;
  if (W.__sbMobileKbAssist) return;      // space-lua can re-evaluate; install once
  W.__sbMobileKbAssist = true;

  // Phones with a virtual keyboard only.
  if (!W.matchMedia("(max-width: 600px)").matches) return;
  if (!W.matchMedia("(pointer: coarse)").matches) return;

  var OPENS_INPUT = new RegExp(W.__sbMobileKbPattern, "i");
  var SEL = '.sb-modal-box input, .sb-modal-box textarea, .sb-modal-box [contenteditable="true"]';
  var WINDOW_MS = 2500, POLL_MS = 60;
  var keeper = null, armedUntil = 0, poll = null;

  function getKeeper() {
    if (keeper && keeper.isConnected) return keeper;
    keeper = D.createElement("input");
    keeper.type = "text";
    keeper.tabIndex = -1;
    keeper.setAttribute("aria-hidden", "true");
    // Must stay a real, focusable, on-screen input: display:none or
    // visibility:hidden would stop the keyboard from opening at all.
    keeper.style.cssText = "position:fixed;top:0;left:0;width:1px;height:1px;" +
      "opacity:0;border:0;padding:0;margin:0;font-size:16px;background:transparent;";
    D.body.appendChild(keeper);
    return keeper;
  }

  // The dialog's own field, which may live inside a panel iframe (Silversearch).
  function modalInput() {
    var el = D.querySelector(SEL);
    if (el) return el;
    var frames = D.querySelectorAll("iframe");
    for (var i = 0; i < frames.length; i++) {
      var d = null;
      try { d = frames[i].contentDocument; } catch (e) { continue; }
      if (!d) continue;
      el = d.querySelector(SEL);
      if (el) return el;
    }
    return null;
  }

  function matched(target) {
    var btn = target && target.closest ? target.closest("#sb-top .sb-actions button") : null;
    return btn && OPENS_INPUT.test(btn.title || "") ? btn : null;
  }

  function isEditable(el) {
    if (!el || !el.tagName) return false;
    var tag = el.tagName;
    return tag === "INPUT" || tag === "TEXTAREA" || el.isContentEditable === true;
  }

  function disarm(alsoBlur) {
    armedUntil = 0;
    if (poll) { clearInterval(poll); poll = null; }
    if (alsoBlur && keeper && D.activeElement === keeper) keeper.blur();
  }

  // Step 2: give the already-open keyboard to the dialog's real input.
  function handOver() {
    if (!armedUntil || Date.now() > armedUntil) { disarm(true); return; }
    var input = modalInput();
    if (!input) return;
    if (input.ownerDocument.activeElement !== input) {
      try { input.focus({ preventScroll: true }); } catch (e) {}
    }
    disarm(false);
  }

  // Step 1: focus a real input synchronously inside the tap. This is the only
  // moment the browser will agree to open the keyboard.
  D.addEventListener("pointerdown", function (e) {
    if (!matched(e.target)) return;
    if (modalInput()) return;            // a dialog is already open
    getKeeper().focus({ preventScroll: true });
    armedUntil = Date.now() + WINDOW_MS;
    if (poll) clearInterval(poll);
    poll = setInterval(handOver, POLL_MS);
  }, true);

  // A button takes focus on mousedown, and focusing a non-editable element is
  // exactly what dismisses the keyboard we just opened. Cancelling the default
  // suppresses that focus; the button's click still fires normally.
  D.addEventListener("mousedown", function (e) {
    if (matched(e.target)) e.preventDefault();
  }, true);

  // Safety net for anything else that grabs focus between the tap and the
  // dialog: hold the keyboard open by bouncing focus back to the keeper.
  D.addEventListener("focusin", function (e) {
    if (!armedUntil || Date.now() > armedUntil) return;
    if (e.target === keeper || isEditable(e.target)) return;
    if (modalInput()) return;
    getKeeper().focus({ preventScroll: true });
  }, true);
})();
  ]==])
end
```

## The scrollable page title

A long page name is truncated in the top bar. Making the *input* scroll needs focus, and focus on a phone means the keyboard and a rename you did not want. This sizes the input to its content and lets the surrounding box scroll instead, so a touch drag pans the name with nothing focused.

The fade at the right edge is the "there is more" hint, and it is removed once you scroll to the end. Without this script the CSS above falls back to a plain ellipsis - the old behaviour, not a broken one.

```space-lua
-- priority: 10

-- Touch-scrollable page title in the top bar (phones only).
if js and js.window and js.window.document then
  js.window.eval([==[
(function () {
  var W = window, D = document;
  if (W.__sbMobileTitleScroll) return;
  W.__sbMobileTitleScroll = true;

  var MQ = W.matchMedia("(max-width: 600px)");

  function box() { return D.querySelector("#sb-top .main #sb-current-page"); }

  function markMore(b) {
    var more = b.scrollWidth - b.clientWidth - b.scrollLeft > 2;
    b.classList.toggle("sb-title-more", more);
  }

  // Measure at zero width, because an input already stretched to its content
  // reports that stretched width as its scrollWidth and would only ever grow.
  function sync() {
    var b = box();
    if (!b) return;
    var inp = b.querySelector(".sb-input");
    if (!inp) return;

    if (!MQ.matches) {                       // landscape / tablet: hand it back
      b.classList.remove("sb-title-scroll", "sb-title-more");
      inp.style.width = "";
      return;
    }

    b.classList.add("sb-title-scroll");
    inp.style.width = "0px";
    inp.style.width = Math.ceil(inp.scrollWidth) + "px";
    // The new width, and the min-width:100% that a short name resolves to,
    // are only real after layout. Marking now would leave the fade showing
    // over a title that in fact fits.
    markMore(b);
    W.requestAnimationFrame(function () { markMore(b); });
    // And once more after the top bar has settled: the save/sync badge
    // appearing beside the title changes the box width after the fact.
    W.setTimeout(function () { markMore(b); }, 300);

    if (!b.__sbBound) {
      b.__sbBound = true;
      b.addEventListener("scroll", function () { markMore(b); }, { passive: true });
      // Typing a rename grows the input; keep the caret end in view.
      inp.addEventListener("input", function () {
        sync();
        b.scrollLeft = b.scrollWidth;
      });
    }
  }

  // The input element survives navigation - only its value is replaced - and
  // SilverBullet rewrites <title> on every page change, so that is the signal.
  var t = D.querySelector("title");
  if (t) new MutationObserver(sync).observe(t, { childList: true, characterData: true, subtree: true });

  // A re-rendered top bar would be a fresh node with no listeners and no width.
  var top = D.querySelector("#sb-top");
  if (top) new MutationObserver(sync).observe(top, { childList: true, subtree: true });

  W.addEventListener("resize", sync);
  W.addEventListener("orientationchange", sync);
  sync();
})();
  ]==])
end
```

## Notes

- Everything here is inside `@media (max-width: 600px)`, matching core's own phone breakpoint, and runs at `priority: 10` so it layers over core rather than fighting it.
- `space-style` propagates into SilverBullet's panel iframes, which is what lets these rules reach inside Silversearch's sandboxed modal without touching the plug.
- Related: [[Library/Styles/UI]] holds this space's non-mobile styling.
