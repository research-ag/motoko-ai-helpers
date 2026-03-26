# Motoko AI Helpers — Agent Skills

This repository provides reusable Agent Skills for Motoko development. These skills are compatible with the `skills` CLI, so you can list and install them into your coding agents (Cursor, Claude Code, Junie, Windsurf, and many more).

Learn more about the ecosystem and CLI: https://skills.sh

## Quick Start

List skills available in this repository (no install yet):

```bash
npx skills add https://github.com/research-ag/motoko-ai-helpers --list
```

Install specific skills:

```bash
# Install one skill to your default agent targets
npx skills add https://github.com/research-ag/motoko-ai-helpers \
  --skill motoko-dot-notation-migration

# Install multiple specific skills
npx skills add https://github.com/research-ag/motoko-ai-helpers \
  --skill motoko-base-to-core-migration \
  --skill motoko-core-code-improvements

# Install to specific agents (examples)
npx skills add https://github.com/research-ag/motoko-ai-helpers \
  --skill motoko-general-style-guidelines \
  -a cursor -a claude-code

# Install all skills from this repository
npx skills add https://github.com/research-ag/motoko-ai-helpers --skill '*'
```

Project vs Global installation:

```bash
# Project (default): installs into your project-specific agent path(s)
npx skills add <source>

# Global: makes skills available to all projects on this machine
npx skills add <source> -g
```

After installing, list installed skills:

```bash
npx skills list
# or
npx skills ls
```

## Included Skills

Each skill is a directory under `skills/**/SKILL.md` with frontmatter that the CLI reads.

- motoko-base-to-core-migration — Complete, AI-ready playbook to migrate Motoko projects from mo:base to mo:core.
- motoko-dot-notation-migration — Convert old `Module.func(self, ...)` style calls to `self.func(...)` dot notation for mo:core.
- motoko-core-code-improvements — Optional cleanups for mo:core projects (import order, unused imports, return removal, etc.).
- motoko-general-style-guidelines — Comprehensive Motoko style guidelines to improve readability and consistency.

## Authoring New Skills

Use the template to create a new skill:

```bash
# Create SKILL.md in current directory
npx skills init

# Or create a named subdirectory and SKILL.md
npx skills init my-new-skill
```

Or copy our template and fill it out:

- Template: `skills/_template/SKILL.md.template`

Frontmatter requirements for compatibility with the `skills` CLI:

- `name` (required): lowercase, kebab-case, unique id, e.g. `motoko-dot-notation-migration`
- `description` (required): concise one-liner describing what the skill does and when to use it
- `metadata.internal` (optional): set `true` to hide the skill from normal discovery (WIP/internal)

## Conventions

- Skill directories live under `skills/` and contain a single `SKILL.md`
- Keep `name` stable once published; changes will affect how users refer to the skill
- Prefer concise, actionable instructions that agents can follow reliably

## Troubleshooting

- If the CLI can’t find skills, ensure your source path or Git URL points to this repository’s root (the scanner looks for `**/SKILL.md`).
- If a skill isn’t selectable by name, verify its `name` is lowercase, kebab-case, and unique across the repo.

## License

See the repository’s LICENSE file
