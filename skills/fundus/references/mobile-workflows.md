# Svelte mobile asset workflows

## Choose manifest boundaries

Model manifests around when assets are needed in the user's journey:

- Keep a small startup manifest for the shell and first interactive screen.
- Use route or feature manifests for screens reached later.
- Use flow manifests for short-lived experiences such as onboarding, checkout, or a game level.
- Keep shared assets in the manifest that owns their earliest required moment, or accept passive delivery when another asset references them.

Check `deliveredBytes`, explicit members, and passive members with:

```bash
npm exec --no -- fundus manifest list --json
```

Capture this output before and after membership or processing changes. Report the `deliveredBytes` delta for every affected manifest. Split a manifest when unrelated screens pay for its assets. Combine manifests when they always load together and separate lifecycle management would add complexity without reducing delivery cost.

## Preload and release

Generated modules export singleton manifest objects. A manifest holds each URL once regardless of how many times that same object receives `preload()`. Give it one route- or layout-level lifecycle owner; repeated component instances must not each call `release()` because the first cleanup would release the shared hold. If ownership cannot be centralized, wrap the manifest in a host-side lease counter and call `release()` only when the consumer count transitions from one to zero.

Handle preload failure explicitly and expose readiness before rendering UI that requires the assets:

```svelte
<script lang="ts">
	import { onMount } from 'svelte';
	import { settings } from '$lib/fundus/settings.generated';

	let preloadState = $state<'loading' | 'ready' | 'error'>('loading');

	onMount(() => {
		let active = true;
		settings.preload().then(
			() => {
				if (active) preloadState = 'ready';
			},
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

Render the asset-dependent screen only when `preloadState === 'ready'`, and show a retry or fallback for `error`. Choose a preloading point that matches navigation behavior. For a likely next screen, begin during route intent or the preceding transition. For optional heavy media, defer until the user actually enters the flow; still handle rejection even when the preload is only opportunistic.

Shared URLs are deduplicated by the Fundus runtime. Releasing one holder — a manifest or a retained handle — does not dispose an asset another holder still keeps.

## Render typed entries

Import runtime components and generated entries instead of assembling raw proxy URLs:

```svelte
<script lang="ts">
	import { Image, Slice, Video, ChromaKeyVideo } from 'fundus';
	import { main } from '$lib/fundus/main.generated';
</script>

<Image src={main.logo} />
<Slice src={main.navigationPanel} />
```

Use the component matching the generated entry kind. Fundus has no audio component; retain the entry and pass the handle's `source` URL to the host audio engine rather than hard-coding a proxy path. Keep the handle until playback no longer reads the URL.

## Retain individual entries

Two APIs hold one entry's own file, sharing loads with manifests and each other:

- `RetainedEntry`: a Svelte class for components. The hold follows the component that constructs it.
- `retainEntry(entry)`: a promise of an `EntryHandle` (`entry`, `source`, `release()`) for code outside component lifecycles.

Verify the installed runtime types export them. Fundus versions before `RetainedEntry` name the handle type `RetainedEntry` instead of `EntryHandle`, and older versions still take raw entries in the drawing helpers and export `audioSourceOf`. The installed runtime types are authoritative for rendering APIs, just as local CLI help is authoritative for commands.

- **Source types:** `HTMLImageElement | ImageBitmap` for images and slices; a playable URL (an object URL over the fully fetched file) for video, chroma-key video, and audio. A chroma-key video retains only its video; retain `mask` or `fallback` separately.
- **Loading:** an asset already preloaded or in flight is shared, not refetched. After `manifest.preload()` a retain resolves without loading but is still a separate hold.

### In Svelte components

Construct `RetainedEntry` during component initialisation; constructing it elsewhere throws. It starts loading on mount, so server rendering never loads, and releases on destroy, including a load still in flight.

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
{#if chime.status === 'error'}
	<p>Audio unavailable</p>
{/if}
<audio src={chime.source}></audio>
```

- **Reactive state:** `source` is `undefined` until loaded, after an error, and after release. `status` is `loading`, `ready`, `error` (reason in `error`), or `released`. Reading `source` in an effect or template re-runs it once the source loads.
- **Fixed entry:** the entry cannot change for an instance. Key the component (`{#key entry}`) to retain a different entry.
- **Early release:** `release()` drops the hold before destroy. It is idempotent and safe to pass unbound.

### Outside components

- **Release:** call `release()` once the last consumer stops reading `source`. It is idempotent and safe to pass unbound. Reading `source` after release throws.
- **Failure:** a rejected `retainEntry` leaves no hold. When retaining several entries, release the ones that resolved if another rejects.
- **Teardown mid-load:** an un-awaited promise still takes its hold. When the consumer can disappear before the promise settles, release the handle once it arrives. In a Svelte component, use `RetainedEntry` instead of handling this by hand.

## Draw into an existing canvas

`drawImage` and `drawSlice` accept an `HTMLCanvasElement` or `CanvasRenderingContext2D`, an `EntryHandle` or `RetainedEntry` of the matching kind, and `x, y, width, height`. They return `void` and draw synchronously:

```ts
import { drawImage, drawSlice, retainEntry } from 'fundus';
import { main } from '$lib/fundus/main.generated';

// In the browser, with the host's existing 2D context `ctx`:
const panel = await retainEntry(main.navigationPanel);
const logo = await retainEntry(main.logo);
drawSlice(ctx, panel, 20, 30, 300, 180);
drawImage(ctx, logo, 40, 50, 120, 60);

// After the last frame that uses them:
panel.release();
logo.release();
```

Apply these contracts:

- **Readiness:** drawing never waits or loads. An `EntryHandle` guarantees a decoded source; a `RetainedEntry` has one only once `source` is set. Keep either unreleased for every frame that draws it; drawing without a loaded source (released, or a `RetainedEntry` still loading) throws, even for an empty box. Decoded images and slice bitmaps stay in memory while held, so account for decoded memory as well as delivery bytes.
- **Coordinates and DPR:** `(x, y)` is the top-left of the destination box in the context's current units. Use the host's existing DPR transform; do not multiply coordinates or sizes by DPR again. Fundus does not resize or clear the canvas, alter the transform, or change clipping, alpha, compositing, or smoothing. Resizing the backing store is the host's responsibility and clears its contents.
- **Asset pixel ratio:** a slice's `slicing.pixelRatio` determines its fixed logical dimensions, such as nine-slice borders; a 20-source-pixel border at @2x is 10 logical units. Do not divide the requested destination size by this ratio. Three-slice caps scale with the cross axis; image entries have no asset pixel-ratio metadata and stretch to the explicit destination size.
- **Overdraw:** the slice's destination box describes its core. Baked-in overdraw paints outside that box, including above/left of `(x, y)`; leave room in the canvas and any caller-owned clip. Painting shares the `<Slice>` component's geometry, while fractional sizes or transforms can antialias cell boundaries.
- **Invalid sizes:** non-positive width or height is a no-op; non-finite coordinates or dimensions throw.

The `<Slice>` component still loads automatically and needs no handle. Choose between the component and canvas helpers based on the host's rendering surface.

## Render with WebGL or Pixi

Fundus ships no renderer adapter. Build one in host code from a retained entry (`EntryHandle` or `RetainedEntry`) and `sliceGrid(entry, x, y, width, height)`, which returns the exact geometry `drawSlice` paints: four source edges (`sourceX`, `sourceY`, in source pixels) and four destination edges (`destX`, `destY`) per axis for a 3×3 grid.

Do not substitute Pixi's `NineSliceSprite`: it has no overdraw, fixes three-slice caps, and shrinks corners in small boxes. Use a `Mesh` built from the grid.

- **Cells:** paint cell `(c, r)` only when all four spans are positive: `sourceX[c + 1] > sourceX[c]`, likewise for `sourceY`, `destX`, and `destY`. Skipped cells cover three-slice and one-slice modes and insets without a source center.
- **UVs:** divide source edges by `entry.sourceWidth` and `entry.sourceHeight`. A 4×4 vertex mesh works when its index buffer omits skipped cells; otherwise a zero-width source span stretches one texel column. On resize, recompute positions and the index buffer.
- **Units:** pass device pixels for seam-free edges. Overdraw and boxes smaller than the fixed borders place edges outside the requested box.
- **Alpha:** always upload with `UNPACK_PREMULTIPLY_ALPHA_WEBGL` set to `true`. Image elements honor it; slice bitmaps are already premultiplied and ignore it. Every texture ends up premultiplied.
- **Lifetime:** GPU upload copies pixels, but keep the handle while the renderer may re-upload, such as after a lost context or lazy texture garbage collection. Destroy textures before releasing.
- **Origin:** retained elements carry no `crossOrigin`; proxies hosted on another origin cannot be uploaded from images or one-slices.
- **Images:** a plain textured quad sized to the destination box; no grid needed.

## Tune mobile assets

- Use slices for panels, buttons, frames, and other surfaces with protected corners or edges.
- Size raster proxies for their actual rendered density instead of defaulting every asset to the largest source.
- Avoid eager video and audio loading in the startup manifest unless the first interaction requires it.
- Treat chroma-key video masks and other referenced assets as part of the delivery budget; passive membership still ships bytes.
- Preserve high-quality originals in the raw directory and let Fundus generate delivery formats.

After changing parameters, membership, or source files, run `npm exec --no -- fundus check --json` and the host app's build or typecheck.
