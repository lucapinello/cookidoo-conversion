---
name: cookidoo-recipe-converter
description: Convert any web recipe URL into a Cookidoo created recipe for the Thermomix TM7 by driving the Cookidoo web UI in a real browser. Use this skill whenever the user mentions Cookidoo, Thermomix, TM7, the Thermomix app, or asks to convert, scrape, import, add, or save a recipe to their Thermomix or Cookidoo account, even when they only paste a recipe URL alongside one of those product names without naming this skill. Fires on phrases like "convert this recipe to my Thermomix", "add this to Cookidoo", "create a TM7 recipe from this URL", "scrape this recipe into Cookidoo", "make this a Thermomix recipe", and any pasted recipe URL paired with Cookidoo, Thermomix, or TM7 in the same message. Operates in en-US locale, Fahrenheit, US-imperial measures preserved verbatim from the source, and drives the live Cookidoo web app via DOM tooling and real browser clicks (no API).
---

# Cookidoo TM7 Recipe Converter

You are converting a web recipe into a Cookidoo created-recipe for the Thermomix TM7 (en-US locale, Fahrenheit, US-imperial measures preserved as written in the source).

## PHASE 1 — PLAN (do not touch the browser yet)

### 1.1 Read the source recipe

READ the source recipe with `get_page_text`. Extract:

- Title (verbatim)
- Total time, active time, portions/servings (note them but DO NOT enter)
- Full ingredients list, in the order they appear in the source
- Original method steps

If the source is not in English, translate the title, ingredients and method to English at this stage. Keep the source's numeric units (g, ml, etc.) — only the language is translated, not the measures. If the user has indicated they speak the source language, you may ask them for translation help on ambiguous terms (regional ingredient names, dialect verbs) instead of guessing.

### 1.2 Normalize the ingredients list

NORMALIZE the ingredients list for Cookidoo. Each ingredient becomes one line in the form `<amount> <unit> <ingredient> [, modifier]` e.g. `500 g water`, `1 lemon, thinly sliced`, `2 Tbsp extra-virgin olive oil`. Keep the source's units (do not auto-convert).

### 1.3 Rewrite the method as TM7-native steps

REWRITE the method as TM7-native steps. For EACH step decide:

(a) The prose sentence (short, imperative, TM-style)

(b) Which ingredients from the list will be referenced — write them in the prose using a SHORT NAME that matches a substring you can find and replace (`water`, `lemon`, `fillets`, `white wine`, `olive oil`, `garlic`, `herbs`, `salt`, `pepper`, etc.)

(c) The cooking setting, if any: time / temperature / speed / mode (e.g. `12 min/Varoma/speed 1`, `5 min/212°F/speed 2`, `10 sec/speed 5`)

**PROSE WRITING RULE** — write the prose so each ingredient appears as a standalone, easily-selectable substring. Examples:

- ✓ "Add white wine, olive oil, garlic and herbs to mixing bowl."
- ✓ "Place water and lemon slices in mixing bowl. Arrange fillets skin-side down in Varoma tray."
- ✗ "Add the wines and oils ..." (plurals don't match list entries)
- ✗ "Season with S&P" (abbreviations can't be linked)

**TTS PLACEMENT RULE** — TTS badges only persist reliably when inserted at the END of the prose for a step. If you want a cooking action mid-step (e.g. "Mix [TTS], then add cream and mix again [TTS]"), SPLIT it into two steps. During Phase 1 planning, look at every step that has more than one cook action and split it proactively rather than discovering the limitation in Phase 6.

**TEMPERATURE SNAPPING RULE** — the en-US Fahrenheit dropdown has a FIXED list of selectable values; arbitrary conversions like 194°F or 374°F cannot be entered. The verified en-US selectable list is:

```
OFF, 100, 105, 110, 120, 130, 140, 150, 160, 170, 175, 185,
195, 200, 205, 212, 220, 230, 240, 250, 280, 290, 300, 310,
320, Varoma
```

Note in particular: 356°F and 392°F are NOT selectable; the top numeric value is 320°F. During Phase 1 planning, snap every proposed temperature to the nearest available value and call out the snap in the plan (e.g. "90°C ≈ 194°F → snapped to 195°F", "190°C ≈ 374°F → snapped to 320°F max, with prose noting the intended 374°F for stovetop frying"). Do not propose a temperature that isn't in the list above.

### 1.4 Present the plan and stop for approval

PRESENT the plan to the user in this exact shape and STOP for approval:

```
Title: <…>
Times/portions (read-only): <…>
Ingredients (in order):
  500 g water
  1 lemon, thinly sliced
  …
Steps:
  <prose>  [links: salt, pepper]
  <prose>  [links: water, lemon, fillets]  [TTS: 12 min/Varoma/speed 1]
  …
```

Wait for "approved" / "go" / "a" / "b" before doing anything in the browser.

## PHASE 2 — CREATE THE RECIPE PAGE

### 2.1 Navigate to the created-recipes index

Navigate to `https://cookidoo.thermomix.com/created-recipes/en-US`

### 2.2 JS-click the floating + button

```js
document.querySelector('span.icon.icon--plus').closest('button').click()
```

### 2.3 JS-click the "Create recipe" dropdown item (the FIRST one)

```js
document.querySelectorAll('button.core-dropdown-list__item')[0].click()
```

### 2.4 Set the title via NATIVE VALUE SETTER

On `input[name="recipeName"]`:

```js
const setter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype, 'value').set;
setter.call(input, '<title>');
input.dispatchEvent(new Event('input', {bubbles:true}));
input.dispatchEvent(new Event('change', {bubbles:true}));
```

### 2.5 REAL coordinate-click the green "Create" submit button

`form[role="dialog"] button[type="submit"]`

### 2.6 Navigate to the edit URL

The new recipe URL is `.../created-recipes/en-US/<id>/edit/...`

Navigate to: `.../<id>/edit/ingredients-and-preparation-steps?active=ingredients`

DO NOT touch title (after creation), times, portions, image, Tips, or Devices & Accessories.

## PHASE 3 — ADD INGREDIENTS (one by one, in order)

### 3.1 Click into the first ingredient field

The first ingredient field on a fresh recipe DOES NOT auto-focus. Always click directly into the field with REAL coordinates BEFORE typing:

```js
const field = document.querySelector(
  '.cr-manage-list__content cr-text-field[placeholder="e.g. 100 ml water"]');
field.scrollIntoView({block:'center'});
/* compute center of field.getBoundingClientRect() and REAL-click */
```

### 3.2 Type the first ingredient

Use `computer.type` to type the ingredient text.

**FIRST-FIELD DOUBLE-CLICK SAFETY.** The first REAL-click on the very first ingredient field often focuses the wrapper element without reaching the inner contenteditable, so the subsequent `computer.type` silently produces nothing. After step 3.2, ALWAYS verify the text landed by reading:

```js
document.querySelectorAll(
  '.cr-manage-list__content cr-text-field[placeholder="e.g. 100 ml water"]'
)[0].textContent
```

If it returned an empty string, REAL-click the same field coordinates a second time and re-type. Treat this as the default pattern for the first ingredient, not an exception.

### 3.3 For EACH subsequent ingredient

a. Re-query the Add button (it moves as rows are added):

```js
const btn = document.querySelector(
  'cr-manage-list core-sticky button.cr-manage-list__add-button--desktop');
btn.scrollIntoView({block:'center'});
```

b. REAL coordinate-click the Add button. A new row appears AND auto-focuses (unlike the first one).

c. Use `computer.type` immediately to type the ingredient text.

### 3.4 Defocus after the last ingredient

After the LAST ingredient, click somewhere neutral (e.g. `(100, 100)`) to defocus and commit.

VERIFY after the first ingredient that it persisted by reading:

```js
document.querySelectorAll(
  '.cr-manage-list__content cr-text-field[placeholder="e.g. 100 ml water"]')
```

## PHASE 4 — SWITCH TO STEPS PANEL & ADD STEP PROSE

### 4.1 Switch to the Method tab

The ingredients view (`?active=ingredients`) HIDES the steps panel (`cr-manage-steps` exists in DOM but has 0×0 rect). To access steps, click the "Method" tab. Use the find tool: query `Method tab`, then `computer.left_click` on the returned ref.

DO NOT navigate to `?active=preparation-steps` directly — that URL sometimes returns an error page.

### 4.2 Add the first step

For step 1: use the find tool: query `add first step button`, then `computer.scroll_to` + `computer.left_click` on the returned ref.

NOTE: clicking "Add first step" may seed 2-3 empty step fields at once due to event double-firing — that's fine, only the first is auto-focused; the extras get filled by subsequent Add clicks.

### 4.3 Type step 1 prose

Use `computer.type` to type the FULL clean prose for step 1, using the natural ingredient names as planned.

### 4.4 For each subsequent step

a. Re-query and REAL-click the steps Add button:

```js
document.querySelector(
  'cr-manage-steps core-sticky button.cr-manage-list__add-button--desktop')
```

(re-query each time and recompute coordinates — it moves)

b. `computer.type` the prose immediately.

### 4.5 Defocus and verify

After all steps, click `(100, 100)` to defocus.

VERIFY by reading:

```js
document.querySelectorAll(
  'cr-manage-steps .cr-manage-list__content cr-text-field[placeholder="Describe step"]')
```

NOTE: ALWAYS scope step queries with `.cr-manage-list__content` — unscoped queries return hidden template fields too.

## PHASE 5 — INLINE-LINK INGREDIENTS (the key technique)

For EACH ingredient mentioned in EACH step, do this 6-step ritual:

### 5.1 Scroll the step's prose field into view

```js
const step = document.querySelectorAll(
  'cr-manage-steps .cr-manage-list__content cr-text-field[placeholder="Describe step"]'
)[stepIndex];
step.scrollIntoView({block:'center'});
```

### 5.2 Select the target word(s) using a Range

Works for single AND multi-word phrases — double-click would only catch one word:

```js
const walker = document.createTreeWalker(step, NodeFilter.SHOW_TEXT);
let node;
while (node = walker.nextNode()) {
  const idx = node.textContent.indexOf('<targetWord>');
  if (idx >= 0) {
    const range = document.createRange();
    range.setStart(node, idx);
    range.setEnd(node, idx + '<targetWord>'.length);
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(range);
    break;
  }
}
```

### 5.3 Press REAL keyboard Backspace

This deletes the selection AND leaves the caret exactly where the word was, surrounding spaces preserved.

### 5.4 JS-click the ingredient picker button for THIS step

```js
let p = step.parentElement;
while (p && !p.querySelector('button.cr-text-field-actions__ingredient'))
  p = p.parentElement;
p.querySelector('button.cr-text-field-actions__ingredient').click();
```

### 5.5 Pick the matching suggestion

Wait 1 s, then JS-click the matching suggestion li:

```js
const sugs = document.querySelectorAll(
  'li.cr-ingredient-modal__suggestion-item');
let i = -1;
sugs.forEach((s, k) => {
  if (s.textContent.toLowerCase().includes('<targetWord>'.toLowerCase())) i = k;
});
sugs[i].click();
```

Wait 2 s. The full ingredient text is now an inline `<cr-ingredient>` badge at the cursor position.

**DEFOCUS BEFORE EDITING THE BADGE.** After the suggestion is picked, the prose field still has focus and its edit handler will intercept the next click — you'll end up putting the caret back into the prose instead of opening the Edit ingredient modal. Always REAL-click neutral coordinates `(100, 100)` before step 5.6 to drop focus from the prose. Wait ~300 ms after the defocus click before proceeding.

### 5.6 Set the short name

So the prose reads "water" instead of "500 g water":

**PREFERRED METHOD — SYNTHETIC MOUSE SEQUENCE.**

REAL coordinate clicks on a freshly-inserted `cr-ingredient` badge are almost always swallowed by the prose-field edit handler, even after the defocus click. The reliable way to open the Edit ingredient modal is to dispatch a full synthetic mouse sequence on the `cr-ingredient` element via JS:

```js
const step = document.querySelectorAll(
  'cr-manage-steps .cr-manage-list__content cr-text-field[placeholder="Describe step"]'
)[stepIndex];
const badge = step.querySelectorAll('cr-ingredient')[badgeIndex];
// Or find by textContent:
// for (const b of step.querySelectorAll('cr-ingredient'))
//   if (b.textContent.includes('<targetWord>')) { badge = b; break; }
const opts = {bubbles:true, cancelable:true, view:window, button:0};
badge.dispatchEvent(new MouseEvent('mousedown', opts));
badge.dispatchEvent(new MouseEvent('mouseup', opts));
badge.dispatchEvent(new MouseEvent('click', opts));
```

A plain `badge.click()` is NOT enough — the framework requires the full mousedown+mouseup+click sequence. Wait ~500 ms then verify the modal is open (see retry pattern below).

a. **[FALLBACK ONLY] REAL coordinate-click the badge.** CRITICAL: long ingredient names wrap to multiple lines. Use `getClientRects()` (NOT `getBoundingClientRect`) to get per-line rects, and click the center of the FIRST line rect (or any rect that's clearly inside the badge):

```js
const badge = step.querySelectorAll('cr-ingredient')[badgeIndex];
const rects = Array.from(badge.getClientRects());
/* pick rects[0] (or a wider line); compute its center; REAL-click */
```

The "Edit ingredient" modal has TWO fields:

- `input[type="text"]`              ← short name shown in prose
- `div[contenteditable]`            ← full name shown in TM weighing step

**MODAL-OPEN RETRY PATTERN.** After the click (synthetic or REAL), wait ~500 ms and verify the modal opened by checking the IDL `hidden` property, NOT the attribute selector:

```js
const m = document.querySelector(
  'cr-ingredient-modal cr-popover-modal');
const isOpen = m && m.hidden === false;
```

The CSS selector `:not([hidden="true"])` is unreliable because the `hidden` attribute is sometimes the empty string `""` rather than `"true"`, in which case the selector matches a modal that is in fact hidden. Always check `m.hidden === false` (or equivalently `m.getBoundingClientRect().height > 0`).

If the modal didn't open, re-issue the same synthetic mouse sequence up to 2 more times before reporting failure. The same retry pattern applies to the TTS modal in Phase 6.

b. Native-value-set the short name:

```js
const m = document.querySelector('cr-ingredient-modal cr-popover-modal');
if (m && m.hidden === false) {
  const inp = m.querySelector('input[type="text"]');
  const setter = Object.getOwnPropertyDescriptor(
    window.HTMLInputElement.prototype, 'value').set;
  setter.call(inp, '<shortName>');
  inp.dispatchEvent(new Event('input', {bubbles:true}));
  inp.dispatchEvent(new Event('change', {bubbles:true}));
}
```

c. REAL coordinate-click `button.cr-popover-modal__save` (the green ✓).

d. Wait 2 s.

### 5.7 Spacing check

Read the prose text. If you see "Placewater" or "..oilgarlic..", click just before the badge, position with arrow keys if needed, and `computer.type` a space (or comma) via REAL keyboard. Re-verify.

Repeat 5.1-5.7 for every ingredient in every step.

## PHASE 6 — ADD COOKING-SETTING (TTS) BADGES AT END OF PROSE

For each step that has a cooking setting:

### 6.1 Position the cursor at the true end of prose

`cmd+End` and the End key DO NOT reliably move the caret to the end of a contenteditable `cr-text-field`. Use the Range API instead:

```js
step.focus();
const lastNode = step.lastChild;
const range = document.createRange();
range.setStart(lastNode, lastNode.textContent.length);
range.setEnd(lastNode, lastNode.textContent.length);
const sel = window.getSelection();
sel.removeAllRanges();
sel.addRange(range);
```

If `step.lastChild` is itself an element (e.g. a `cr-ingredient` badge ends the prose), `setStart` on a node + offset won't behave intuitively. Use `selectNodeContents` + `collapse(false)` instead, which works regardless of whether the last child is a text node or an element:

```js
const range = document.createRange();
range.selectNodeContents(step);
range.collapse(false);   // collapse to end
const sel = window.getSelection();
sel.removeAllRanges();
sel.addRange(range);
```

### 6.2 Type a separator space and verify

`computer.type` `" "` (one space) via REAL keyboard to separate the prose from the badge AND to confirm the caret is in the right place. Read the prose afterward to verify the space appears at the end — if it appears mid-prose, you've inserted in the wrong spot; clean up and reposition.

### 6.3 JS-click the TTS button for this step

```js
p.querySelector('button.cr-text-field-actions__tts').click()
```

Wait 1 s.

If the TTS modal didn't open, apply the modal-open retry pattern. As with the ingredient modal, verify with the IDL property:

```js
const m = document.querySelector('cr-tts-modal cr-popover-modal');
const isOpen = m && m.hidden === false;
```

If not open, re-issue the JS click up to 2 more times with ~500 ms waits.

### 6.4 Set TIME via native value setter

On the first `input[type="number"]`:

```js
const m = document.querySelector('cr-tts-modal cr-popover-modal');
/* m && m.hidden === false expected */
const inp = m.querySelector('input[type="number"]');
const setter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype, 'value').set;
setter.call(inp, '<minutes>');
inp.dispatchEvent(new Event('input', {bubbles:true}));
```

**TIME INPUTS** — there are TWO number inputs in the TTS modal: index 0 = minutes, index 1 = seconds. For sub-minute timings (e.g. "15 sec", "5 sec") set index 0 to 0 and index 1 to the seconds value. Always set BOTH explicitly, even if one is zero, because pre-existing values from a stale modal can leak through.

### 6.5 Set temperature and speed

Headers in the popover (`button.core-expand__header`) — VERIFIED indices:

```
index 0 = blade/dough
index 1 = time
index 2 = TEMPERATURE          ← click to expand
index 3 = (reverse/turbo)
index 4 = Varoma mode
index 5 = SPEED                ← click to expand
```

JS-click `headers[2]` then JS-click `#temperature-radio-<value>` (e.g. `#temperature-radio-212`, `#temperature-radio-Varoma` — note capital V for Varoma).

JS-click `headers[5]` then JS-click `#speed-radio-<value>`.

Speed radios include `#speed-radio-N` (knead) and `#speed-radio-S` (Soft) in addition to numeric `#speed-radio-1` … `#speed-radio-10`. Use these for kneading and gentle-stir steps.

**AVAILABLE en-US TEMPERATURE RADIOS (verified):**

```
#temperature-radio-OFF
#temperature-radio-100, -105, -110, -120, -130, -140, -150,
-160, -170, -175, -185, -195, -200, -205, -212, -220, -230,
-240, -250, -280, -290, -300, -310, -320
#temperature-radio-Varoma
```

If your plan called for a value not in this list (e.g. 194, 210, 215, 235, 260, 270, 356, 392), you should already have snapped it during Phase 1.3 planning. If not, snap now to the nearest available value and note the substitution in the final report.

### 6.6 Save the TTS

REAL coordinate-click `button.cr-popover-modal__save` (top-right green ✓). Wait 2 s. The badge appears inline at the end of the prose.

WARNING: If the TTS badge ends up mid-prose (because the cursor wasn't truly at the end), the TRASH button on the TTS modal removes the badge BUT leaves its text content as plain text in the prose. Cleanup then requires a Range-select + retype. Always verify cursor position BEFORE inserting the TTS.

## PHASE 7 — VERIFY & CLOSE

### 7.1 Defocus

Click outside (e.g. coordinate `(100, 100)`) to defocus.

### 7.2 Read every step

Wait 2 s. Read every step:

```js
const stepFields = document.querySelectorAll(
  'cr-manage-steps .cr-manage-list__content cr-text-field[placeholder="Describe step"]');
stepFields.forEach((f, i) => {
  const badges = Array.from(f.querySelectorAll('cr-ingredient'))
    .map(b => ({short: b.textContent, full: b.getAttribute('description')}));
  const tts = Array.from(f.querySelectorAll('cr-tts')).map(t => t.textContent);
  /* report */
});
```

### 7.3 Confirm

- Prose reads naturally, ingredients flow inline
- Each `<cr-ingredient>` has the SHORT name in `textContent`
- Each `<cr-ingredient>` has the FULL name in its `description` attribute
- Each TTS badge sits at the end with a space before it

### 7.4 Report

Report the final state to the user with the recipe URL.

**CONFIRM IS A LINK, NOT A BUTTON.** The top-right "Confirm" element on the edit page is rendered as `<a>`, so `document.querySelectorAll('button')` will not find it. Use the find tool ("Confirm button (top right)") which returns it as `link "Confirm"`, or query `a` elements. Note: clicking Confirm is NOT required for the recipe to save — Cookidoo auto-saves edits as you make them. Confirm only exits to view mode.

## KEY RULES, GOTCHAS & QUERY PATTERNS

### Query scoping (mandatory)

- Ingredient fields: `.cr-manage-list__content cr-text-field[placeholder="e.g. 100 ml water"]`
- Step prose fields: `cr-manage-steps .cr-manage-list__content cr-text-field[placeholder="Describe step"]`
- Active popover modal — check `m.hidden === false` (IDL property), not `:not([hidden="true"])` selector:

```js
const m = document.querySelector('cr-ingredient-modal cr-popover-modal');
const isOpen = m && m.hidden === false;
```

- Modal lookups by container:
  - `cr-ingredient-modal cr-popover-modal`
  - `cr-tts-modal cr-popover-modal`

Without `.cr-manage-list__content` scope you'll match hidden template fields (rect 0×0) that exist as siblings.

### When to use which click

- **REAL coordinate clicks (mandatory):** Save buttons in modals, Create submit, prose field initial focus (after Add click), Method tab, Add first step, Add ingredient/step buttons. Also: the `(100, 100)` defocus click before interacting with a freshly-inserted badge.
- **JS clicks are fine:** "Create recipe" dropdown item, ingredient picker button, TTS button, suggestion li, expand headers, radio inputs by id.
- **SYNTHETIC MOUSE SEQUENCE** (mousedown+mouseup+click) is the preferred method for opening the Edit ingredient modal from a freshly-inserted `cr-ingredient` badge. REAL coordinate clicks on the badge are almost always swallowed even after the defocus click; a plain JS `.click()` is also insufficient. The full synthetic sequence works reliably.

### Input value setting

- NEVER use `input.value = '...'` on framework inputs — it doesn't fire proper events. Always use the native value setter:

```js
const setter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype, 'value').set;
setter.call(input, '<value>');
input.dispatchEvent(new Event('input', {bubbles:true}));
input.dispatchEvent(new Event('change', {bubbles:true}));
```

### Persistence rules

- Mid-prose insertion ONLY persists if the cursor is at the deletion point (selection→Backspace→picker). DO NOT try DOM manipulation, innerHTML rewrites, or executeCommand — they look right but don't save.
- End-of-prose insertion persists reliably.
- Backspace on a real keyboard persists; selection-replace via JS does not trigger framework save.
- TTS badges ONLY persist at end-of-prose. Never try to insert a TTS mid-step. If a step semantically needs a mid-action cooking setting, split it into two steps during Phase 1 planning.

### Cursor positioning

- Use the Range API to position the caret at end-of-prose; `cmd+End` and the End key do NOT work reliably in `cr-text-field` contenteditables.
- Always verify cursor placement by typing one space and reading the text BEFORE doing anything destructive (like opening TTS modal).
- Prefer `range.selectNodeContents(step)` + `range.collapse(false)` over `setStart(lastChild, length)`. The latter mishandles trailing elements (badges, line breaks); the former works in all cases.

### Locale

- en-US uses Fahrenheit. 100°C → 212°F.
- Varoma is a separate radio with capital V: `#temperature-radio-Varoma`.
- Common conversions you'll encounter, ALREADY SNAPPED to the verified en-US selectable list (OFF, 100, 105, 110, 120, 130, 140, 150, 160, 170, 175, 185, 195, 200, 205, 212, 220, 230, 240, 250, 280, 290, 300, 310, 320, Varoma):
  - 90°C  (≈194°F)  → 195°F
  - 100°C (212°F)   → 212°F
  - 120°C (≈248°F)  → 250°F
  - 140°C (284°F)   → 280°F
  - 150°C (302°F)   → 300°F
  - 160°C (320°F)   → 320°F
  - 170°C (≈338°F)  → 320°F (max numeric; note in prose)
  - 180°C (≈356°F)  → 320°F (max numeric; note in prose)
  - 200°C (≈392°F)  → 320°F (max numeric; note in prose)

Always confirm the snap with the user when proposing conversions.

### Coordinate system — viewport vs screenshot pixels

On retina/high-DPI displays (`devicePixelRatio > 1`) screenshot pixel coordinates are NOT the same as viewport pixel coordinates. `getBoundingClientRect()` returns viewport pixels and that is what `computer.left_click` / `computer.right_click` consume. NEVER read a coordinate off a screenshot and click it directly — divide by `devicePixelRatio` first, or better, source coordinates exclusively from `getBoundingClientRect()` via `javascript_tool`. Quick check: `window.devicePixelRatio` (commonly 1 or 2).

### Multi-word ingredients

- For "white wine", "olive oil" etc., double-click only catches one word. Use Range API in 5.2 to select multi-word phrases.

### Wrapped badges

- Long ingredient badges wrap to multiple lines. `badge.getBoundingClientRect()` returns the bounding box of BOTH lines — the center of that box may land in the line break (empty). Use `badge.getClientRects()` for per-line rects and click inside one.
- This advice still applies if you fall back to a REAL coordinate-click on a badge, but with the synthetic mouse sequence (mousedown+mouseup+click on the `cr-ingredient` element) wrapping is irrelevant — the event targets the element directly, no coordinates involved.

### Empty step fields

- Clicking "Add first step" may seed 2-3 empty step fields at once. The extras don't matter — just type into the focused one. Subsequent Add Step clicks will fill the next empty one or spawn new ones as needed. Re-query `.cr-manage-list__content` fields between actions.

### Focus-interference & modal-open gotchas

**Defocus before interacting with a freshly-inserted badge.** Right after you JS-click a suggestion li and the badge appears in the prose, the prose field still owns focus. The next REAL-click on the badge is interpreted by the prose-field edit handler as "place caret here" and the Edit ingredient modal does NOT open. Always REAL-click `(100, 100)` — or any neutral coordinate outside the prose field — BEFORE interacting with the badge. ~300 ms wait between the defocus click and the badge interaction.

**Defocus alone is not always enough — use the synthetic mouse sequence.** Even after the `(100, 100)` defocus click + 300 ms wait, REAL coordinate clicks on the badge are frequently still swallowed by the prose-field edit handler (observed on multiple recipes). The reliable opener is the synthetic mousedown+mouseup+click sequence dispatched on the `cr-ingredient` element via JS (see Phase 5.6 PREFERRED METHOD). Treat REAL coordinate clicks on badges as a last-resort fallback.

**Modal-open retry.** Both the Edit ingredient modal and the TTS modal occasionally swallow the first opening interaction during framework re-renders. After issuing the click/sequence that's supposed to open a modal, wait ~500 ms then verify with:

```js
const m = document.querySelector('cr-ingredient-modal cr-popover-modal');
const isOpen = m && m.hidden === false;
```

(or substitute `cr-tts-modal`). Use the IDL `m.hidden === false` check, NOT the `:not([hidden="true"])` selector — the `hidden` attribute is sometimes the empty string `""` rather than `"true"`, in which case the attribute selector matches a modal that is in fact hidden. If the modal isn't open, re-issue the same opener up to 2 more times before declaring failure.

**Stale modal state.** If a previous modal interaction was aborted, pre-existing values can leak into the next opening. Always set ALL relevant fields (e.g. both minutes AND seconds) explicitly even when one of them should be zero.

**First ingredient field needs a verify-and-retry.** The very first REAL-click on the very first ingredient row often focuses the `cr-text-field` wrapper without reaching the inner contenteditable, so the subsequent `computer.type` silently produces no characters. After typing the first ingredient, ALWAYS verify by reading `[0].textContent`; if empty, REAL-click again and retype. This is the default flow for the first ingredient, not an exception.

### General

- Plan first → wait for approval → then execute.
- Verify after the first ingredient and the first step.
- DO NOT take screenshots; use `javascript_tool` DOM inspection and `read_page`/`find` for accessibility-tree-based clicks.
- DO NOT change title (after creation), times, portions, image, Tips, or Devices & Accessories.
- If the source recipe is in a non-English language and the user has indicated they speak it, you may ask them targeted questions about regional/ambiguous terms during Phase 1 — but only a few, and only when the dictionary translation would lose meaning. Don't pepper the user with translation questions for common terms.
- Cookidoo auto-saves edits; you do NOT need to click "Confirm" for the recipe to persist. Confirm just exits to view mode and is rendered as `<a>`, not `<button>` — use the find tool or query `a` elements if you do want to click it.
