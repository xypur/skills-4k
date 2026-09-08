# Generation Info

- **Source:** Forked & distilled from `skill-creator` (anthropics/skills, Apache-2.0)
  - Upstream vendored at: `/home/lvheng/dev/coding/skills/.agents/skills/skill-creator` (pinned by that repo's `skills-lock.json`)
- **Forked:** 2026-09-02
- **License:** Apache-2.0 (`./LICENSE`) — inherited verbatim from upstream per skills-4k's per-skill licensing policy; see `scripts/NOTICE.md`.
- **Rationale:** Upstream hard-codes `claude -p` as the subprocess backend for trigger
  evaluation and description optimization, and bakes Claude-specific environment
  branches into the body. skill-forge abstracts every external capability behind a
  runner layer (`complete()` / `run_agent()` / in-host), defaults to a zero-dependency
  in-host path, and distills the ~485-line upstream body (with its Claude.ai/Cowork
  branches) into a tool-agnostic workflow.
- **Reused verbatim (Apache-2.0, see `scripts/NOTICE.md`):**
  `scripts/aggregate_benchmark.py`, `scripts/generate_review.py`, `scripts/viewer.html`,
  `references/schemas.md`
- **New in this fork:** runner abstraction & backend table (`references/backends.md`),
  trigger-probing strategy tiers (native / explicit / simulated), in-host default path,
  contamination rule (baseline agents must not discover the skill), `trigger-evals.json`
  convention.
- **Deferred (P2):** Python port of `run_eval.py` / `run_loop.py` /
  `improve_description.py` behind the runner contract — until then, Steps 6/8 run
  manually in-host and land in identical schemas.
