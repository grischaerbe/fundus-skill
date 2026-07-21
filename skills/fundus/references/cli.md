# Fundus CLI workflow

Use the host package manager's local-only runner and run commands from the host project root. The examples below use npm's `npm exec --no -- fundus`; adapt the runner to the detected lockfile without allowing an implicit registry download. Use `<runner> <command> --help` as the source of truth for the current Fundus CLI.

## Contents

- [Verify the local installation](#verify-the-local-installation)
- [Inspect before changing](#inspect-before-changing)
- [Analyze typed asset usage](#analyze-typed-asset-usage)
- [Prune unused assets safely](#prune-unused-assets-safely)
- [Initialize and author visually](#initialize-and-author-visually)
- [Ingest and update assets](#ingest-and-update-assets)
- [Import and refresh Figma selections](#import-and-refresh-figma-selections)
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

## Analyze typed asset usage

First confirm that the installed CLI and project configuration support usage analysis:

```bash
npm exec --no -- fundus usage --help
npm exec --no -- fundus usage --json
npm exec --no -- fundus usage --unused --json
npm exec --no -- fundus usage navigationPanel --json
```

The host project must configure both `usage.referenceProvider` and `usage.openLocation` in `fundus.config.ts`. The provider should reuse the project's existing TypeScript, language-server, or framework integration. Fundus does not bring its own TypeScript or framework language server. If the callbacks are absent, usage analysis exits with an expected error; `start` and `check` also print a non-fatal advisory.

Every usage record has one of three statuses:

- `used`: at least one typed source reference is attributed to the asset through a manifest root or dependency path.
- `unused`: every relevant declaration was resolved conclusively and no source reference was found.
- `unknown`: dynamic access, an incomplete language-service response, or a provider failure prevents a conclusion.

Only `unused` proves absence strongly enough for pruning. Never collapse `unknown` into `unused`, even when a separate text search returns no matches.

The all-assets response has this envelope:

```json
{
	"ok": true,
	"assets": [
		{ "assetId": "navigationPanel", "status": "used", "references": [], "roots": [] },
		{ "assetId": "obsoleteBanner", "status": "unused", "references": [], "roots": [] }
	]
}
```

Use `usage <id> --json` when concrete attribution matters. It returns absolute, one-based source locations plus the manifest roots and asset-reference paths that lead to them:

```json
{
	"ok": true,
	"assets": [
		{
			"assetId": "navigationPanel",
			"status": "used",
			"references": [
				{
					"id": "r0",
					"file": "/project/src/routes/settings/+page.svelte",
					"line": 24,
					"column": 15,
					"label": "settings.navigationPanel"
				}
			],
			"roots": [
				{
					"manifest": "settings",
					"rootAssetId": "navigationPanel",
					"paths": [["navigationPanel"]],
					"result": { "status": "resolved", "referenceIds": ["r0"] }
				}
			]
		}
	]
}
```

CLI results intentionally omit editor `openToken` values because a one-shot process cannot keep them valid. Use the Fundus editor to navigate to returned references. Use `--unused` only without an asset ID; it filters the response to conclusively unused records.

## Prune unused assets safely

Asset pruning is destructive and must be explicitly requested by the user. `fundus build` only removes orphaned derived proxy files. It never deletes an asset record or raw original.

1. Ensure derived state is current. Run `build --json` when inputs or generated output may be stale, then `check --json`.
2. Capture a fresh candidate set with `usage --unused --json`. Keep the JSON outside the repository when it is large.
3. For each candidate, run both `usage <id> --json` and `asset show <id> --json`. Report the exact asset ID, raw file, manifest roots, and dependency paths that deletion affects.
4. Delete direct unused referrers and manifest roots before their referenced dependencies. `asset delete` refuses to remove a target while another asset references it.
5. Delete one asset with `asset delete <id> --json`, then immediately run `usage --unused --json` again. Continue only if the next candidate still reports `unused`.
6. Stop rather than delete when analysis becomes `unknown`, the library changed concurrently, deletion fails, or validation reports a new issue.
7. Finish with `build --json`, `check --json`, the host typecheck, and `git status --short --untracked-files=all`. Review deleted raw files, proxies, library records, and regenerated manifest modules.

There is intentionally no assumption that every item in the first `--unused` response remains deletable. Attribution is a graph, and deleting one root can change the status or deletion order of another asset.

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

Prefer `replace` over delete-and-ingest for an authoritative local file when the asset should retain its ID, type, parameters, references, folder, and manifest memberships. Replacing raw bytes disconnects a repeatable source; use `reimport` instead when an asset has one.

## Import and refresh Figma selections

Read the focused source-import help before using the commands:

```bash
npm exec --no -- fundus asset ingest --help
npm exec --no -- fundus asset reimport --help
```

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

`--from figma` is explicit; never infer it merely because the positional value looks like a URL. Figma import requires `--type image|slice` and `--id`. Omit `--scale` to use `import.figma.defaults.scale`. The export scale controls downloaded pixels and is independent of Slice `pixelRatio`. Folder, manifest, and parameter flags have the same semantics as file ingest.

Inspect `asset.importSource` in `asset show --json` before choosing an update operation. A Figma source persists only the file key, node ID, format, and scale; it never contains the token or temporary download URL.

```bash
# Refresh the exact saved source.
npm exec --no -- fundus asset reimport navigationPanel --json

# Change either source field independently; the omitted value is preserved.
npm exec --no -- fundus asset reimport navigationPanel --scale 4 --json
npm exec --no -- fundus asset reimport navigationPanel \
  --figma-link 'https://www.figma.com/design/def/UI?node-id=56-78' --json
```

Reimport preserves the asset ID, type, parameters, folder, and manifest memberships. It aborts rather than overwriting local raw-file drift or a concurrent source change. If the user intentionally wants local raw bytes to become authoritative, use `asset replace` and make clear that future source refreshes will no longer be available.

## Organize assets and manifests

```bash
npm exec --no -- fundus folder create "UI/Settings" --json
npm exec --no -- fundus asset move settingsPanel "UI/Settings" --json
npm exec --no -- fundus manifest create settings --json
npm exec --no -- fundus manifest palette --json
npm exec --no -- fundus manifest set settings --color blue --emoji '⚙️' --json
```

Asset folders organize persisted raw and proxy files. Manifests control delivery and preloading. Do not substitute one concept for the other.

Before renaming or deleting an asset, use `usage <id> --json` when the project configures usage analysis, then search for non-typed or dynamic access the provider may classify as `unknown`. Before renaming or deleting a manifest, search for its generated module path and exported constant. Apply the CLI mutation and update every host usage in the same task, then run the host typecheck. Fundus validates asset-to-asset references; host-source discovery depends on the configured usage provider.

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
