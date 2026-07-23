# Fundus CLI workflow

Use the host package manager's local-only runner and run commands from the host project root. The examples below use npm's `npm exec --no -- fundus`; adapt the runner to the detected lockfile without allowing an implicit registry download. Use `<runner> <command> --help` as the source of truth for the current Fundus CLI.

## Contents

- [Verify the local installation](#verify-the-local-installation)
- [Inspect before changing](#inspect-before-changing)
- [Initialize and author visually](#initialize-and-author-visually)
- [Ingest and update assets](#ingest-and-update-assets)
- [Import, replace, and refresh Figma selections](#import-replace-and-refresh-figma-selections)
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
			defaults: { scale: 3 }
		}
	}
});
```

Import one stable Figma selection as an Image or Slice asset:

```bash
npm exec --no -- fundus asset ingest \
  'https://www.figma.com/design/abc/UI?node-id=12-34' \
  --from figma --type slice --id navigationPanel --scale 3 \
  --manifests main --json
```

`--from figma` is explicit; never infer it merely because the positional value looks like a URL. Figma import requires `--type image|slice` and `--id`. Omit `--scale` to use `import.figma.defaults.scale`. Folder, manifest, and parameter flags have the same semantics as file ingest.

Replace an existing local or sourced Image/Slice asset from a new Figma selection:

```bash
npm exec --no -- fundus asset replace navigationPanel \
  'https://www.figma.com/design/def/UI?node-id=56-78' \
  --from figma --scale 3 --json
```

Figma replacement takes its ID and type from the existing asset, preserves references, folder, manifest memberships, and authored parameters other than a Slice's `pixelRatio`, and stores the selection as its new repeatable source. It aborts rather than overwriting local raw-file drift or a concurrent asset/source change. For a Slice, the explicit or default Figma export scale synchronizes `pixelRatio`; inspect the returned asset before making further parameter changes.

Inspect `asset.importSource` in `asset show --json` before choosing an update operation. A Figma source persists only the file key, node ID, format, and scale; it never contains the token or temporary download URL.

```bash
# Refresh the exact saved source.
npm exec --no -- fundus asset reimport navigationPanel --json

# Change either source field independently; the omitted value is preserved.
npm exec --no -- fundus asset reimport navigationPanel --scale 4 --json
npm exec --no -- fundus asset reimport navigationPanel \
  --figma-link 'https://www.figma.com/design/def/UI?node-id=56-78' --json
```

Choose the mutation from the intended source transition:

| Current source | Intended source | Command |
| --- | --- | --- |
| Saved Figma source | Refresh it or change its link/scale | `asset reimport` |
| Any Image/Slice source | New Figma selection | `asset replace <id> <link> --from figma` |
| Any source | New local file | `asset replace <id> <file>` |

A refresh without overrides preserves the asset ID, type, authored parameters, folder, and manifest memberships. A reimport with `--figma-link` or `--scale` also synchronizes a Slice's `pixelRatio` to the resulting source scale. Reimport aborts rather than overwriting drift or concurrent source changes. A local-file replacement makes local bytes authoritative and removes future source refreshes; a Figma replacement installs new provenance and remains refreshable.

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
