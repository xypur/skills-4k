# Changelog

## 2026-09-02 — Complete Chinese mirror references

Added `skills-zh/skill-forge/references/backends.md` and `skills-zh/skill-forge/references/schemas.md` — Chinese translations of both English reference documents, so SKILL.zh.md's relative links resolve within the mirror directory. Section structure, tables, JSON examples, and code blocks match the English versions verbatim (only prose translated). No English content changed.

## 2026-09-02 — Migrated to skills-4k; per-skill Apache-2.0 license

Moved from private-skills to the skills-4k repository, which uses per-skill licensing (no root license by design). Added `LICENSE` (Apache-2.0, inherited verbatim from upstream skill-creator) at the skill root per that policy; `scripts/NOTICE.md` now points at it. No SKILL.md content changed.

## 2026-09-02 — Initial fork from skill-creator

Forked & distilled from anthropics/skills `skill-creator` into a runner-agnostic
skill pack. All upstream external capabilities (`claude -p` subprocesses) are
replaced by a pluggable runner layer with an in-host zero-dependency default.

### Changes

- `SKILL.md` → new (~150 lines, distilled from ~485): 5 core principles (adds
  "host-agnostic by default"), 9-step workflow (capture intent → scaffold →
  frontmatter → body → test prompts → eval loop → iterate → description
  optimization → package/integrate), runner-strategy table, contamination rule.
- `references/backends.md` → new: runner contract, backend table (claude / codex /
  opencode / pi / inhost) with per-tool commands and detection notes, trigger
  strategy tiers with fidelity ranking, simulated-judge prompt template,
  trigger-evals.json contract, P2 porting plan for the optimization loop.
- `references/schemas.md` → copied verbatim from upstream.
- `scripts/aggregate_benchmark.py`, `scripts/generate_review.py`, `scripts/viewer.html`
  → copied verbatim (Apache-2.0, see `scripts/NOTICE.md`); viewer template path
  verified to resolve relative to the script location.
- `evals.json` → 3 self-test prompts (forge a skill from a described workflow;
  design a trigger eval set; validate a description change).
- Deferred to P2: `runners/` adapters and the refactored `run_eval` / `run_loop` /
  `improve_description` scripts.
