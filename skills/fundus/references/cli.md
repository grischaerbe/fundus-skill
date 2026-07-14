# Fundus CLI workflow

Use `npx fundus <command> --help` as the source of truth for the installed version. Run commands from the host project root.

## Inspect before changing

```bash
npx fundus state --json
npx fundus asset list --json
npx fundus asset show <id> --json
npx fundus manifest list --json
npx fundus folder list --json
```

`state --json` is the preferred first read. It includes assets, parameters, explicit and passive manifest memberships, proxy state, references, warnings, validation issues, and available preset names.

## Initialize and author visually

```bash
npx fundus init --json
npx fundus start
npx fundus start --port 8080 --no-open
```

Only initialize when no Fundus config exists. `start` is interactive and does not support `--json`.

## Ingest and update assets

```bash
npx fundus asset ingest ./exports/panel@2x.png \
  --type slice --manifests main --json

npx fundus asset replace navigationPanel ./exports/panel-v2.png --json

npx fundus asset set navigationPanel \
  --parameters '{"mode":"three-horizontal"}' \
  --manifests main,settings --json
```

Pass `--type` when a file extension is ambiguous. `--parameters` is a shallow top-level merge. `--manifests` replaces the complete explicit membership set; an empty value clears it. Ingest reference targets before creating parameters that point to them.

Prefer `replace` over delete-and-ingest when the asset should retain its ID, type, parameters, references, folder, and manifest memberships.

## Organize assets and manifests

```bash
npx fundus folder create "UI/Settings" --json
npx fundus asset move settingsPanel "UI/Settings" --json
npx fundus manifest create settings --json
npx fundus manifest palette --json
npx fundus manifest set settings --color blue --emoji '⚙️' --json
```

Asset folders organize persisted raw and proxy files. Manifests control delivery and preloading. Do not substitute one concept for the other.

Renaming a manifest changes its generated module filename and exported constant. Update host imports manually after the rename.

## Build and verify

```bash
npx fundus build --json
npx fundus check --json
```

`build` processes stale or missing proxies, prunes orphans, and regenerates manifest modules. `check` is read-only and fails when committed derived state is stale or invalid. Warnings fail the check unless the user deliberately accepts `--allow-warnings`.

## Handle command results

- Exit `0`: success.
- Exit `1`: expected usage, validation, or domain error. With `--json`, parse the JSON error or check report from stdout.
- Exit `2`: unexpected crash. Preserve stderr and report it as a probable Fundus defect.

Do not scrape human output when `--json` is available. Do not guess flags or parameter schemas; consult the focused help and the current state or preset names.
