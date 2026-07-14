# Fundus agent skill

An agent skill for managing production assets with Fundus in Svelte mobile app projects.

## Install

Install the `fundus` skill in the current project:

```bash
npx skills add grischaerbe/fundus-skill --skill fundus
```

Install it globally for every supported agent:

```bash
npx skills add grischaerbe/fundus-skill --skill fundus --agent '*' -g -y
```

The skill requires Fundus to be installed in the Svelte host project. It teaches agents to use the installed Fundus CLI as the authoritative interface, preserve generated artifacts, design mobile delivery manifests, and validate asset changes.

## Use without installing

```bash
npx skills use grischaerbe/fundus-skill@fundus
```
