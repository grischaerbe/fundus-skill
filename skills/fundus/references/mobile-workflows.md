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

Shared URLs are deduplicated by the Fundus runtime. Releasing one manifest does not dispose an asset still held by another preloaded manifest.

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

Use the component matching the generated entry kind. Fundus has no audio component; call `audioSourceOf(entry)` at playback time after preloading and pass the result to the host audio engine rather than hard-coding a proxy path.

## Draw into an existing canvas

Use `drawImage` and `drawSlice` from `fundus` when the host already renders into a canvas. Verify that the installed package exports these helpers by inspecting its runtime type declarations; older Fundus versions may not provide them. The installed runtime types are authoritative for rendering APIs, just as local CLI help is authoritative for commands.

Both helpers accept an `HTMLCanvasElement` or `CanvasRenderingContext2D`, followed by the typed entry and `x, y, width, height`. They return `void` and draw synchronously:

```ts
import { drawImage, drawSlice } from 'fundus';
import { main } from '$lib/fundus/main.generated';

// In the browser, with the host's existing 2D context `ctx`:
await main.preload();
drawSlice(ctx, main.navigationPanel, 20, 30, 300, 180);
drawImage(ctx, main.logo, 40, 50, 120, 60);
```

Apply these contracts:

- **Readiness and ownership:** `preload()` loads and decodes by default. Await it successfully before drawing and keep the manifest held for every frame that uses the assets. The lifecycle owner calls `release()` after the last consumer finishes. Repeated drawing needs no additional preload; the decoded source is reused. Preloaded images retain their decoded elements until release, so account for decoded memory as well as delivery bytes.
- **Missing sources:** the helpers never load or decode on demand. A non-empty draw throws if no decoded source is retained, including before preload completes, after the last holding manifest releases it, or when `image: { decode: false }` / `slice: { decode: false }` was used. Retrying the same manifest's `preload()` with different options does not upgrade its existing hold; choose decoding-enabled preload for canvas consumers from the outset.
- **Coordinates and DPR:** `(x, y)` is the top-left of the destination box in the context's current units. Use the host's existing DPR transform; do not multiply coordinates or sizes by DPR again. Fundus does not resize or clear the canvas, alter the transform, or change clipping, alpha, compositing, or smoothing. Resizing the backing store is the host's responsibility and clears its contents.
- **Asset pixel ratio:** a slice's `slicing.pixelRatio` determines its fixed logical dimensions, such as nine-slice borders; a 20-source-pixel border at @2x is 10 logical units. Do not divide the requested destination size by this ratio. Three-slice caps scale with the cross axis; image entries have no asset pixel-ratio metadata and stretch to the explicit destination size.
- **Overdraw:** the slice's destination box describes its core. Baked-in overdraw paints outside that box, including above/left of `(x, y)`; leave room in the canvas and any caller-owned clip. Painting shares the `<Slice>` component's geometry, while fractional sizes or transforms can antialias cell boundaries.
- **Invalid sizes:** non-positive width or height is a no-op; non-finite coordinates or dimensions throw.

The `<Slice>` component still loads automatically and does not require explicit preloading. Choose between the component and canvas helpers based on the host's rendering surface.

## Tune mobile assets

- Use slices for panels, buttons, frames, and other surfaces with protected corners or edges.
- Size raster proxies for their actual rendered density instead of defaulting every asset to the largest source.
- Avoid eager video and audio loading in the startup manifest unless the first interaction requires it.
- Treat chroma-key video masks and other referenced assets as part of the delivery budget; passive membership still ships bytes.
- Preserve high-quality originals in the raw directory and let Fundus generate delivery formats.

After changing parameters, membership, or source files, run `npm exec --no -- fundus check --json` and the host app's build or typecheck.
