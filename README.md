# Fundus agent skill

An agent skill for managing production assets with Fundus in Svelte mobile app projects,
including typed usage analysis and safe pruning of conclusively unused assets.

## Install

Install the `fundus` skill in the current project:

```bash
npx skills add grischaerbe/fundus-skill --skill fundus
```

Install it globally for every supported agent:

```bash
npx skills add grischaerbe/fundus-skill --skill fundus --agent '*' -g -y
```

The skill requires Node.js 20.19 or newer, Svelte 5, and Fundus as a runtime dependency in the host project. It teaches agents to use only the locally installed Fundus CLI as the authoritative interface, preserve generated artifacts, work safely with local and repeatable sources, inspect typed source references, prune only conclusively unused assets, design mobile delivery manifests, and validate asset changes. Its released command contract is continuously checked in CI.

## Documentation

- [Skill instructions](skills/fundus/SKILL.md)
- [CLI workflow](skills/fundus/references/cli.md)
- [Svelte mobile asset workflows](skills/fundus/references/mobile-workflows.md)

## Use without installing

```bash
npx skills use grischaerbe/fundus-skill@fundus
```
