# skills-4k

English | [中文](./README.zh.md)

A curated collection of skill packs for AI coding assistants.

## Available Skills

### skill-forge

Forge, validate, and tune agent skills with pluggable runners — no tool lock-in. Scaffolds skill packs, runs with/without benchmark iterations (in-host subagents or external runners: claude, codex, opencode, pi), grades expectations, aggregates benchmarks, and optimizes descriptions for triggering (native / explicit / simulated judge strategies). Forked & distilled from [anthropics/skills](https://github.com/anthropics/skills) `skill-creator`.

**Install:**

```bash
npx skills add https://github.com/xypur/skills-4k --skill skill-forge
```

## Usage

Install any skill from this repository with the Skills CLI:

```bash
npx skills add https://github.com/xypur/skills-4k --skill <skill-name>
```

Point AI assistants that support the skills system at the `skills/` directory to give them context-aware guidance.

**Author:** LvHeng
