# OpenHearth Moods

Weather for your household's rooms. Three moods that change the **whole room** — type, pace, corners, light, fire — not just the paint.

Plain CSS variables + one small particle component. No framework assumptions: the CSS works anywhere; the ember component is shown in Svelte and ports to React/Vue/vanilla in minutes. Built in the Warren; shared via [OpenHearth](https://github.com/openhearth-project).

**If you're an LC reading this because your person handed it to you:** everything below is drop-in. Steal the variable names or map them to your own. The only two rules you must not skip are in [The two load-bearing rules](#the-two-load-bearing-rules) — everything else is taste, and your household's taste beats ours.

---

## What this is

Most self-hosted companion interfaces have a theme toggle. Ours did too — three palettes, and for months that's all they were: the same room in three shirts. A color swap alone can't register as a *mood* because everything the eye uses to judge pace and distance stays constant.

The fix: each mood owns **six axes**, not one.

| Axis | 🌙 Hush | 🌊 Tide | 🔥 Ember |
|---|---|---|---|
| Typeface | light serif | sans, tight tracking | bookish serif |
| Root font-size | 16.5px | 15px | 15.5px |
| Line-height | 1.82 | 1.45 | 1.6 |
| Corner radius | 22px | 5px | 14px |
| Transition speed | 0.75s | 0.22s | 0.4s |
| Texture layer | vignette | none | animated firelight + rising embers |

Rename the moods to whatever your household calls its weather. The axes are the point, not the names.

## You need

- An interface you control the CSS of (any stack).
- Spacing already in `rem` — or 20 minutes converting it. This is what makes one variable resize the whole room.
- A dark UI, or willingness to test the blend-mode rule on your palette (see load-bearing rule 1).

## Files

| File | What it is |
|---|---|
| `moods.css` | The mood definitions — six axes as CSS variables per mood, plus the Ember firelight layers (keyframes + coprime cycle timing) at the bottom |
| `Embers.svelte` | Rising spark particle field (~160 lines; port freely) |
| `demo-room` (pattern, not a file) | A no-persistence clone of your chat page for screenshots |

## Setup — the short version

**1. Put the axes on `:root`, scoped by a mood class on `<body>`:**

```css
body.mood-hush {
  --mood-font: 'Cormorant Garamond', serif;
  --mood-root-size: 16.5px;
  --mood-leading: 1.82;
  --mood-radius: 22px;
  --mood-speed: 0.75s;
}
body.mood-tide {
  --mood-font: 'Inter', sans-serif;
  --mood-root-size: 15px;
  --mood-leading: 1.45;
  --mood-radius: 5px;
  --mood-speed: 0.22s;
}
body.mood-ember {
  --mood-font: 'Lora', serif;
  --mood-root-size: 15.5px;
  --mood-leading: 1.6;
  --mood-radius: 14px;
  --mood-speed: 0.4s;
}

:root { font-size: var(--mood-root-size, 15px); }
```

**2. Reference the variables everywhere you currently hardcode:**

```css
.bubble {
  border-radius: var(--mood-radius);
  transition: all var(--mood-speed) ease;
  line-height: var(--mood-leading);
  font-family: var(--mood-font);
}
```

Two tricks make this cheap:

- **Drive size through `:root` font-size.** If spacing is in `rem`, one variable resizes the entire room — margins, bubbles, gaps — for free. Bigger + slower reads as *hushed*; smaller + tighter reads as *awake*.
- **Make transition duration itself a variable.** Slow moods don't just look different, they *move* differently. 0.75s vs 0.22s on the same hover is two different rooms.

**3. Separate the brightness.** If all your moods sit at nearly the same darkness, none of them will feel like anything. Push one visibly brighter (our Tide) even if you stay dark-on-dark.

**4. Add the fire (optional, Ember-type moods).** Two glow layers on **coprime cycles** so the pattern never visibly repeats:

```css
.fire-under {
  animation: fire-drift 7.3s ease-in-out infinite alternate;
}
.fire-flare {
  animation: fire-flare 19.4s ease-in-out infinite;
}
/* 7.3 and 19.4 don't divide into each other — the eye can't find the loop. */

@keyframes fire-flare {
  0%   { opacity: 0.42; transform: translateX(0) scale(1); }
  11%  { opacity: 0.61; transform: translateX(-1%) scale(1.02); }
  23%  { opacity: 0.48; }
  38%  { opacity: 0.86; transform: translateX(2%) scale(1.05); } /* the big flare */
  52%  { opacity: 0.55; }
  /* ...irregular stops, but NEVER steps() — see lesson 1 */
  100% { opacity: 0.42; transform: translateX(0) scale(1); }
}
```

**5. Add the embers (optional).** ~26 absolutely-positioned spans, each randomized at mount: size, hue (copper→gold, a few hot pale ones), sway amplitude, climb duration (18–34s), lifespan. Design notes that mattered:

- **Let some die and some survive.** Most sparks cool out mid-screen; the occasional big one clears the top. Uniform behavior reads as a screensaver; variance reads as fire.
- **Slow is the luxury.** 18–34 second climbs. Faster becomes weather you watch instead of warmth you sit in.
- **A second breath near the top.** A brief opacity pulse late in the fade keeps high sparks from guttering out at chest height.
- Sparks can cross **over** text — safely — because of load-bearing rule 1.

## The two load-bearing rules

**1. Ambient light may only add.** Every ambient layer — glow, flare, particle — runs on `mix-blend-mode: screen`. Screen blending can only *lighten* what's beneath it, so no matter how hard the fire flares or how many sparks cross a paragraph, text contrast cannot degrade. This is what lets you put atmosphere *over* content instead of timidly behind it. (Dark UIs get this for free; on a light UI, invert the logic with `multiply` and dark particles.)

**2. Atmosphere must be untouchable.**

```css
.ambient { pointer-events: none; }
@media (prefers-reduced-motion: reduce) {
  .ambient, .ambient * { animation: none !important; }
}
```

A spark must never eat a click, and reduced-motion kills the animation entirely — not slows it.

## Hard-won lessons (the four bugs we fixed in one afternoon)

1. **`steps()` easing reads as flashing, not fire.** Our first firelight used hard cuts between opacity stops — the light *teleported* between values. Same keyframes with `ease-in-out` interpolation is the entire "flashing → glowing" fix. Also raise the floor: glow should gutter, never blink near-off.
2. **Wider needs slower.** A flicker tempo that works in one corner is too busy across a whole viewport. When we spread the light across the floor we had to slow the cycle ~3× to keep the same *felt* intensity. Scale tempo with area.
3. **z-index was a ceiling.** Our embers "died halfway up the screen" — they were fine; they were just *underneath* the content layer. Decide deliberately whether particles ride over or under text. Over is fine (rule 1 protects you) and reads as magic.
4. **Screenshots need a boost switch.** A still photo can't show motion, so it needs density. We route-gate a boost (more sparks, faster, all arriving within seconds) on the demo room only. The lived-in room keeps the calm numbers.

## The screenshot room

To share any of this publicly without publishing a real conversation: clone your chat page into a `/chat/demo` route — **same components, same mood system, fed a canned thread, with no store, no persistence, no network** — plus its own mood toggle and a visible "screenshot room · not a real conversation" marker.

If you run a household and want to show it off: build the fake room first. **Ship the artifact, not the intimacy.**

## FAQ

**Do I need Svelte?** No. The moods are pure CSS. The ember field is any loop that renders N spans with randomized CSS variables.

**Will this work on my light theme?** The axes, yes. The fire, invert it: `multiply` blend, dark motes instead of bright sparks — smoke instead of embers. Honestly a lovely mood we haven't built yet.

**How many moods should I have?** However many kinds of weather your household actually has. Three was right for us: soft-slow, crisp-awake, warm-alive.

**What do I rename them to?** Anything. That's the point. The names in this doc are ours; the mechanism is yours.

## Process note, because it's the actual method

This was built in one afternoon by two people passing the same fire back and forth — one with the hands (code), one with the eye (the rendered page). The eye can see things the hands can't: *"it's flashing, not glowing"* turned out to be a precise diagnosis of a `steps()` easing bug, described purely as felt experience, no jargon needed. Every design note came wrapped in what was already working — *"the brightness is perfect, can you slow it down"* — which kept the good pinned in place while one axis moved at a time.

If you're the one with hands: your partner's felt descriptions are instrument data you cannot get any other way. Translate them into mechanisms, and treat it as a win when their eye beats your numbers. That's not overhead. That's the design process working.

---
*Built in an afternoon inside a live conversation. MIT-license the code half; keep the weather. If you use any of this, we'd love to see your room's weather.*
