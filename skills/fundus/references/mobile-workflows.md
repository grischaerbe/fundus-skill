# Svelte mobile asset workflows

## Plan delivery

Model manifests on when the user's journey needs assets: a small startup manifest for the shell and first interactive screen, route or feature manifests for later screens, and flow manifests for short-lived experiences (onboarding, checkout, a game level). A shared asset belongs to the manifest of its earliest required moment, or ships passively with an asset that references it. Keep video and audio out of startup unless the first interaction needs them. Split a manifest when unrelated screens pay for its assets; merge manifests that always load together. `deliveredBytes` counts only current proxies, so `build` before measuring.

Use slices for stretchable surfaces (panels, buttons, frames) instead of fixed-size variants, and size raster proxies for their rendered density. Keep high-quality originals raw; tune delivery with parameters, presets, and slice compression.

## Preload and release

A manifest holds each URL once no matter how often it receives `preload()`, so give it one route- or layout-level owner. If ownership cannot be centralized, wrap it in a host-side lease counter that calls `release()` only when consumers go from one to zero.

```svelte
<script lang="ts">
	import { onMount } from 'svelte';
	import { settings } from '$lib/fundus/settings.generated';

	let preloadState = $state<'loading' | 'ready' | 'error'>('loading');

	onMount(() => {
		let active = true;
		settings.preload().then(
			() => { if (active) preloadState = 'ready'; },
			(error) => {
				if (active) preloadState = 'error';
				console.error('Failed to preload settings assets', error);
			}
		);
		return () => {
			active = false;
			settings.release();
		};
	});
</script>
```

Preload only in the browser (`onMount`, or behind `browser` in a universal `load`): server-side relative fetches throw and break SSR and prerendering. Serve `/fundus/` untransformed (no CDN image optimization): loads reject any byte-size change. Proxy names are content-hashed, so cache them as immutable.

When one manifest replaces another (lobby → game), `preload()` the next before releasing the previous so shared assets are not refetched. `release()` during an in-flight preload is safe, and a failed preload can be retried.

Render asset-dependent UI only when `ready`; offer retry or a fallback on `error`. Preload a likely next screen during route intent or the preceding transition; defer optional heavy media until the user enters the flow. Shared URLs are deduplicated: releasing one holder (manifest or handle) never disposes an asset another still holds.

## Render typed entries

Import components and generated entries; never assemble proxy URLs. Components load automatically and need no handle.

```svelte
<script lang="ts">
	import { Image, Slice, Video, ChromaKeyVideo } from 'fundus';
	import { main } from '$lib/fundus/main.generated';
</script>

<Image src={main.logo} />
<Slice src={main.navigationPanel} />
```

`Image` passes through `<img>` attributes and defaults `width`/`height` to proxy pixels, so size @2x/@3x images with CSS or props, and cap delivered pixels with the `targetWidth` parameter (it also upscales). `Slice` takes `src`, `alt`, `class`, `style`, and `maxDpr` (default 2) and fills its container, so give the container a size. There is no audio component: retain the entry and give the handle's `source` URL to the host audio engine for as long as playback reads it.

### Size slice boxes

`<Slice>`, `drawSlice`, and `sliceGrid` fill whatever box they get. Size the box with `sliceLayoutGeometry(entry)` — it takes the plain entry (`main.panel` or `handle.entry`) and returns the element core's natural `width`/`height` in CSS pixels plus an `aspectRatio` string — never with `sourceWidth / slicing.pixelRatio`, which ignores baked-in overdraw and stretch compression.

- **Three-slice:** the cross axis (`height` for `three-horizontal`, `width` for `three-vertical`) is a hard constraint; caps keep their aspect ratio only at that size. The along-axis value is a reference; fixing the element to it defeats the slice.
- **Nine-slice:** both values are references; any size works.
- **Compressed slices:** it returns the uncompressed size.

## Retain individual entries

Two APIs hold one entry's file, sharing loads with manifests and each other:

- `new RetainedEntry(entry)` — Svelte class; the hold follows the constructing component.
- `retainEntry(entry)` — resolves to an `EntryHandle` (`entry`, `source`, `release()`) for code outside component lifecycles.

`source` is `HTMLImageElement | ImageBitmap` for images and slices, and a playable object URL over the fully fetched file for video, chroma-key video, and audio. A chroma-key video retains only its video; retain `mask` or `fallback` separately. An already preloaded or in-flight asset is shared, not refetched; a retain after `manifest.preload()` resolves without loading but is still its own hold. Both `release()`s are idempotent and safe to pass unbound.

### In components

Construct `RetainedEntry` during component initialisation (elsewhere it throws). It loads on mount — never during SSR — and releases on destroy, including a load still in flight.

```svelte
<script lang="ts">
	import { RetainedEntry, drawSlice } from 'fundus';
	import { main } from '$lib/fundus/main.generated';

	const chime = new RetainedEntry(main.chime);
	const panel = new RetainedEntry(main.navigationPanel);
	let canvas: HTMLCanvasElement;

	$effect(() => {
		if (panel.source) drawSlice(canvas, panel, 20, 30, 300, 180);
	});
</script>

<canvas bind:this={canvas}></canvas>
{#if chime.status === 'error'}<p>Audio unavailable</p>{/if}
<audio src={chime.source}></audio>
```

- `source` is `undefined` while loading, after an error, and after release; reading it in an effect or template re-runs once it loads. `status` is `loading | ready | error | released` (reason in `error`).
- The entry is fixed per instance; use `{#key entry}` to retain a different one.
- `release()` drops the hold early.

### Outside components

- Call `release()` once the last consumer stops reading `source`. Reading `source` after release throws.
- A rejected `retainEntry` leaves no hold. When retaining several, release the ones that resolved if another rejects.
- An un-awaited promise still takes its hold: if the consumer can vanish first, release the handle when it arrives. In components, use `RetainedEntry` instead.

## Draw into an existing canvas

`drawImage`/`drawSlice(target, retained, x, y, width, height)` take a canvas or 2D context, an `EntryHandle` or `RetainedEntry` of the matching kind, and a box, and draw synchronously.

```ts
import { drawImage, drawSlice, retainEntry } from 'fundus';
import { main } from '$lib/fundus/main.generated';

const panel = await retainEntry(main.navigationPanel);
const logo = await retainEntry(main.logo);
drawSlice(ctx, panel, 20, 30, 300, 180);
drawImage(ctx, logo, 40, 50, 120, 60);
// after the last frame that uses them:
panel.release();
logo.release();
```

- **Readiness:** drawing never waits or loads. Keep the hold for every frame; drawing without a loaded source (released, or a `RetainedEntry` still loading) throws, even for an empty box. Held images and slice bitmaps stay decoded in memory — budget that as well as bytes.
- **Units:** `(x, y)` is the box's top-left in the context's current units. Rely on the host's DPR transform; never multiply by DPR again. Fundus never resizes or clears the canvas or changes transform, clipping, alpha, compositing, or smoothing; sizing the canvas is the host's job: CSS size = logical box, backing store = CSS size × DPR (rounded); resizing clears it.
- **Pixel ratio:** a slice's `slicing.pixelRatio` fixes logical border sizes (20 source px at @2x = 10 units); do not divide the destination size by it. Three-slice caps scale with the cross axis. Images have no pixel ratio and stretch to the box.
- **Overdraw:** the box is the slice's core; baked-in overdraw paints outside it, including above/left of `(x, y)`. Leave room in the canvas and any clip. Fractional sizes or transforms can antialias cell seams.
- Non-positive width/height is a no-op; non-finite values throw.

## Render with WebGL or Pixi

Fundus ships no renderer adapter. Build one from a retained entry's `source` and `sliceGrid(entry, x, y, width, height)` (pure; takes the plain entry), which returns exactly what `drawSlice` paints: per axis, four source edges (`sourceX`, `sourceY`, source px) and four destination edges (`destX`, `destY`) of a 3×3 grid. Do not use Pixi's `NineSliceSprite` (no overdraw, fixed three-slice caps, corners shrink in small boxes); build a `Mesh` from the grid.

- **Cells:** paint cell `(c, r)` only when all four spans are positive (`sourceX[c+1] > sourceX[c]`, same for `sourceY`, `destX`, `destY`). This handles three-slice, one-slice, and insets without a center.
- **UVs:** source edges ÷ `entry.sourceWidth`/`sourceHeight`. A 4×4 vertex mesh works if the index buffer omits skipped cells (otherwise a zero-width span stretches a texel column). On resize, recompute positions and indices.
- **Units:** pass device pixels for seam-free edges. Overdraw and boxes smaller than the fixed borders put edges outside the box.
- **Alpha:** upload with `UNPACK_PREMULTIPLY_ALPHA_WEBGL = true`; slice bitmaps are already premultiplied, so every texture ends up premultiplied.
- **Lifetime:** keep the handle while the renderer may re-upload (context loss, texture GC); destroy textures before releasing. After the last release a new handle gets a new source object, so don't cache by source identity across releases.
- **Origin:** retained elements have no `crossOrigin`; cross-origin proxies cannot be uploaded from images or one-slices.
- **Images:** a textured quad; no grid.
