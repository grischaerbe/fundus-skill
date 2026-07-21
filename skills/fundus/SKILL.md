---
name: fundus
description: Use Fundus to import, reimport, organize, process, validate, inspect typed usage, safely prune unused assets, preload, and render production assets in Svelte 5 mobile app projects that already declare Fundus, contain fundus.config.ts, or are explicitly adopting Fundus. Use for Fundus setup, asset libraries, local files, Figma selection sources, images, slices, video, chroma-key video, audio, manifests, generated modules, usage tracking, asset pruning, the Fundus CLI or editor, runtime components, delivery budgets, and preload lifecycles.
---

# Fundus

Manage a Svelte mobile app's production assets through Fundus while preserving its deterministic, typed asset pipeline.

## Start from project state

Run Fundus commands from the host project root, next to `fundus.config.ts`.

1. Detect the host package manager from its lockfile. Confirm Node.js 20.19 or newer, Svelte 5, and a runtime dependency on `fundus` before executing the CLI.
2. If Fundus is missing and the user explicitly requested setup, install it with the host package manager, for example `npm install fundus`. Do not rely on an executor downloading Fundus temporarily.
3. Use the package manager's local-only runner for every command. With npm, use `npm exec --no -- fundus`; `--no` prevents a missing local package from falling back to a registry download.
4. Inspect `fundus.config.ts` before running it because it is executable TypeScript.
5. If the config exists, inspect project state with `npm exec --no -- fundus state --json`. For a large library, capture the JSON in a temporary file outside the repository and use `jq` to read only the assets or manifests relevant to the task.
6. If the config does not exist and setup was requested, run `npm exec --no -- fundus init --json` once after installation.
7. Read focused command help before using unfamiliar flags, for example `npm exec --no -- fundus asset ingest --help`.

Use JSON output for agent-driven work. Treat the installed CLI's help and JSON responses as the authoritative contract.

## Preserve the asset pipeline

- Treat raw originals, the Fundus library, and `fundus.config.ts` as inputs.
- Never hand-edit processed proxies or generated manifest modules. Change their inputs and regenerate them.
- Prefer stable asset IDs. Refresh a repeatable source with `asset reimport`; use `asset replace` for authoritative local files when the asset's identity and metadata should remain intact. Replacing raw bytes disconnects any repeatable source.
- Use Fundus CLI mutations instead of editing the library JSON directly. Mutations validate, persist, process affected assets, and regenerate manifests as one operation.
- Inspect Fundus references and typed host-source usage before renaming or deleting an asset or manifest. Update generated-module imports, manifest export names, and typed asset-property usages in the same task.
- Do not perform destructive asset, folder, or manifest deletion unless the user requested it.
- Run `npm exec --no -- fundus check --json` after changes. Run `npm exec --no -- fundus build --json` first when files, configuration, or generated output may be stale.
- Report warnings and validation issues. Do not leave known host import or type errors for the user to repair manually.

## Track usage and prune safely

Use `npm exec --no -- fundus usage --help` to confirm that the locally installed Fundus version exposes usage analysis. The host project owns the `usage.referenceProvider` and `usage.openLocation` integration in `fundus.config.ts`; Fundus reuses its language tooling and does not bundle TypeScript, a framework language server, or an editor SDK.

- Run `npm exec --no -- fundus usage --json` for the complete status set. Treat `used`, `unused`, and `unknown` as distinct outcomes.
- Only `unused` is a pruning candidate. It means analysis resolved conclusively without a source reference. `unknown` means analysis was incomplete or non-conclusive and must never be treated as unused.
- Run `npm exec --no -- fundus usage <id> --json` before changing one candidate. Inspect its source references, manifest roots, and dependency paths together with `asset show <id> --json`.
- If usage integration is unavailable, report that limitation. A text search may help locate known consumers, but it cannot prove an asset is unused and must not authorize pruning.
- `fundus build` prunes orphaned derived proxies; it does not remove library assets or raw originals. User-requested asset pruning uses `fundus asset delete <id> --json`, which deletes the record, raw file, and proxies.
- Delete unused referrers or manifest roots before unused assets they reference. `asset delete` rejects a referenced target. Re-run `usage --unused --json` after every deletion because removing a root can change attribution for its dependencies.
- Never bulk-delete from a stale candidate list. Stop on a new `unknown` result, a failed deletion, an introduced validation issue, or concurrent project changes.

Read [references/cli.md](references/cli.md) for the result contract and complete pruning workflow.

## Design for mobile delivery

- Treat manifests as delivery units and preload groups, not merely folders.
- Group assets by the screen, route, overlay, or interaction flow that needs them. Avoid one catch-all manifest when it inflates startup cost.
- Record manifest membership and `deliveredBytes` before and after delivery changes with `npm exec --no -- fundus manifest list --json`; report the byte delta.
- Account for passive members pulled into a manifest through asset references.
- Give each preloaded manifest one lifecycle owner. Multiple `preload()` calls on the same generated manifest do not create independent holds, so never let repeated component instances each call `release()` independently.
- Preload shortly before a screen or flow becomes interactive, handle rejection explicitly, and release only after the final consumer is gone.
- Prefer `Slice` for stretchable mobile UI surfaces instead of shipping multiple fixed-size variants.
- Keep original source quality in raw assets and tune delivered proxies through Fundus parameters and presets.

Read [references/mobile-workflows.md](references/mobile-workflows.md) when planning manifest boundaries, runtime loading, rendering, or mobile asset budgets.

## Operate through the CLI

Use the visual editor for authoring that benefits from direct preview, such as slice insets, chroma-key tuning, or browsing a large library. Use the CLI for repeatable agent work, batch inspection, CI, and precise mutations.

Read [references/cli.md](references/cli.md) before analyzing usage, pruning, ingesting, mutating, organizing, or validating assets.

## Finish the task

1. Run `npm exec --no -- fundus check --json`.
2. Run the host project's relevant typecheck, tests, or build when generated imports or runtime rendering changed.
3. Run `git status --short --untracked-files=all` before reviewing the diff so new raw assets and generated files are included. Review the library, proxies, and generated modules as well as tracked changes.
4. Summarize changed and pruned asset IDs, usage-analysis outcomes, manifest membership, delivered-size impact, warnings, and checks run.
