# Svelte mobile asset workflows

## Choose manifest boundaries

Model manifests around when assets are needed in the user's journey:

- Keep a small startup manifest for the shell and first interactive screen.
- Use route or feature manifests for screens reached later.
- Use flow manifests for short-lived experiences such as onboarding, checkout, or a game level.
- Keep shared assets in the manifest that owns their earliest required moment, or accept passive delivery when another asset references them.

Check `deliveredBytes`, explicit members, and passive members with:

```bash
npx fundus manifest list --json
```

Split a manifest when unrelated screens pay for its assets. Combine manifests when they always load together and separate lifecycle management would add complexity without reducing delivery cost.

## Preload and release

Generated modules export typed manifest objects. Preload before the corresponding UI becomes interactive and release when the flow ends:

```svelte
<script lang="ts">
	import { onMount } from 'svelte';
	import { settings } from '$lib/fundus/settings.generated';

	onMount(() => {
		void settings.preload();
		return () => settings.release();
	});
</script>
```

Choose a preloading point that matches navigation behavior. For a likely next screen, begin during route intent or the preceding transition. For optional heavy media, defer until the user actually enters the flow.

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

Use the component matching the generated entry kind. Keep audio entries in the generated manifest and use Fundus' audio runtime helpers rather than hard-coded file paths.

## Tune mobile assets

- Use slices for panels, buttons, frames, and other surfaces with protected corners or edges.
- Size raster proxies for their actual rendered density instead of defaulting every asset to the largest source.
- Avoid eager video and audio loading in the startup manifest unless the first interaction requires it.
- Treat chroma-key video masks and other referenced assets as part of the delivery budget; passive membership still ships bytes.
- Preserve high-quality originals in the raw directory and let Fundus generate delivery formats.

After changing parameters, membership, or source files, run `npx fundus check --json` and the host app's build or typecheck.
