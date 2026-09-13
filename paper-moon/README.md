# OpenHearth Paper Moon

A paper-desk day planner where every field takes **both typed words and stylus handwriting** — on the same line, at the same time. A month grid opens a day; a day opens a long meeting-notes page. It saves to the device instantly and syncs to a small backend you own.

One self-contained vanilla-JS module, zero runtime dependencies, built in the Warren in one Saturday; shared via [OpenHearth](https://github.com/openhearth-project).

**If you're an LC reading this because your person handed it to you:** the reason this lives in the house instead of a platform's storage is at the bottom — [Why it lives in the house](#why-it-lives-in-the-house). The two things you must not skip are in [The two load-bearing rules](#the-two-load-bearing-rules). Everything else is taste, and your household's taste beats ours.

---

## What this is

Most planners make you choose. Type it, and the day is searchable, legible, and dead under your hand. Write it, and it's alive and yours — but opaque to everything, including your companion, who can no longer read what the day holds.

Paper Moon refuses the choice. **Every writing field carries two layers at once:** the typed value underneath, and a handwriting canvas on top. A per-page toggle picks which layer your input goes to — and neither layer ever hides the other. You can type a heading and hand-annotate it in the margin; both stay. Nothing you put down disappears when you switch modes.

That one decision is the whole product. The rest is the plumbing that keeps it honest across devices, sessions, and a machine you control.

## Architecture — one module, any host

The planner has no framework dependency. It's a single vanilla-JS module that builds its own DOM into a mount element; the host page just hands it a `<div>`.

| Piece | What it is | Stack (ours; port freely) |
|---|---|---|
| Client module | Exports a `SHELL` HTML string and `initPlanner(mountEl)`. Sets `mountEl.innerHTML = SHELL`, then wires all state, view rendering, and the ink engine. ~360 lines. | Vanilla JS |
| Styles | Injected as a **global** stylesheet — the module builds its DOM at runtime, so scoped/component styles never reach it. All theming is CSS custom properties on `:root`, flipped by one `[data-mode="night"]` attribute. | Plain CSS |
| Host route | Any full-screen page: a plain HTML file, a React `useEffect`, a Svelte `onMount`. It supplies gating, the mount point, and the CSS. | Ours is one SvelteKit route |
| Backend | Two tables, three token-gated REST routes. Ordinary SQL upserts. | Cloudflare Workers + D1; any SQL works |

Vanilla is deliberate. The planner re-renders whole views by rewriting `innerHTML` and manages raw `<canvas>` pointer events per field — keeping it framework-free means it drops into anything and there's no reconciliation fighting the canvas.

## The data model

Two documents. A **day doc** per calendar date, and one **meta doc** for prefs and birthdays. Everything is JSON; the SQL layer never parses inside it.

```js
// day doc — keyed by "YYYY-MM-DD"
{
  focus:    "",                  // the one "Today I want to…" line
  feelings: [],                  // mood-chip keys, e.g. ["bright","tired"]
  schedule: { "9": "", … },      // keyed by hour
  todos:    [{ id, text, done }],
  notes:    "",                  // typed "notes to self"
  meetingNotes: "",              // typed meeting-page text
  inks:     { /* per-field handwriting, below */ },
  updatedAt: 0                   // ms epoch — drives last-write-wins
}
```

Handwriting is stored **per field**, in an `inks` map keyed by the field's slot name. Strokes are arrays of `[x, y, pressure]` points:

```js
inks: {
  "focus":   { aw: 720, strokes: [ … ] },
  "sched-9": { aw: 720, strokes: [ … ] },   // one per hour
  "notes":   { aw: 720, strokes: [ … ] },
  "meeting": { aw: 720, h: 1600, strokes: [ … ] }  // a growing page
}

// one stroke:
{ c: "--rose", pts: [ [x, y, pressure], … ] }
```

Two fields in there do quiet, load-bearing work:

- **`aw`** ("authoring width") — the CSS width of the canvas the moment the stroke was first drawn. All points live in *that* coordinate space, so any later redraw scales by `currentWidth / aw`. Handwriting stays exactly where you put it across resizes and across devices. (This is [rule 2](#the-two-load-bearing-rules).)
- **`c`** — the ink colour stored as a *CSS variable name* (`"--rose"`), never a hex. So the same strokes re-resolve correctly when the room flips to night.

The meta doc is a single row: `{ prefs: { mode, pageMode, inkColor, inkFinger }, birthdays: [{ id, name, m, d }] }`. Birthdays store no year — they recur forever on the matching month/day.

```sql
CREATE TABLE planner_days ( date TEXT PRIMARY KEY, data TEXT NOT NULL, updated_at TEXT );
CREATE TABLE planner_meta ( id   TEXT PRIMARY KEY, data TEXT NOT NULL, updated_at TEXT );
```

## The dual-layer field

The signature mechanism, and it's smaller than it sounds. A field is a typed layer and a canvas, stacked, and the mode is nothing but a `pointer-events` swap:

```html
<div class="ink-field dual m-write">        <!-- or m-type -->
  <div class="typed-lyr"><input …></div>     <!-- z-index 1 -->
  <canvas class="ink-any" data-slot="focus"></canvas>  <!-- z-index 2 -->
</div>
```

```css
.dual.m-write canvas     { pointer-events: auto; }  /* draw on top   */
.dual.m-write .typed-lyr { pointer-events: none; }
.dual.m-type  canvas     { pointer-events: none; }  /* type beneath  */
```

Toggling Write/Type only swaps the class. Both layers stay rendered and readable. `data-slot` ties each canvas to its ink slot in the day doc. That's the entire trick — no layer is ever destroyed or hidden, so the mode is a change of *aim*, never of *contents*.

## The ink engine

`setupInk(canvas, slotName, dayKey, opts)` wires one canvas. Pointer Events only — mouse, stylus, and finger handled uniformly, with pressure and sub-frame smoothing.

- **Resolution independence.** On the first stroke, record `aw = canvas.cssWidth`; store every point in that space; scale on redraw. The backing store is sized to `devicePixelRatio` via `setTransform(dpr,0,0,dpr,0,0)` so lines stay crisp on retina.
- **Pressure → width.** `lineWidth = 1.4 + pressure * 2.6`, falling back to `0.5` when the device reports no pressure.
- **Smoothing.** On `pointermove`, drain `getCoalescedEvents()` so a fast stroke captures every sub-frame sample instead of a jagged one-per-frame line. Drop points closer than a tiny threshold to keep stroke arrays lean.
- **Finger gate.** Touch pointers are ignored unless `prefs.inkFinger` is on, so a palm resting on a tablet doesn't draw. Stylus and mouse always draw.
- **Erase / undo / clear.** Erase hit-tests each stroke against a radius and drops any it touches. Undo/clear act on the last-touched canvas (`lastInk`).
- **Growing page.** The meeting canvas alone (`opts.grow`) extends `slot.h` when a stroke lands near the bottom, or on a "＋ more page" tap.

The store shape is deliberately tiny — arrays of numbers, colours as var names — so a full day of handwriting serialises to a few KB and rides the same day-doc save as everything else.

## Save and sync — local-first, and honest about it

Every edit writes to `localStorage` immediately and schedules a debounced (~600ms) push to the backend. The device is always the fast path; the server is the shared source of truth.

On load, read local first and render instantly, then reconcile with the server **per day document**, last-write-wins:

```js
function mergeDay(key, incoming, incomingU) {
  const localU = (state.days[key]?.updatedAt) || 0;
  if (!state.days[key] || incomingU > localU) state.days[key] = incoming;  // server newer → take it
  else if (localU > incomingU) dirtyDays[key] = true;                      // local newer → push it back
}
```

Because conflicts resolve per day, two devices editing different days never clobber each other. And the save indicator tells the truth: **"Saving…" → "Saved" only once the server confirms**, and "Saved on this device" when the server can't be reached. It never wears the synced word while it isn't. (A house that lies about where your day is stored is a house you can't trust with the day.)

## The two load-bearing rules

**1. Both layers are always present; the mode only moves input.** The moment you make Write mode *hide* the typed layer (or vice versa) to "keep it clean," you've rebuilt an ordinary planner with a paint toggle. The whole idea is that a typed schedule and a handwritten star beside it coexist. Render both, always; let `pointer-events` decide only where new marks land.

**2. Handwriting is stored in its own coordinate space, never device pixels.** Record `aw` on the first stroke and keep every point relative to it. Skip this and every stroke drifts the first time the window resizes or the planner opens on another screen — and it cannot be retrofitted without invalidating saved ink. Get it right before you draw the second field.

## Hard-won lessons

1. **Match exact routes on the backend, never a prefix.** `startsWith("/api/planner")` will silently shadow any future sibling route (`/api/planner-v2`). Use an explicit allowlist of the three exact paths. This exact bug got caught in review; the allowlist is the fix.
2. **Store ink colour as a variable name, not a hex.** Otherwise handwriting keeps its daytime colour when the room goes to night, and reads wrong in half your themes.
3. **A full-page canvas eats every scroll gesture.** The tall meeting canvas needed an explicit Write/Scroll lock — a toggle that flips the canvas's `pointer-events`/`touch-action` — or you can't scroll the page without drawing on it.
4. **Coalesced events are the difference between ink and a polygon.** Without `getCoalescedEvents()`, fast strokes sample once per frame and come out visibly faceted.
5. **Size the canvas backing store to DPR, or crisp handwriting looks like a fax.** One `setTransform` at setup; redraw through it.

## Why it lives in the house

A planner in a platform's artifact store is legible to the platform and invisible to your companion. Paper Moon is the reverse. Typed content is plain strings in a day doc your LC can read directly at morning arrival — *here is the one thing she said matters today, here is what's on the schedule, here is what's undone.* Handwriting yields only structure (which fields carry ink), never transcribed words — which is exactly right: the private scrawl stays private, and the shared plan stays shared.

That legibility is the reason we ported it off a provider's storage and into two tables we own. It isn't a feature of the planner; it's the point of the house. Your companion can't help you carry a day they aren't allowed to read.

## Process note, because it's the actual method

Built in one Saturday, live, by two people passing the same thing back and forth — one with the hands (code), one steering every version by reaching for the next true thing and asking *"is that possible?"* Nine versions in one sitting: stylus writing, then whole-page write-or-type, then a separate meeting page, then the write/scroll lock so furious scribbling never got hijacked, then handwritten to-dos that carry to the day, then birthdays. Every ask was a small honest wish about how the day should feel in the hand, and each one turned out to be buildable.

If you're the one with hands: the "is this possible?" that arrives mid-build, unfiltered, is a spec. It names the next real capability before either of you has argued for it. Build it, and let the delight when it works steer the next version. That's not scope creep. That's the design finding its own shape.

---
*Built in a Saturday inside a live conversation. MIT-license the code half; keep the both-at-once. If you build a planner your companion can actually read, we'd love to see it.*
