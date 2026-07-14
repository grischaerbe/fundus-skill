---
name: fundus
description: Use Fundus to import, organize, process, validate, preload, and render production assets in Svelte 5 mobile app projects. Use when working with Fundus setup, fundus.config.ts, asset libraries, images, slices, video, chroma-key video, audio, manifests, generated modules, the Fundus CLI or editor, runtime components, asset delivery budgets, or preload lifecycles.
---

# Fundus

Manage a Svelte mobile app's production assets through Fundus while preserving its deterministic, typed asset pipeline.

## Start from project state

Run Fundus commands from the host project root, next to `fundus.config.ts`.

1. Check whether `fundus.config.ts` exists.
2. If it exists, inspect the complete project state with `npx fundus state --json`.
3. If it does not exist and the user wants Fundus initialized, run `npx fundus init --json` once.
4. Read focused command help before using unfamiliar or version-sensitive flags, for example `npx fundus asset ingest --help`.

Use JSON output for agent-driven work. Treat the installed CLI's help and JSON responses as the authoritative contract.

## Preserve the asset pipeline

- Treat raw originals, the Fundus library, and `fundus.config.ts` as inputs.
- Never hand-edit processed proxies or generated manifest modules. Change their inputs and regenerate them.
- Prefer stable asset IDs. Use `asset replace` when the source file changes but the asset's identity and metadata should remain intact.
- Use Fundus CLI mutations instead of editing the library JSON directly. Mutations validate, persist, process affected assets, and regenerate manifests as one operation.
- Inspect references before renaming or deleting assets. Do not perform destructive asset, folder, or manifest deletion unless the user requested it.
- Run `npx fundus check --json` after changes. Run `npx fundus build --json` first when files, configuration, or generated output may be stale.
- Report warnings, validation issues, and any host imports that require manual updates.

## Design for mobile delivery

- Treat manifests as delivery units and preload groups, not merely folders.
- Group assets by the screen, route, overlay, or interaction flow that needs them. Avoid one catch-all manifest when it inflates startup cost.
- Inspect manifest membership and `deliveredBytes` with `npx fundus manifest list --json`.
- Account for passive members pulled into a manifest through asset references.
- Preload a manifest shortly before its screen or flow becomes interactive. Release it when its assets are no longer needed and no other active flow depends on it.
- Prefer `Slice` for stretchable mobile UI surfaces instead of shipping multiple fixed-size variants.
- Keep original source quality in raw assets and tune delivered proxies through Fundus parameters and presets.

Read [references/mobile-workflows.md](references/mobile-workflows.md) when planning manifest boundaries, runtime loading, rendering, or mobile asset budgets.

## Operate through the CLI

Use the visual editor for authoring that benefits from direct preview, such as slice insets, chroma-key tuning, or browsing a large library. Use the CLI for repeatable agent work, batch inspection, CI, and precise mutations.

Read [references/cli.md](references/cli.md) before ingesting, mutating, organizing, or validating assets.

## Finish the task

1. Run `npx fundus check --json`.
2. Run the host project's relevant typecheck, tests, or build when generated imports or runtime rendering changed.
3. Review the diff for raw assets, the library, proxies, and generated modules.
4. Summarize changed asset IDs, manifest membership, delivered-size impact, warnings, and checks run.
