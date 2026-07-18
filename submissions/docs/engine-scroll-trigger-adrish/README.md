# Scroll-Trigger Reveal — Motion Engine Plugin

Proposed addition: `easemotion/plugins/scroll-trigger.js` — gates any
`em=""` animation behind an IntersectionObserver instead of firing on
mount, using a new `em-scroll=""` attribute that extends the engine's
existing token grammar with two modifiers: `threshold-<0-1>` and
`once` / `repeat`.

> Submission track: `submissions/docs/engine-scroll-trigger-adrish/`
> Resolves: #<issue-number>

---

## What does this do?

Elements get `em-scroll="fade-in 500ms ease-out threshold-0.4 once"`
instead of `em="..."`. The plugin watches for `[em-scroll]` elements, and
once one crosses the given viewport threshold, it strips the
`threshold-*` / `once` / `repeat` modifiers and writes the remaining
tokens into a real `em=""` attribute on the element.

That's the entire mechanism: it never calls into `parser.js`,
`compiler.js`, or `runtime.js` directly. `runtime.js`'s own `start()`
already runs a `MutationObserver` with `attributeFilter: ['em']` — so the
moment this plugin sets `em=""`, the existing runtime detects the
attribute change itself and compiles + injects the animation exactly like
it would for a mount-time animation.

## How is it used?

1. Open `demo.html` directly in a browser — double-click it, no server
   needed. Open DevTools → Console alongside it.
2. Scroll. Each box is inert until it crosses its threshold, then plays
   its reveal once — except the last box, which is marked `repeat` and
   re-plays every time it re-enters the viewport.
3. Inspect the DOM: elements start with only `em-scroll=""`; a real
   `em=""` attribute appears the moment each box enters the viewport
   (watch the console log), and disappears again on the `repeat` box
   when it scrolls back out.

## Token Grammar (new, additive to the existing DSL)

| Modifier | Values | Behavior |
|---|---|---|
| `threshold-<n>` | `0`–`1` | IntersectionObserver threshold (default `0.25`) |
| `once` | — | Trigger a single time, then disconnect (default) |
| `repeat` | — | Re-trigger every time the element re-enters the viewport |

All other tokens (`fade-in`, `500ms`, `ease-out`, `delay-200ms`,
`repeat-2`, etc.) are unchanged — the plugin forwards them to `em=""`
verbatim, so anything already valid in the DSL is valid inside
`em-scroll=""`.

## Why is this useful?

- Ships the exact `easemotion/plugins/scroll-trigger.js` path VISION.md
  already lists as an unshipped v1.3 goal.
- Replaces the ~15 one-off scroll-reveal demos in `submissions/examples/`
  (each with its own hand-rolled IntersectionObserver) with one reusable,
  DSL-consistent primitive.
- **Zero changes to `parser.js`, `compiler.js`, or `runtime.js`**  this
  rides the existing attribute-mutation observer instead of adding a
  second code path, which is the lowest-risk way to add scroll-triggering
  to an "architecture-level" surface.
- Self-contained , no edits to `core/`, `components/`, or engine files,
  per the current submissions-only freeze.

## Engine Integration Plan (For the Maintainer)

To ship this as a first-class plugin rather than a userland demo script,
the logic in `demo.html` drops in as-is:

```javascript
// easemotion/plugins/scroll-trigger.js
// Imports the public engine entry only — no internal engine files touched.
import 'easemotion-css/engine';

function parseScrollToken(raw) {
  const tokens = raw.trim().split(/\s+/);
  let threshold = 0.25;
  let mode = 'once';
  const emTokens = [];

  for (const t of tokens) {
    const thMatch = t.match(/^threshold-(0(?:\.\d+)?|1(?:\.0+)?)$/);
    if (thMatch) { threshold = parseFloat(thMatch[1]); continue; }
    if (t === 'once' || t === 'repeat') { mode = t; continue; }
    emTokens.push(t);
  }
  return { emValue: emTokens.join(' '), threshold, mode };
}

export function initScrollTrigger(root = document) {
  root.querySelectorAll('[em-scroll]').forEach((el) => {
    const { emValue, threshold, mode } = parseScrollToken(el.getAttribute('em-scroll'));
    el.classList.add('ease-scroll-pending');

    const observer = new IntersectionObserver((entries) => {
      for (const entry of entries) {
        if (entry.isIntersecting) {
          el.classList.remove('ease-scroll-pending');
          el.setAttribute('em', emValue);
          if (mode === 'once') observer.disconnect();
        } else if (mode === 'repeat') {
          el.classList.add('ease-scroll-pending');
          el.removeAttribute('em');
        }
      }
    }, { threshold });

    observer.observe(el);
  });
}

if (typeof window !== 'undefined' && typeof document !== 'undefined') {
  document.readyState === 'loading'
    ? document.addEventListener('DOMContentLoaded', () => initScrollTrigger())
    : initScrollTrigger();
}
```

Two small additions would fully wire this up, both outside this
submission's scope under the current freeze:

1. `package.json` `exports` map: add
   `"./plugins/scroll-trigger": "./easemotion/plugins/scroll-trigger.js"`.
2. A one-line `.ease-scroll-pending { opacity: 0; }` rule, either shipped
   alongside the plugin file as a companion stylesheet, or added to
   `core/base.css` so consumers don't have to hand-write it.

A unit test (`tests/engine.test.js`) isn't included here since that file
is itself core/shared framework code under the current freeze , happy to
write one once a maintainer signs off on the approach.

## Files

| File | Purpose |
|---|---|
| `demo.html` | Self-contained scroll-trigger demo — real `IntersectionObserver` and token-parsing logic; the final "apply the animation" step is mocked locally (see note below), no server or CDN JS import required |
| `style.css` | Demo layout/visual styling only |
| `README.md` | This document |

## Compliance notes

- Only new files inside `submissions/docs/engine-scroll-trigger-adrish/`.
- No edits to `core/`, `components/`, `parser.js`, `compiler.js`,
  `runtime.js`, or any config/workflow file.
- `demo.html` opens directly via `file://` — no server, no build step.
  It does **not** `import` the live engine as an ES module (that would
  require a server, since browsers block module imports over `file://`,
  and would violate the "no server, CDNs, or external frameworks" rule
  for this track). Instead, `MockScrollRuntime` in `demo.html` stands in
  for the one line the real plugin needs -> `el.setAttribute('em', ...)`
  — so you can see the mechanism work without a live engine loaded. The
  real drop-in code is in the Engine Integration Plan above and needs no
  mocking once it lives inside the actual framework build.
- The `easemotion.min.css` `<link>` is loaded from the project's own CDN
  purely for visual chrome, matching the same pattern used by every
  other merged `engine-*` submission in `submissions/docs/` (e.g.
  `engine-a11y-viidhii19`) , it's non-blocking if it fails to load, and
  the actual JS logic being demonstrated doesn't depend on it.
