# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Repository Overview

A curated collection of skill packs for AI coding assistants, providing framework-specific knowledge and architecture guidance. Skills follow the Skills CLI ecosystem conventions (see [skills.sh](https://skills.sh//)).

## Project Structure

```
skills/
  <skill-name>/           # kebab-case, the distributable skill package
    SKILL.md              # English skill definition (required)
    GENERATION.md         # Provenance & generation metadata
    CHANGES.md            # Modification changelog (English)
    evals.json            # Optional: 2-3 test prompts + expectations (skill-forge Step 5)
    references/           # Optional: detailed reference documents
      ...
skills-zh/
  <skill-name>/           # Chinese versions of skill documents
    SKILL.zh.md           # Chinese skill definition, same content as skills/<skill-name>/SKILL.md
example/                  # Reference examples (gitignored), not part of this project
```

## Chinese/English Sync Rule

- **English version**: `skills/<skill-name>/SKILL.md` — the distributable, canonical skill file.
- **Chinese version**: `skills-zh/<skill-name>/SKILL.zh.md` — for Chinese-speaking users, stored separately to keep the `skills/` directory clean.
- **When modifying**: Always update both files simultaneously. Content must be structurally and semantically identical — same sections, same tables, same code blocks, same examples. Only the language differs.

## Skill Document Format

`SKILL.md` files must follow this format:

```markdown
---
name: <skill-name>
description: "<brief description including trigger scenarios>"
---

# <Skill Title>

<One-line overview of what the skill does.>

## When to Use This Skill

<Bullet list of specific scenarios that trigger this skill.>

## Workflow

<Numbered steps describing how the agent should work through the task:>

### Step 1: ...

### Step 2: ...

...

## <Additional Sections>

<Tables for quick reference, layer details, code examples, decision flows, etc.>

## Prohibitions

<What the agent must NOT do.>

## When Unsure

<Fallback behavior when the agent cannot determine the answer.>
```

Key formatting conventions:

- Use **tables** for quick reference data (overview, naming, mappings).
- Use **numbered steps** in the Workflow section for procedural guidance.
- Use **code blocks** for directory trees and commands.
- Use **bullet lists** for scenario descriptions and constraints.
- Keep front matter `description` concise — it determines when the skill is activated.

**Reference-style skills** (quick-lookup skills such as `vue-tsx`) may replace the Workflow skeleton with topic tables plus `references/` pointers, as long as triggering info stays in the frontmatter `description`.

## Naming Conventions

- **Skill directories**: `kebab-case` (e.g., `skill-forge`, `find-skills`)
- **SKILL.md**: Always uppercase, always this exact filename
- **SKILL.zh.md**: Always uppercase prefix, always this exact filename
- **GENERATION.md**: Always uppercase, always this exact filename
- **CHANGES.md**: Always uppercase, always this exact filename
- **Reference files**: under `references/`, `kebab-case` or descriptive names

## Context Efficiency Guidelines

Skills are loaded on-demand — the agent sees only the skill name and description at startup. The full `SKILL.md` is read into context only when the agent decides the skill is relevant. To minimize context:

- **Keep SKILL.md under 500 lines** — put detailed reference material in `references/`
- **Write specific descriptions** — helps the agent know exactly when to activate
- **Use progressive disclosure** — link to reference files that get read only when needed
- **Front-load key information** — put the most important guidance early

## Working with Skills in This Repo

### Adding a New Skill

1. Create `skills/<skill-name>/` with:
   - `SKILL.md` (English, following the format above)
   - `GENERATION.md` (metadata: source, git SHA, generation date)
   - `CHANGES.md` (changelog in English)
   - `references/` (optional, for supplementary docs)
   - `evals.json` (optional but recommended: 2–3 realistic test prompts with `expected_output` and `expectations`)
2. Create `skills-zh/<skill-name>/` with:
   - `SKILL.zh.md` (Chinese, structurally identical to the English version)
3. Update `README.md` and `README.zh.md` with the new skill entry.

### Modifying an Existing Skill

1. Edit `skills/<skill-name>/SKILL.md` (English) first.
2. Synchronize all changes to `skills-zh/<skill-name>/SKILL.zh.md` (Chinese).
3. Update `skills/<skill-name>/CHANGES.md` with the modification record (English).
4. Update `README.md` and `README.zh.md` if the skill's scope or description changes.

### Verifying Consistency

After modifying a skill, compare section headers to ensure both language versions are aligned:

```bash
grep -n '^##' skills/<skill-name>/SKILL.md
grep -n '^##' skills-zh/<skill-name>/SKILL.zh.md
```

Line counts and section numbers should match.

## Current Skills

| Skill | Description | Directory |
|---|---|---|
| skill-forge | Forge, validate, and tune agent skills with pluggable runners (forked from anthropics/skills skill-creator) | `skills/skill-forge/` |

## Build / Test / Lint

This repository has no build pipeline, test suite, or linter configuration. It is a plain documentation/skills repository. Verification is manual — review the rendered markdown and check for structural consistency between language versions.
