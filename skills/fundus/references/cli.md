# Fundus CLI

## Contents

- [Set up](#set-up)
- [Inspect](#inspect)
- [Ingest and edit](#ingest-and-edit)
- [Choose an update](#choose-an-update)
- [Figma sources](#figma-sources)
- [Tag Figma nodes from an agent](#tag-figma-nodes-from-an-agent)
- [Declare cross-package dependencies](#declare-cross-package-dependencies)
- [Organize](#organize)
- [Build and check](#build-and-check)

## Set up

`fundus init --json` (only without a config) writes `fundus.config.ts`, `assets/fundus.library.json` with an empty `main` manifest, and `assets/raw/`. Proxies go to `static/fundus`, served at `/fundus/` (if `kit.paths.base` is set, include it in `proxyBasePath`); modules go to `manifestsDir`, `src/lib/fundus`, imported as `$lib/fundus/<kebab-name>.generated`. Run `fundus build --json` right after init: `check` fails until the modules exist. No manifest is special: the host decides what preloads at startup.

## Inspect

```bash
fundus state --json             # first read: assets, parameters, explicit/passive memberships, proxy state,
                                # references, requiredBy, warnings, issues, preset names
fundus asset list --json        # filters: --manifest <name>, --type <type>, --warnings-only
fundus asset show <id> --json   # includes importSource and requiredBy
fundus manifest list --json     # explicit and passive members, deliveredBytes
fundus folder list --json
fundus usage [<id> | --unused] --json   # host source references; needs usage.referenceProvider/openLocation in config
```

## Ingest and edit

```bash
fundus asset ingest ./exports/panel@2x.png --type slice --manifests main --json
fundus asset replace navigationPanel ./exports/panel-v2.png --json
fundus asset set navigationPanel --parameters '{"mode":"three-horizontal"}' --manifests=main,settings --json
fundus asset rename sidebarPanel navigationPanel --json
```

- **Type:** `--type` is needed for everything but audio (raster files are `image` or `slice`, video files `video` or `chroma-key-video`). Never pick it from the user's wording ("button background" suggests a slice): propose, confirm, then pass it.
- **Id:** defaults to the camelCased file name without `@2x`-style suffixes (`logo@2x.png` → `logo`); `--id` overrides.
- **Parameters:** `--parameters` is a shallow top-level merge. A nested object replaces the whole field and must be complete (a partial `chromaKey` is rejected: copy it from `asset show`, edit, send it all); `null` clears an optional field; `{"$asset": "<id>"}` references an asset, which must exist first. For unknown shapes, copy an existing asset of that type from `state` (which also lists preset names); validation errors name bad fields. Never guess.
- **Memberships:** `--manifests` sets the initial set on `ingest` and replaces the whole set on `set`, so read the current memberships before adding one; `--manifests=` clears. Every name must exist.
- **Slices:** parameters are flat — `mode` (`nine`, `three-horizontal`, `three-vertical`, `one`), `insets` and optional `overdraw` (`{top, right, bottom, left}` in source pixels), `pixelRatio`, optional `compression`, `preset`. A local ingest seeds `mode: nine`, guesses insets (¼ per side), and reads `pixelRatio` only from an `@Nx`/`_Nx`/`-Nx` suffix (else 1): set real values or have the user author them in the editor. Insets and overdraw never rescale, so rescale them whenever the source density changes; a local-file replace keeps the old `pixelRatio`. `--compression-x`/`-y` (0–100, on `ingest` and `set`; both 0 removes) shrink stretch regions to cut bytes.
- **Video:** proxies are H.264 without alpha and keep audio (AAC 128 kbps). Ask for a solid-backdrop render of transparent video and ingest it as `chroma-key-video`; strip silent audio first; cap size with `maxWidth`.
- **Chroma key:** all values are 0–1; `keyColor` is RGB and out-of-range values clamp silently (`[0,177,64]` → cyan); `despillCoverage` < 1 protects key-colored artwork. Keying changes rewrite only the module, never re-encode. `mask` and `fallback` are top-level references to `image` assets that ship as passive members (no membership needed) and appear as `entry.mask`/`entry.fallback`.

## Choose an update

Read `asset.importSource` in `asset show --json` first. A Figma source (`importer: "figma"`) is a **tag** source when `parameters.mode` is `"tag"`; tag sources also store the last resolved `nodeId`, so a node id alone does not mean node link.

| Current source | Goal | Command |
| --- | --- | --- |
| Figma node link | Refresh, or change link/scale | `asset reimport <id> [--figma-link <url>] [--scale <n>]` |
| Figma tag | Refresh, or change file/scale/tag | `asset reimport <id> [--figma-file <key>] [--scale <n>] [--figma-tag <new>]` — a new tag renames the asset and must already be on the node: [Rename a tag asset](#rename-a-tag-asset) |
| Any Image/Slice | New Figma node | `asset replace <id> <url> --from figma [--scale <n>]` |
| Any Image/Slice | Switch to its own tag | [Move a node-link asset to a tag](#move-a-node-link-asset-to-a-tag) |
| Any | New local file | `asset replace <id> <file>` |

- A plain `reimport` re-resolves the saved source; a tag asset looks up the node carrying its tag again, so it follows node moves. Overrides keep omitted values. Node-link assets reject `--figma-tag`/`--figma-file`; tag assets reject `--figma-link`.
- `replace` keeps the id, type, parameters, references, folder, and memberships. A local file makes local bytes authoritative and ends refreshes; a Figma replace installs a new refreshable source.
- **Slice `pixelRatio`:** a Figma replace, and a reimport with any flag — even `--figma-file` alone — resets it to the export scale; only a flagless reimport keeps it. Other parameters are kept. Inspect the returned asset before more parameter changes.
- `reimport` and Figma `replace` abort instead of overwriting drift in their own raw file or a concurrent change.

## Figma sources

Configure Figma only when the user asked. The token needs `file_content:read` and `file_dev_resources:read`, plus `file_dev_resources:write` for `fundus figma set-tag`/`delete-tag`. It lives in a server-side environment variable — never commit, print, persist, or pass it as an argument, and recommend rotating one pasted into chat. A 403 names the missing scope. `build` and `check` need no token:

```ts
export default defineConfig({
	import: {
		figma: {
			token: process.env.FIGMA_ACCESS_TOKEN,
			defaults: { scale: 2 },
			defaultFileKey: process.env.FIGMA_FILE_KEY // used when --figma-file/--file is omitted
		}
	}
});
```

The CLI does not load `.env`: export the token in the shell. On a rate-limit error, stop and tell the user; never retry in a loop.

`--from figma` is always explicit; a URL positional never implies it. Figma creates Image or Slice assets only. Omitted `--scale` (1–4) uses `defaults.scale`. `<Slice>` never renders above `maxDpr` (default 2), so exporting slices above scale 2 wastes bytes unless the host raises `maxDpr`. Folder, manifest, and parameter flags work as for files.

```bash
# Node link: --type and --id are required.
fundus asset ingest 'https://www.figma.com/design/abc/UI?node-id=12-34' \
  --from figma --type slice --id navigationPanel --scale 2 --manifests main --json

# Tag: the tag IS the asset id — no --id, no positional link.
fundus asset ingest --from figma --figma-tag navigationPanel --type slice --figma-file abc --json
```

A **tag** is an asset id attached to a node as a Dev Resources link (`https://fundus.invalid/asset/<tag>`, named `Fundus: <tag>`). Fundus lists the file's links at import time, so a tag survives node moves and file duplication; node ids do not. Pasting a node into another file drops its tag: tag it again there. Designers set tags with the Fundus Figma plugin; agents use the flow below. Cmd+D in Figma copies the link, so a duplicated node makes the tag fail with `ambiguous-tag` ("More than one Figma node is tagged"): `set-tag --force` on the node that should keep it removes the tag from the copies.

## Tag Figma nodes from an agent

`fundus figma set-tag` checks the request against the file's links and the library, then writes the link over the REST API. No Figma MCP is involved.

```bash
fundus figma set-tag --file abc --node-id 12-34 --tag coinBadge --json
```

1. Choose the tag like an asset id: stable camelCase naming what it is. `--file` takes a key or link (default `defaultFileKey`); `--node-id` takes `12:34` or `12-34`. Tag the instance you placed or the node in the main component — never a layer inside an instance (`I12:34;56:78`), a page, or the document, which Fundus refuses.
2. A node that already carries the tag is left as is (success, nothing written). Exit 1 with `existingTags` is a refusal; read `error.message`. `--force` overrides the refusals below — use it only for the outcome the user wants, and relay what it did. Anything else (bad id, missing or deleted node, missing file key, token or fetch failure): fix the input.

   | Refusal | Without `--force` | With `--force` |
   | --- | --- | --- |
   | Other nodes carry the tag | Pick another name that fits `existingTags` (tag, node id, node name) | Removes the tag from the others (`movedFrom`) and keeps it on this node; its asset resolves here on the next reimport |
   | An asset with that id exists from another source | Pick another name | Tags anyway; only to switch that asset, run the warned `asset replace <id> --from figma --figma-tag <id> --figma-file <key> --scale <n>`, which drops its old source |
   | Node carries another tag | Keep it | Replaces it; if the old tag feeds an asset, finish with `asset reimport <old> --figma-tag <new>` ([rename](#rename-a-tag-asset)) |

3. On `{"ok": true, "nodeId", "tag", "previousTag", "movedFrom", "warnings", "next"}`, relay every warning and `next`. The node's annotation appears once someone opens its page in the Fundus Figma plugin; never write it yourself.
4. A failed write exits 1 with `written` and `notWritten`: nothing was lost (the new link is written first). Rerun the same command to finish.
5. Import from the same file: `fundus asset ingest --from figma --figma-tag coinBadge --type slice --figma-file abc --json`.

Tag one node per command. Figma throttles rapid writes; on a rate-limit error, stop and tell the user.

### Move a node-link asset to a tag

Tag the asset's own node (same file) with the asset id — not a conflict, no `--force` — then replace from the tag with `--scale` set to its current `importSource.parameters.scale` (omitted, it uses `defaults.scale` and resets a slice's `pixelRatio`). The follow-up that `set-tag` warns with already carries it. The id, type, folder, other parameters, references, and memberships stay.

```bash
fundus figma set-tag --file abc --node-id 12-34 --tag navigationPanel --json
fundus asset replace navigationPanel --from figma --figma-tag navigationPanel --figma-file abc --scale 2 --json
```

A tag replace must use the asset's own id. Never `asset ingest` (the id collides) and never rename assets to get past a refusal.

### Rename a tag asset

The tag is the id, so renaming `navigationPanel` to `mainNavigationPanel` means changing the tag. `asset rename` refuses tag assets, and `reimport --figma-tag <new>` looks up the node that already carries `<new>`, so retag the node first, then reimport. Update host usages of `navigationPanel` in the same task:

```bash
fundus figma set-tag --file abc --node-id 12-34 --tag mainNavigationPanel --force --json
fundus asset reimport navigationPanel --figma-tag mainNavigationPanel --json
```

### Remove a tag

Only when asked: `fundus figma delete-tag --tag navigationPanel --json` (or `--node-id`; both must match if given). It refuses when the tag is on several nodes (pass `--node-id`), when a node carries several tags and only `--node-id` is given (pass `--tag`), when no existing node carries it, and, without `--force`, when an asset imports from the tag and no other node keeps it; disconnecting that asset with a local-file `replace` first avoids breaking its reimport. The Fundus Figma plugin removes the annotation once it opens the node's page.

## Declare cross-package dependencies

A package can require assets a host must provide, by `id` and `type` only; bytes, parameters, and manifests stay in the host. Declare them in a `.ts`/`.js` module (JSON cannot carry the literal types):

```ts
import { defineDependencies, type ManifestContract } from 'fundus/config';

const dependencies = defineDependencies([{ id: 'coin', type: 'slice' }]);
export type CoinContract = ManifestContract<typeof dependencies>; // exact readonly entry map
export const inject = (manifest: CoinContract) => { /* manifest.coin is a typed slice entry */ };
export default dependencies;
```

The host passes a generated manifest (`inject(main)`); a manifest lacking `coin` or with the wrong kind is a compile error. A package exposes the module through `package.json`: `{ "fundus": { "dependencies": "./src/fundus.deps.ts" } }`. The host lists sources as bare package names or paths (`.`/`/` prefix or `.ts`/`.js` extension; no package subpaths):

```ts
export default defineConfig({ dependencies: ['@org/other-package', './src/fundus.deps.ts'] });
```

- Flat: a package's own dependencies are not resolved.
- Lower bound: extra host assets are fine; a missing id or wrong type is a hard, library-global failure in `check`, `build`, and the editor banner.
- The same id with conflicting types across sources fails at config load, naming both sources.
- Validation checks only id and type. `ManifestContract` needs the asset as an explicit member of the manifest the host injects — passive members are never top-level (an asset in no manifest is also an unreachable warning). When a declared id is added or renamed, satisfy it in the host (camelCase id, type, membership) in the same task and run `check`; check an asset's `requiredBy` before renaming or deleting it.

## Organize

```bash
fundus folder create UI --json               # parents must exist
fundus folder create UI/Settings --json      # always '/' separators
fundus asset move settingsPanel UI/Settings --json
fundus manifest create settings --json
fundus manifest set settings --color blue --emoji '⚙️' --json   # colors: manifest palette
```

Also: `asset delete`, `manifest rename|delete` (a rename replaces the module file and export name), `folder rename|move`, `folder delete --recursive`. An asset in no manifest and unreferenced by a delivered asset is never processed or delivered and fails `check` as unreachable. `folder delete --recursive` does not check references: inspect every asset in the subtree first.

## Build and check

`build` processes stale or missing proxies, prunes orphans, and regenerates modules; it does nothing and exits 1 while validation, raw drift, or unmet dependencies fail. `check` additionally requires proxies and modules to match an in-memory regeneration.

`potential-image-duplicate` is a perceptual match between two `image` assets (e.g. idle and pressed states) and cannot be acknowledged per pair: compare them in the editor, then remove the duplicate or ask the user to accept `--allow-warnings`.

Proxy names hash the raw file, parameters, and resolved preset settings, and staleness is a name check. Editing a preset therefore turns every affected proxy `proxy-missing` until `build` (commit the result). Hand-edited proxies are not stale but cause `module-out-of-sync` (and runtime size mismatches); `build` will not restore them — restore them from git, then `build`.

Deleting: `usage --unused` lists only conclusively unused assets (`unknown` is not unused). `asset delete` removes the record, raw original, and proxies, recoverable only from git; `folder delete --recursive` deletes every asset in the subtree. Present candidates to the user and delete only what they confirm.
