---
name: fundus
description: Use Fundus to import, replace, reimport, organize, process, validate, preload, and render production assets in Svelte 5 mobile app projects that already declare Fundus, contain fundus.config.ts, or are explicitly adopting Fundus. Use for Fundus setup, asset libraries, local files, Figma selection sources, Figma tag references, tagging Figma nodes through the Figma MCP, images, slices, video, chroma-key video, audio, manifests, generated modules, cross-package asset dependencies and dependency-injection contracts, the Fundus CLI or editor, runtime components, retained entries, canvas and WebGL drawing, delivery budgets, and preload lifecycles.
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
- Prefer stable asset IDs. Refresh an existing repeatable source with `asset reimport`. Use `asset replace` to keep identity and metadata while establishing a new authoritative source: a local file disconnects provenance, while `--from figma` stores a new repeatable Figma source.
- Prefer a Figma **tag** over a raw node id for repeatable Figma sources. A tag is a stable Fundus asset id stored on the node as shared plugin data (`fundus/assetId`); it survives node moves and file copies. In tag mode the tag is the asset id, so renaming means changing the tag, not the id.
- Tag Figma nodes yourself when you create or place them: designers use the Fundus Figma plugin, agents use `fundus figma prepare-set-tag` and run the returned script unchanged with the Figma MCP `use_figma` tool. Never write `fundus` plugin data with hand-written `use_figma` code. Read [Tag Figma nodes from an agent](references/cli.md#tag-figma-nodes-from-an-agent).
- Use Fundus CLI mutations instead of editing the library JSON directly. Mutations validate, persist, process affected assets, and regenerate manifests as one operation.
- Inspect Fundus references and search host source before renaming or deleting an asset or manifest. Update generated-module imports, manifest export names, and typed asset-property usages in the same task.
- Do not perform destructive asset, folder, or manifest deletion unless the user requested it.
- Run `npm exec --no -- fundus check --json` after changes. Run `npm exec --no -- fundus build --json` first when files, configuration, or generated output may be stale.
- Report warnings and validation issues. Do not leave known host import or type errors for the user to repair manually.

## Design for mobile delivery

- Treat manifests as delivery units and preload groups, not merely folders.
- Group assets by the screen, route, overlay, or interaction flow that needs them. Avoid one catch-all manifest when it inflates startup cost.
- Record manifest membership and `deliveredBytes` before and after delivery changes with `npm exec --no -- fundus manifest list --json`; report the byte delta.
- Account for passive members pulled into a manifest through asset references.
- Give each preloaded manifest one lifecycle owner. Multiple `preload()` calls on the same generated manifest do not create independent holds, so never let repeated component instances each call `release()` independently. Per-instance consumers hold entries individually instead: Svelte components construct `new RetainedEntry(entry)` during initialisation, which holds until destroy; other code uses `retainEntry(entry)`, where every handle is its own hold with its own `release()`.
- Preload shortly before a screen or flow becomes interactive, handle rejection explicitly, and release only after the final consumer is gone.
- Prefer `Slice` for stretchable mobile UI surfaces instead of shipping multiple fixed-size variants.
- For an existing canvas, pass a `RetainedEntry` (inside a `$effect`, once its `source` is set) or an `await retainEntry(entry)` handle to synchronous `drawSlice()` or `drawImage()`; release a `retainEntry` handle after the final draw. For WebGL or Pixi, combine the handle's `source` with `sliceGrid()`. Read [Retain entries](references/mobile-workflows.md#retain-individual-entries), [Canvas rendering](references/mobile-workflows.md#draw-into-an-existing-canvas), and [Custom renderers](references/mobile-workflows.md#render-with-webgl-or-pixi).
- Keep original source quality in raw assets and tune delivered proxies through Fundus parameters and presets.

Read [references/mobile-workflows.md](references/mobile-workflows.md) when planning manifest boundaries, runtime loading, rendering, or mobile asset budgets.

## Share assets across packages with dependencies

- A package declares the assets a consuming host must provide with `defineDependencies([{ id, type }, …])` from `fundus/config`, and derives a typed injection contract with `ManifestContract<typeof deps>`. A dependency carries identity and kind only — bytes, parameters, presets, and manifests always live in the host library.
- The host lists dependency sources in `fundus.config.ts` under `dependencies`: a bare npm package name (resolved through that package's `package.json` `"fundus".dependencies` module path) or a path to a `.ts`/`.js` module that default-exports `defineDependencies([...])`. JSON is unsupported because it cannot carry the literal types the contract needs.
- Dependencies are flat (not resolved transitively) and are a lower bound: extra host assets are fine, but a declared id that is missing or has the wrong type is a hard, library-global failure in `check`, `build`, and the editor banner. Conflicting types for the same id across sources fail at config load, naming both sources.
- Treat dependency modules as inputs. When adding or renaming a declared id, satisfy it in the host library (correct id and type) in the same task, then run `npm exec --no -- fundus check --json`. An unmet dependency is surfaced but never freezes the editor's autosave processing and codegen.
- Read [references/cli.md](references/cli.md) for the concrete declaration and config wiring.

## Operate through the CLI

Use the visual editor for authoring that benefits from direct preview, such as slice insets, chroma-key tuning, or browsing a large library. Use the CLI for repeatable agent work, batch inspection, CI, and precise mutations.

Read [references/cli.md](references/cli.md) before ingesting, mutating, organizing, or validating assets.

## Finish the task

1. Run `npm exec --no -- fundus check --json`.
2. Run the host project's relevant typecheck, tests, or build when generated imports or runtime rendering changed.
3. Run `git status --short --untracked-files=all` before reviewing the diff so new raw assets and generated files are included. Review the library, proxies, and generated modules as well as tracked changes.
4. Summarize changed asset IDs, manifest membership, delivered-size impact, warnings, and checks run.
