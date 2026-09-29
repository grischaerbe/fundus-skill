---
name: fundus
description: Operate Fundus (CLI, editor, runtime) in SvelteKit/Svelte 5 projects that declare Fundus, contain fundus.config.ts, or are adopting it. Use to set up Fundus; ingest, replace, reimport, and organize production assets from local files or Figma nodes and tags (including tagging via the Figma MCP); manage manifests, generated modules, and cross-package asset dependencies; validate; and preload or render images, slices, video, chroma-key video, and audio with components, canvas, or WebGL within mobile delivery budgets.
---

# Fundus

Fundus turns raw originals into processed proxies and typed, generated manifest modules. Keep that pipeline deterministic and let the CLI own every mutation.

## Start

1. Work in the package that contains `fundus.config.ts` (a monorepo lockfile may sit at the root). Require Node.js 20.19+, SvelteKit with Svelte 5, and `fundus` as a runtime dependency of that package, not the workspace root.
2. Run the CLI only through the local runner — npm: `npm exec --no -- fundus`, pnpm: `pnpm exec fundus`, yarn: `yarn fundus` (unless a `fundus` package script shadows it) — and confirm `fundus --version` matches `node_modules/fundus/package.json`. Never let a runner download Fundus (`npx fundus`, `pnpm dlx`); read `npx fundus` in help texts as your runner. Below, `fundus` means that runner.
3. Read `fundus.config.ts` and any dependency modules it lists before the first command: loading them executes host code.
4. Setup, only when requested: install `fundus` in that package (`npm install fundus`, `pnpm add fundus`), then follow [Set up](references/cli.md#set-up).
5. Read state with `fundus state --json`. For a large library, write it to a temp file outside the repo and `jq` only the relevant records.

The installed CLI's `--help` and JSON output, and the installed runtime types, are the contract. Read focused help before unfamiliar flags. If help lacks a command or flag described here, the installed Fundus predates it: never guess, emulate, or work around it — ask the user to update. Pass `--json` to every command except the interactive editor, `fundus start`.

## Rules

- **Inputs vs derived.** Raw originals, the library, `fundus.config.ts`, and dependency modules are inputs; proxies and generated modules are derived. Never hand-edit derived files or the library JSON — CLI mutations validate, persist, process, and regenerate in one step. Keep formatters and lint-staged off the library and `manifestsDir` (`check` needs byte-identical modules). Commit all of it. Only for a VCS merge conflict, merge the library JSON by hand, take either side of generated modules, then `build` and `check`.
- **Files enter only through the CLI.** Ingest or replace from a path outside the raw directory (ingesting a file already inside it makes a renamed copy); never add, edit, move, or delete files inside it. Edited, missing, or moved raw files block `build` and processing; unmanaged new files are silently skipped by `build`; both fail `check` unconditionally. Prefer restoring the exact original from git; the editor's ingestion queue also resolves drift, but disconnects Figma sources, and "process" resets parameters.
- **One writer.** Run mutations strictly one at a time — no parallel shells or concurrent tool calls; there is no cross-process lock — and never while the editor is changing the same project (ask the user to close it). Suggest the editor for visual authoring (slice insets, chroma-key tuning); do agent work through the CLI.
- **Read results, not just exit codes.** Exit 0 means the change was written, but non-empty `issues` means the project is unhealthy: existing validation errors or raw drift skip processing and codegen, `Processing failed: …` means processing broke, and unmet dependencies block `build` and `check`. Fix them before continuing. Exit 1 is a rejected command (nothing written): read the JSON on stdout — `error.message`, or for `check` its `errors` and `warnings` (`advisories` are non-fatal). Exit 2 is a crash: keep stderr and report a probable Fundus defect (or a host config or dependency-module bug).
- **Stable IDs.** Update assets in place with `reimport` or `replace`, never delete-and-ingest. Prefer Figma **tags** over node links. Ids and manifest names are camelCase (`^[a-z][A-Za-z0-9]*$`) and never `preload`, `release`, `entries`, `prototype`, or `Object.prototype` members (`constructor`, `toString`, …). Manifest names also cannot be JS reserved words (`new`, `class`, `default`, …) or `createManifest`, since each becomes `export const <name>`: Fundus 0.28 rejects them and flags existing ones (rename those), while older versions accept them and generate a module that breaks the host build. A manifest's module is its kebab-case name: `mainCatalog` → `main-catalog.generated.ts`.
- **Figma tags are written by a generated script.** Agents run `fundus figma prepare-set-tag` and pass its `code` unchanged to the Figma MCP `use_figma` tool. Never hand-write `use_figma` code that touches `fundus` plugin data.
- **Renames and deletions reach host code.** Delete assets, folders, manifests, or tags only when the user asked. Fundus refuses to rename, retag, or delete an asset another asset references: clear the reference with `asset set`, rename, then re-point it. It cannot see host code, so before renaming or deleting an asset or manifest, find host usages — `fundus usage <id> --json` if configured, and a search for the module path, export names, and property accesses when it isn't or reports `unknown` — then update them in the same task and run `check` and the host typecheck.
- **Validate.** When files, config, or generated output may be stale, run `fundus build --json`; then `fundus check --json` (read-only). Warnings fail it; pass `--allow-warnings` only if the user accepts them. After a Fundus upgrade, read its GitHub release notes (github.com/grischaerbe/fundus; the changelog is not in `node_modules`) for breaking changes such as server-plugin APIs, then `build`, `check`, and commit the regenerated modules.

Read [references/cli.md](references/cli.md) before ingesting, updating, tagging, organizing, or declaring dependencies.

## Mobile delivery

Manifests are delivery and preload units; folders only organize files. Capture `fundus manifest list --json` before and after changes (passive members pulled in by references count) and report each `deliveredBytes` delta.

Generated manifests are singletons: repeated `preload()` calls share one hold, so give each one lifecycle owner and never let repeated component instances each `release()`. Per-consumer holds use `new RetainedEntry(entry)` in components, `retainEntry(entry)` elsewhere.

Read [references/mobile-workflows.md](references/mobile-workflows.md) before planning manifests or preloading, or rendering with components, slices, audio, canvas, or WebGL/Pixi.

## Finish

1. `fundus check --json`, plus the host typecheck, tests, or build when generated imports or rendering changed.
2. `git status --short --untracked-files=all`, then review the diff, including new raw files, the library, proxies, and generated modules.
3. Summarize changed asset IDs, manifest membership, delivered-size deltas, warnings, and checks run.
