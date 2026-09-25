# Fundus CLI workflow

Use the host package manager's local-only runner and run commands from the host project root. The examples below use npm's `npm exec --no -- fundus`; adapt the runner to the detected lockfile without allowing an implicit registry download. Use `<runner> <command> --help` as the source of truth for the current Fundus CLI.

## Contents

- [Verify the local installation](#verify-the-local-installation)
- [Inspect before changing](#inspect-before-changing)
- [Initialize and author visually](#initialize-and-author-visually)
- [Ingest and update assets](#ingest-and-update-assets)
- [Import, replace, and refresh Figma selections](#import-replace-and-refresh-figma-selections)
- [Reference Figma nodes by tag](#reference-figma-nodes-by-tag)
- [Tag Figma nodes from an agent](#tag-figma-nodes-from-an-agent)
- [Declare cross-package dependencies](#declare-cross-package-dependencies)
- [Organize assets and manifests](#organize-assets-and-manifests)
- [Build and verify](#build-and-verify)
- [Handle command results](#handle-command-results)

## Verify the local installation

```bash
node --version
npm ls fundus --depth=0
npm exec --no -- fundus --version
```

Fundus requires Node.js 20.19 or newer and Svelte 5. It must be a runtime dependency because generated modules and host components import from `fundus`. If the user explicitly requested setup and Fundus is absent, install it first with the host package manager, for example `npm install fundus`. Inspect `fundus.config.ts` before executing the CLI because loading the config runs TypeScript from the host project.

## Inspect before changing

```bash
npm exec --no -- fundus state --json
npm exec --no -- fundus asset list --json
npm exec --no -- fundus asset show <id> --json
npm exec --no -- fundus manifest list --json
npm exec --no -- fundus folder list --json
```

`state --json` is the preferred first read. It includes assets, parameters, explicit and passive manifest memberships, proxy state, references, warnings, validation issues, and available preset names. For a large library, write the response to a temporary file outside the repository and use `jq` to select only task-relevant records instead of loading the entire document into context.

## Initialize and author visually

```bash
npm exec --no -- fundus init --json
npm exec --no -- fundus start
npm exec --no -- fundus start --port 8080 --no-open
```

Only initialize when no Fundus config exists. `start` is interactive and does not support `--json`.

## Ingest and update assets

```bash
npm exec --no -- fundus asset ingest ./exports/panel@2x.png \
  --type slice --manifests main --json

npm exec --no -- fundus asset replace navigationPanel ./exports/panel-v2.png --json

npm exec --no -- fundus asset set navigationPanel \
  --parameters '{"mode":"three-horizontal"}' \
  --manifests main,settings --json
```

Pass `--type` when a file extension is ambiguous: PNG can be `image` or `slice`, and MP4 can be `video` or `chroma-key-video`. Do not infer the intended kind from the user's wording; confirm it or inspect focused help. `--parameters` is a shallow top-level merge. `--manifests` replaces the complete explicit membership set; an empty value clears it. Ingest reference targets before creating parameters that point to them.

Prefer `replace` over delete-and-ingest when the asset should retain its ID, type, parameters, references, folder, and manifest memberships. A local-file replacement disconnects a repeatable source. Use `reimport` to refresh or edit an existing saved source, and use Figma replacement to establish a new repeatable source on an existing Image or Slice asset.

## Import, replace, and refresh Figma selections

Read the focused source-import help before using the commands:

```bash
npm exec --no -- fundus asset ingest --help
npm exec --no -- fundus asset replace --help
npm exec --no -- fundus asset reimport --help
```

Confirm that the installed `asset replace --help` documents `--from <file|figma>` before using the replacement workflow. If it does not, the host Fundus version predates Figma replacement; do not guess unsupported flags.

Configure the Figma token only when the user requested Figma integration. Keep it in a server-side environment variable; never commit, print, persist, or pass it as a CLI argument:

```ts
export default defineConfig({
	// Existing paths and plugin config...
	import: {
		figma: {
			token: process.env.FIGMA_ACCESS_TOKEN,
			defaults: { scale: 3 },
			// Default file for tag imports that omit an explicit file key.
			defaultFileKey: process.env.FIGMA_FILE_KEY
		}
	}
});
```

The token needs the `file_content:read` scope. Import one stable Figma selection as an Image or Slice asset:

```bash
npm exec --no -- fundus asset ingest \
  'https://www.figma.com/design/abc/UI?node-id=12-34' \
  --from figma --type slice --id navigationPanel --scale 3 \
  --manifests main --json
```

`--from figma` is explicit; never infer it merely because the positional value looks like a URL. A node-selection import requires `--type image|slice` and `--id`; a tag import (below) supplies the id through `--figma-tag` instead. Omit `--scale` to use `import.figma.defaults.scale`. Folder, manifest, and parameter flags have the same semantics as file ingest.

Replace an existing local or sourced Image/Slice asset from a new Figma selection:

```bash
npm exec --no -- fundus asset replace navigationPanel \
  'https://www.figma.com/design/def/UI?node-id=56-78' \
  --from figma --scale 3 --json
```

Figma replacement takes its ID and type from the existing asset, preserves references, folder, manifest memberships, and authored parameters other than a Slice's `pixelRatio`, and stores the selection as its new repeatable source. It aborts rather than overwriting local raw-file drift or a concurrent asset/source change. For a Slice, the explicit or default Figma export scale synchronizes `pixelRatio`; inspect the returned asset before making further parameter changes.

Inspect `asset.importSource` in `asset show --json` before choosing an update operation, and distinguish a **node-link** source from a **tag** source: they accept different reimport overrides. A Figma source persists only its identifiers — a file key plus either a node id or a tag, along with format and scale — and never the token or temporary download URL. The reimport overrides below apply to a node-link asset; see [Reference Figma nodes by tag](#reference-figma-nodes-by-tag) for tag assets.

```bash
# Refresh the exact saved source (re-resolve the node or tag).
npm exec --no -- fundus asset reimport navigationPanel --json

# Node-link asset: change either source field independently; the omitted value is preserved.
npm exec --no -- fundus asset reimport navigationPanel --scale 4 --json
npm exec --no -- fundus asset reimport navigationPanel \
  --figma-link 'https://www.figma.com/design/def/UI?node-id=56-78' --json
```

Choose the mutation from the intended source transition:

| Current source | Intended source | Command |
| --- | --- | --- |
| Saved Figma node-link source | Refresh it or change its link/scale | `asset reimport [--figma-link … \| --scale …]` |
| Saved Figma tag source | Refresh, re-point, rescale, or rename via the tag | `asset reimport [--figma-tag … \| --figma-file … \| --scale …]` |
| Any Image/Slice source | New Figma node selection | `asset replace <id> <link> --from figma` |
| Any Image/Slice source | New Figma tag selection | `asset replace <id> --from figma --figma-tag <tag>` |
| Any source | New local file | `asset replace <id> <file>` |

A refresh without overrides preserves the asset ID, type, authored parameters, folder, and manifest memberships. A reimport with `--figma-link` or `--scale` also synchronizes a Slice's `pixelRatio` to the resulting source scale. Reimport aborts rather than overwriting drift or concurrent source changes. A local-file replacement makes local bytes authoritative and removes future source refreshes; a Figma replacement installs new provenance and remains refreshable.

## Reference Figma nodes by tag

A node id is brittle: restructuring a frame or importing from a copied document can change it. A **tag** — a stable Fundus asset id stored on the node as shared plugin data (`fundus/assetId`) — resolves to the current node through the REST API at import time, so the reference survives node moves and file copies. Designers assign tags with the Fundus Figma plugin (imported from `figma-plugin/manifest.json` in the Figma desktop app, or from the bundle attached to the Fundus GitHub release); it is never published to npm. Agents assign them through the Figma MCP instead — see [Tag Figma nodes from an agent](#tag-figma-nodes-from-an-agent).

In tag mode **the tag is the asset id** — do not pass `--id` or a positional node-selection link. A tag import still needs a file: pass `--figma-file` (a file key or a link) or set `import.figma.defaultFileKey` for imports that omit it.

```bash
# Import by tag; the id is the tag. Uses defaultFileKey when --figma-file is omitted.
npm exec --no -- fundus asset ingest --from figma --figma-tag navigationPanel \
  --type slice --figma-file 'https://www.figma.com/design/abc/UI' --scale 3 \
  --manifests main --json

# Refresh: re-resolve the tag to its current node.
npm exec --no -- fundus asset reimport navigationPanel --json

# Re-point in place to another document and/or scale (the tag, hence the id, stays).
npm exec --no -- fundus asset reimport navigationPanel --figma-file def456 --scale 4 --json

# Change the tag — this renames the asset to the new tag.
npm exec --no -- fundus asset reimport navigationPanel --figma-tag mainNavigationPanel --json

# Replace an existing asset's bytes from a Figma tag.
npm exec --no -- fundus asset replace navigationPanel --from figma \
  --figma-tag navigationPanel --json
```

A node-link asset takes `--figma-link` on reimport and rejects `--figma-tag`/`--figma-file`; a tag asset takes `--figma-tag`/`--figma-file` and rejects `--figma-link`. Changing a tag renames the asset, so update every host usage of the old id in the same task as with any rename, and reject a tag already used by another asset. The editor import dialog offers the same choice through a **Link / Tag** switch, and a tag asset's Source section edits the tag, file key, and scale in place.

## Tag Figma nodes from an agent

When an agent creates or places a node in Figma that should become a Fundus asset, it tags the node itself instead of asking a designer to run the plugin. The Figma REST API cannot write plugin data, so Fundus never writes the tag: `fundus figma prepare-set-tag` validates the request and returns a script, and the agent runs that script with the Figma MCP `use_figma` tool.

Prerequisites: the Figma MCP server with `use_figma` write access to the file, and the Fundus Figma token from `fundus.config.ts` (read access is enough for Fundus). Confirm that `npm exec --no -- fundus figma --help` lists `prepare-set-tag`; if the command is unknown, the host Fundus version predates agent tagging — do not emulate it with hand-written plugin code.

```bash
npm exec --no -- fundus figma prepare-set-tag \
  --file 'https://www.figma.com/design/abc/UI' --node-id 12-34 \
  --tag navigationPanel --json
```

1. Choose the tag like any asset id: a stable camelCase name for what the asset is. `--file` takes a file key or link and defaults to `import.figma.defaultFileKey`. `--node-id` takes `12:34` or the `12-34` form from a `node-id` link parameter.
2. On `{"ok": false, "error": …, "existingTags": […]}` the tag is taken in the file, or a Fundus asset with that id already exists from another source. Pick another name; use `existingTags` (tag, node id, node name, page) to stay consistent with the file's naming.
3. On `{"ok": true, "code": …}`, relay every entry in `warnings`, then call `use_figma` with the returned `fileKey` and `code`. Pass `code` **unchanged** — never edit, shorten, or reassemble it. It re-checks every page of the live document before writing, which takes a few seconds on large files.
4. The script returns `{"ok": true, "nodeId", "nodeName", "tag", "previousTag", "annotated"}`. If it throws instead, it wrote nothing: the document changed since the prepare step. Rerun `prepare-set-tag` rather than retrying the old script.
5. Import the tagged node: `npm exec --no -- fundus asset ingest --from figma --figma-tag navigationPanel --type slice --json`.

Rules to preserve:

- A node that already carries a tag — the same one or another — is refused unless you pass `--force`. Use `--force` only when the user wants that node retagged. When the old tag feeds a Fundus asset, the error and warning name the follow-up `asset reimport <old> --figma-tag <new>`, which renames the asset; update host usages as with any rename.
- Tag the instance you placed, or the node inside the main component — never a layer inside an instance (ids starting with `I`, such as `I12:34;56:78`). Those layers mirror the main component's node, including its tag, and Fundus refuses them.
- A node created moments ago may not be in the REST snapshot yet; the plan then carries a warning and the script verifies the node exists.
- Each `prepare-set-tag` call plans one node. Tag several new assets with one prepare-and-run pair per node.

Remove a tag only when the user asked for it:

```bash
npm exec --no -- fundus figma prepare-delete-tag --tag navigationPanel --json
```

Identify the node by `--tag` or `--node-id` (both must match when given), then run the returned `code` with `use_figma` the same way. The script removes the tag and its Dev Mode annotation and keeps every other annotation. It is refused without `--force` when a Fundus asset is imported from the tag, because that asset's reimport would fail afterwards.

## Declare cross-package dependencies

A package can require the assets a consuming host library must provide — by `id` and `type` only — and derive a typed injection contract. Bytes, parameters, presets, and manifests always stay in the host; a dependency carries nothing but identity and kind.

Declare the dependencies in a `.ts`/`.js` module that default-exports `defineDependencies([...])`, and derive the contract for dependency injection:

```ts
import { defineDependencies, type ManifestContract } from 'fundus/config';

const dependencies = defineDependencies([{ id: 'coin', type: 'slice' }]);

// The contract is the exact, readonly entry map the host must supply.
export type CoinContract = ManifestContract<typeof dependencies>;

export default dependencies;
```

Point the host `fundus.config.ts` at dependency sources — a bare npm package name or a path to the module:

```ts
export default defineConfig({
	// paths and plugins…
	dependencies: [
		'@org/other-package', // resolved via that package's package.json "fundus".dependencies
		'./src/fundus.deps.ts' // or a direct path to a .ts/.js module
	]
});
```

An npm dependency exposes its module through `package.json`:

```json
{ "name": "@org/other-package", "fundus": { "dependencies": "./src/fundus.deps.ts" } }
```

Rules to preserve:

- Each source is either a bare npm package name or a path (a `.`/`/` prefix or a `.ts`/`.js` extension). An npm subpath such as `@org/pkg/deps.js` is not supported, and JSON is not supported because it cannot carry the literal types the contract needs.
- Dependencies are flat; a referenced package's own dependencies are not resolved transitively.
- Dependencies are a lower bound. `check`, `build`, and the editor fail hard when a declared id is missing from the host library or has the wrong type; extra, undeclared host assets are fine. The same id declared with conflicting types across sources fails at config load, naming both sources.
- An unmet dependency is a library-global issue (no asset id) surfaced in `check`, `build`, and the editor banner, but it never freezes the editor's autosave processing and codegen.

When adding or renaming a declared dependency id, satisfy it in the host library with the matching id and type in the same task, then run `npm exec --no -- fundus check --json`.

## Organize assets and manifests

```bash
npm exec --no -- fundus folder create "UI/Settings" --json
npm exec --no -- fundus asset move settingsPanel "UI/Settings" --json
npm exec --no -- fundus manifest create settings --json
npm exec --no -- fundus manifest palette --json
npm exec --no -- fundus manifest set settings --color blue --emoji '⚙️' --json
```

Asset folders organize persisted raw and proxy files. Manifests control delivery and preloading. Do not substitute one concept for the other.

Before renaming or deleting an asset, search host source for its typed manifest-property usages. Before renaming or deleting a manifest, search for its generated module path and exported constant. Apply the CLI mutation and update every host usage in the same task, then run the host typecheck. Fundus validates asset-to-asset references, but it cannot discover imports or property accesses in host source.

## Build and verify

```bash
npm exec --no -- fundus build --json
npm exec --no -- fundus check --json
```

`build` processes stale or missing proxies, prunes orphans, and regenerates manifest modules. `check` is read-only and fails when on-disk derived state is stale or invalid. Warnings fail the check unless the user deliberately accepts `--allow-warnings`.

## Handle command results

- Exit `0`: success.
- Exit `1`: expected usage, validation, or domain error. With `--json`, parse the JSON error or check report from stdout.
- Exit `2`: unexpected crash. Preserve stderr and report it as a probable Fundus defect.

Do not scrape human output when `--json` is available. Do not guess flags or parameter schemas; consult the focused help and the current state or preset names.
