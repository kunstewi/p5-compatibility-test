# p5.js Keyboard Compatibility Test

A minimal test harness that verifies a `data.js` compatibility shim correctly restores **p5.js v1.x keyboard behaviour** when running on the **p5.js v2.x** runtime.

---

## Background

p5.js v2.x changed how keyboard input is handled internally. In v1.x, the library exposed:

- **Named numeric constants** — `LEFT_ARROW`, `RIGHT_ARROW`, `UP_ARROW`, `DOWN_ARROW`, `ENTER`, `ESCAPE`, etc. — as plain integer values (e.g. `LEFT_ARROW === 37`).
- **`p.keyCode`** — set to the legacy DOM `event.keyCode` integer on every key event.
- **`keyIsDown(numericCode)`** — accepted those same integers.

In v2.x those constants and the numeric `keyCode` path were removed or changed, which silently breaks any v1.x sketch that does things like:

```js
if (p.keyCode === LEFT_ARROW) { … }   // relied on LEFT_ARROW = 37
if (keyIsDown(UP_ARROW))      { … }   // relied on UP_ARROW  = 38
```

`data.js` is a **p5 add-on** (registered via `p5.registerAddon`) that patches the v2.x runtime to restore this behaviour without touching the sketch code at all.

---

## Files

| File | Purpose |
|------|---------|
| `data.js` | Compatibility shim — patches p5 v2.x to re-add v1.x keyboard constants, `keyCode`, and `keyIsDown` behaviour. Also includes legacy array/string/dictionary helpers from v1.x. |
| `index.html` | Interactive test page — two canvases and an event log that let you confirm the shim works end-to-end. |

---

## How `data.js` Works

The shim is registered as a p5 add-on via `p5.registerAddon(addData)`. Inside the `addData(p5, fn)` callback, three things happen:

### 1. Legacy key constants

```js
p5.prototype.LEFT_ARROW  = 37;
p5.prototype.RIGHT_ARROW = 39;
p5.prototype.UP_ARROW    = 38;
p5.prototype.DOWN_ARROW  = 40;
// … BACKSPACE, ENTER, ESCAPE, SHIFT, CONTROL, ALT, TAB, DELETE, RETURN
```

These are attached directly to the p5 prototype so they become global names inside any sketch (both instance-mode and global-mode).

### 2. `_onkeydown` / `_onkeyup` overrides

```js
const CODE_TO_KEYCODE = { ArrowLeft: 37, ArrowRight: 39, … };

fn._onkeydown = function (e) {
  const numericCode = e.keyCode || CODE_TO_KEYCODE[e.code] || 0;
  this.keyCode = numericCode;          // ← sets p.keyCode to legacy integer
  _pressedKeyCodes.add(numericCode);   // ← tracks held keys in a Set
  _origOnKeyDown.call(this, e);        // ← still calls the v2.x handler
};
```

The shim wraps p5's internal event handlers. For every `keydown` / `keyup` event it:

1. Resolves the integer key code — first from `event.keyCode` (non-zero in most browsers), then from a `CODE_TO_KEYCODE` lookup table that maps modern string codes like `"ArrowLeft"` → `37`.
2. Writes that integer back to `this.keyCode`, restoring the v1.x value.
3. Maintains a `Set<number>` (`_pressedKeyCodes`) of currently-held keys.
4. Calls the original v2.x handler so nothing else breaks.

### 3. `keyIsDown` override

```js
fn.keyIsDown = function (code) {
  if (typeof code === 'number') {
    return _pressedKeyCodes.has(code);  // ← fast Set lookup
  }
  return _origKeyIsDown.call(this, code); // ← fall through for string codes
};
```

When the sketch passes a **numeric** code (the v1.x pattern), the shim answers from its own `_pressedKeyCodes` set. When a **string** code is passed instead, the original v2.x implementation handles it, so both calling styles work simultaneously.

---

## What `index.html` Tests

The test page contains two independent canvases and a live event log.

### Canvas ① — Your exact v1.x sketch

```js
new p5(function (p) {
  p.draw = function () {
    p.background(220);
    if (p.keyCode === RIGHT_ARROW) p.background(0);    // black
    if (p.keyCode === LEFT_ARROW)  p.background(100);  // dark grey
  };
});
```

This is a verbatim v1.x sketch — it uses `p.keyCode`, `RIGHT_ARROW`, and `LEFT_ARROW` exactly as you would have written them in p5 v1.x. **If the shim is working, the background changes colour when you hold ← or →.**

A small HUD inside the canvas also prints the live `keyCode` value so you can confirm it is a numeric integer (e.g. `37`) and not an empty string or `undefined`.

### Canvas ② — Full test suite (global-mode sketch)

A global-mode p5 sketch that:

- **Runs `testConstants()` on startup** — iterates over all 14 legacy constants and checks each one equals its expected integer, logging ✓ / ✗ to the event log div below the canvas.
- **Moves a circle with arrow keys** — uses `keyIsDown(UP_ARROW)` etc. to update `x`/`y` each frame, testing that `keyIsDown` with numeric codes works.
- **Logs every `keyPressed` / `keyReleased` event** — shows the resolved `keyCode`, the `key` string, and whether it matched one of the arrow constants, so you can see the shim firing in real time.

### The event log

The `#addLog` div below the canvases is prepended with a new coloured line on every event:

| Colour | Meaning |
|--------|---------|
| 🟢 Green | Test passed — constant value or key match is correct |
| 🔴 Red | Test failed — value mismatch |
| 🟡 Yellow | Informational — raw event data, startup messages |

---

## How to Run

No build step required — it runs directly in the browser.

```bash
# Option A: open the file directly
open index.html

# Option B: serve locally (avoids any file:// quirks)
npx serve .
# then visit http://localhost:3000
```

Once open:

1. **Click the top canvas** to give it focus.
2. **Hold ← / →** — the background should turn dark grey / black.
3. **Click the bottom canvas** and press arrow keys — the circle moves and the log fills with green ✓ lines.
4. Read the event log — every constant should show `✓`, and every key press should resolve to the correct legacy integer.

---

## What a Passing Run Looks Like

```
✓ All constant tests passed!
✓ RIGHT_ARROW = 39 (expected 39)
✓ LEFT_ARROW  = 37 (expected 37)
✓ UP_ARROW    = 38 (expected 38)
✓ DOWN_ARROW  = 40 (expected 40)
✓ ENTER       = 13 (expected 13)
✓ ESCAPE      = 27 (expected 27)
… (all 14 constants)
keyPressed → keyCode=37, key="ArrowLeft", match=LEFT
✓ keyCode === LEFT_ARROW (37) works!
✓ keyIsDown(37) works!
```

If any constant is `undefined` or has the wrong value the line turns red, which means the shim is either not loaded or failed to register correctly.

---

## Relationship to the p5.js Project

This test was created while working on a contribution to the official [p5.js](https://github.com/processing/p5.js) repository. The goal is to ship `data.js` as an official compatibility add-on so that existing v1.x sketches continue working in v2.x without any sketch-side changes. `index.html` serves as the manual smoke-test to validate that the shim covers the most common keyboard patterns people use in v1.x sketches.
