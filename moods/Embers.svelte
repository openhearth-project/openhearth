<script lang="ts">
	// Flying embers — the Ember mood's fire, coming off the floor.
	// Cousins of the Build Room's fireflies, but they RISE: sparks lifting off
	// the low firelight, swaying as they go, cooling and winking out on the way.
	// Purely decorative — pointer-events: none, sits behind the content.
	//
	// `boost` is the screenshot-room dial (/chat/demo only): more sparks,
	// faster climbs, bigger and brighter, short delays so the screen fills
	// within seconds instead of half a minute. The live Warren never sees it.
	let { count = 26, boost = false } = $props();

	type Spark = {
		left: number; // vw — where it lifts off the floor
		size: number; // px
		delay: number; // s
		dur: number; // s — how long the whole climb takes
		sway: number; // px — how far it wanders sideways on the way up
		rise: number; // vh — how high it gets before it cools out
		hue: number;
		blur: number; // px
	};

	function make(n: number, hot: boolean): Spark[] {
		const sparks: Spark[] = [];
		for (let i = 0; i < n; i++) {
			const big = Math.random() < (hot ? 0.32 : 0.18);
			sparks.push({
				left: Math.random() * 100,
				size: hot
					? (big ? 4 + Math.random() * 3 : 2.2 + Math.random() * 2.4)
					: (big ? 3 + Math.random() * 2.5 : 1.4 + Math.random() * 1.8),
				// boost: short delays so a screenshot never catches an empty sky
				delay: Math.random() * (hot ? 7 : 26),
				// slow. embers don't hurry — 18–34s for a full climb
				// (boosted: 8–16s — visibly moving in a screenshot session)
				dur: hot ? 8 + Math.random() * 8 : 18 + Math.random() * 16,
				sway: (Math.random() - 0.5) * 130,
				// they climb the whole room now — most clear the ceiling,
				// only a few give out in the top third
				rise: big ? 118 + Math.random() * 20 : 92 + Math.random() * 40,
				// copper → gold, with the odd hot pale one
				hue: Math.random() < 0.15 ? 44 + Math.random() * 8 : 24 + Math.random() * 14,
				blur: big ? 0 : Math.random() * (hot ? 0.7 : 1.2)
			});
		}
		return sparks;
	}

	// derived, not const — the field regenerates if boost flips while the
	// component is alive (walking into/out of the demo room mid-Ember)
	const sparks = $derived(make(boost ? Math.round(count * 1.7) : count, boost));
</script>

<div class="ember-field" class:boost aria-hidden="true">
	{#each sparks as s, i (i)}
		<span
			class="spark"
			style="
				left:{s.left}vw;
				--size:{s.size}px;
				--delay:{s.delay}s; --dur:{s.dur}s;
				--sway:{s.sway}px; --rise:{s.rise}vh;
				--hue:{s.hue}; --blur:{s.blur}px;
			"
		></span>
	{/each}
</div>

<style>
	.ember-field {
		position: fixed;
		inset: 0;
		/* above the room's content — sparks drift OVER the words.
		   pointer-events:none means nothing is ever blocked, and the screen
		   blend means they can only add light, never obscure text. */
		z-index: 60;
		pointer-events: none;
		overflow: hidden;
		/* screen blend: these can only ever ADD light, never muddy anything */
		mix-blend-mode: screen;
	}

	.spark {
		position: absolute;
		bottom: -4vh;
		width: var(--size);
		height: var(--size);
		border-radius: 50%;
		background: radial-gradient(
			circle,
			hsla(var(--hue), 100%, 82%, 0.98) 0%,
			hsla(var(--hue), 96%, 62%, 0.5) 45%,
			transparent 72%
		);
		box-shadow: 0 0 7px 2px hsla(var(--hue), 95%, 58%, 0.32);
		filter: blur(var(--blur));
		opacity: 0;
		will-change: transform, opacity;
		animation:
			lift var(--dur) linear var(--delay) infinite,
			cool var(--dur) ease-in-out var(--delay) infinite;
	}

	/* The climb. Sways side to side on the way up — a spark riding the draft,
	   not a bubble going straight. Ends fully cooled, off the top. */
	@keyframes lift {
		0% {
			transform: translate(0, 0) scale(1);
		}
		25% {
			transform: translate(calc(var(--sway) * 0.55), calc(var(--rise) * -0.26)) scale(0.96);
		}
		50% {
			transform: translate(calc(var(--sway) * -0.35), calc(var(--rise) * -0.52)) scale(0.86);
		}
		75% {
			transform: translate(calc(var(--sway) * 0.8), calc(var(--rise) * -0.78)) scale(0.72);
		}
		100% {
			transform: translate(calc(var(--sway) * 0.3), calc(var(--rise) * -1)) scale(0.55);
		}
	}

	/* Brightens as it leaves the fire, then cools out. Never a hard cut. */
	@keyframes cool {
		0% {
			opacity: 0;
		}
		12% {
			opacity: 0.9;
		}
		38% {
			opacity: 0.6;
		}
		56% {
			opacity: 0.82;
		}
		74% {
			opacity: 0.5;
		}
		88% {
			opacity: 0.62;
		}
		100% {
			opacity: 0;
		}
	}

	/* Screenshot-room heat: hotter halo on every spark. Screen blend still
	   applies, so even boosted they can only ADD light, never dim text. */
	.ember-field.boost .spark {
		box-shadow: 0 0 12px 4px hsla(var(--hue), 95%, 60%, 0.55);
	}

	@media (prefers-reduced-motion: reduce) {
		.spark {
			animation: none;
			opacity: 0.35;
		}
	}
</style>
