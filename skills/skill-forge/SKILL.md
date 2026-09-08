---
name: skill-forge
description: "Forge, validate, and tune agent skills with pluggable runners — no tool lock-in. Use when scaffolding a new skill pack, drafting or restructuring a SKILL.md, running test prompts as with/without benchmark iterations (via in-host subagents or external runners such as claude, codex, opencode, or pi), grading expectations and aggregating benchmarks, or optimizing a skill description for triggering (native, explicit, or simulated judge strategies). Not for repo-specific packaging conventions — defer to the host repository's authoring skill (e.g. skill-dev) when present."
metadata:
  author: LvHeng
  version: "2026.9.2"
  source: Forked & distilled from anthropics/skills skill-creator (Apache-2.0); external dependencies abstracted behind a runner layer
---

# Skill Forge

Forge agent skills and validate them empirically — with **no runner hard-coded**. Every external capability (one-shot LLM calls, agent sessions, trigger probing) goes through a pluggable runner layer, defaulting to whatever the current host already provides.

## When to Use This Skill

- Scaffolding a new skill pack from scratch or from a captured workflow
- Drafting, restructuring, or slimming a SKILL.md
- Running test prompts as with/without benchmark iterations
- Grading expectations, aggregating benchmarks, reviewing outputs
- Optimizing a skill `description` for triggering accuracy
- Porting an eval/optimization loop to a different agent tool (claude, codex, opencode, pi, …)

## Core Principles

1. **Progressive disclosure** — context is the scarce resource. L1 frontmatter (`name` + `description`, ~100 words, always loaded) earns the trigger; L2 body (<500 lines, on trigger); L3 `references/` + `scripts/` (on demand). Detailed material never inflates the body.
2. **Explain the why** — an instruction with its reasoning generalizes to unanticipated cases; a bare `ALWAYS`/`NEVER` does not. Reserve hard prohibitions for what is genuinely always wrong.
3. **Generalize, don't overfit** — you iterate on a few test cases but ship to countless future prompts. Ask "does this fix generalize?" before applying it.
4. **Script the deterministic parts** — scaffolding, validation, aggregation belong in `scripts/`, not in repeated agent effort.
5. **Host-agnostic by default** — never hard-code a vendor CLI in the workflow. Prefer the in-host path; degrade gracefully via the runner layer ([references/backends.md](references/backends.md)).

## Workflow

### Step 1: Capture Intent

Answer before writing anything; extract from the conversation first if the user says "turn this into a skill":

1. What should the skill enable the agent to do?
2. When should it trigger? (user phrasings, contexts, near-misses)
3. What is the expected output/behavior when it fires?
4. Are outcomes objectively verifiable? If yes, plan test prompts (Step 5); if subjective (writing style, design), rely on qualitative review.

### Step 2: Scaffold the Package

```text
<skill-name>/            # kebab-case, matches frontmatter name
├── SKILL.md             # required
├── references/          # detailed docs, loaded on demand
├── scripts/             # deterministic/repetitive logic, executable without loading
├── assets/              # files used in output (templates, fonts, icons)
└── evals.json           # test prompts + expectations
```

Multi-domain skills organize `references/` by variant so only the relevant file gets read.

### Step 3: Write the Frontmatter & Description

The description is the trigger mechanism: it must carry **what the skill does** and **when to use it** — 3+ concrete trigger scenarios, domain nouns users actually type, slightly pushy (models under-trigger). No usage instructions in the description; that belongs in the body.

**Weak:** `description: "Vue component library conventions."`

**Strong:** `description: "Vue 3 component library authoring conventions. Use when writing or reviewing library components, designing props/emits/slots APIs, deciding emits vs callback props in JSX, or organizing tests and hooks in a component library."`

### Step 4: Write the Body

- One-line overview → trigger scenarios → workflow (numbered `### Step N:` for procedures) → additional sections (tables for quick reference) → Prohibitions / When Unsure only if they earn their place.
- Front-load key guidance; assume later sections may never be read.
- Reference files with a pointer saying **when** to read each; files >300 lines need a table of contents.
- Prefer tables for mappings, code blocks for directory trees and commands.

### Step 5: Create Test Prompts

Write 2–3 realistic prompts — what a real user would type, concrete and substantive enough that consulting the skill genuinely helps. Record them with `expected_output` and `expectations` (verifiable statements) in `evals.json` at the skill root. Full schema: [references/schemas.md](references/schemas.md).

```json
{
  "skill_name": "<skill-name>",
  "evals": [
    {
      "id": 1,
      "prompt": "The task as a user would phrase it",
      "expected_output": "What a correct result looks like",
      "expectations": ["Verifiable statement 1", "Verifiable statement 2"]
    }
  ]
}
```

### Step 6: Run the Eval Loop

Workspace layout (the aggregation script requires the `run-N` level):

```text
<skill-name>-workspace/
└── iteration-1/
    └── eval-<descriptive-name>/
        ├── eval_metadata.json          # {eval_id, eval_name, prompt, assertions}
        ├── with_skill/run-1/           # grading.json + outputs/ (+ timing.json)
        └── without_skill/run-1/
```

**Executor runs** — spawn one run per configuration, all in the same turn:

- *In-host path (default, zero external dependencies):* use the host's native subagent capability (pi subagents, Claude Code Task tool, …). Each with_skill run reads the skill path first and applies it; each without_skill run receives only the task.
- *External runner path:* route through a backend adapter — see [references/backends.md](references/backends.md) for the per-tool commands and detection notes.
- **Contamination rule:** without_skill runs must not be able to discover the skill. When the skill lives in the same repo the agent searches, execute from a pruned copy (or neutral cwd) — agents do read skill files they find, and it silently invalidates your baseline.

**Grading** — check programmatically wherever possible (scripts beat eyeballing); use an LLM grader (host model or `complete()` runner) for judgment calls. Write `grading.json` with `expectations[{text, passed, evidence}]` — exact field names matter to the viewer.

**Aggregate & review:**

```bash
python3 scripts/aggregate_benchmark.py <workspace>/iteration-1 --skill-name <name>
python3 scripts/generate_review.py <workspace>/iteration-1 \
  --skill-name <name> --benchmark <workspace>/iteration-1/benchmark.json \
  --static <workspace>/review.html
```

Add an analyst pass on top of the aggregates: assertions that pass in both arms (non-discriminating), high-variance evals (flaky), time/token trade-offs — record them in `benchmark.json` `notes`.

### Step 7: Iterate on Feedback

- Fix causes, not symptoms; a fiddly patch for one test case is overfitting.
- Remove instructions that aren't pulling their weight; read transcripts, not just outputs.
- If every executor run independently reinvented the same helper, promote it into the skill's `scripts/`.
- Rerun as `iteration-2/` (baseline = previous iteration or no-skill; keep it consistent), regenerate the viewer with `--previous-workspace`, and stop when feedback is empty or progress stalls.

### Step 8: Optimize the Description

Build a trigger eval set: 20 realistic queries — 8–10 should-trigger (varied phrasings, some not naming the skill) + 8–10 near-miss should-not (shared keywords, adjacent domains, ambiguous phrasing; never obviously irrelevant). Save as `trigger-evals.json`.

Pick a probing strategy by fidelity and availability (details: [references/backends.md](references/backends.md)):

| Strategy | How | Fidelity | Needs |
|---|---|---|---|
| `native` | install skill into the tool's own skill dir, run real sessions, detect skill usage in transcripts | highest | claude, pi |
| `explicit` | task says "read skill X first"; measures execution quality, not triggering | medium | any `run_agent` |
| `simulated` | give a judge only name+description list + query; ask which skill it would consult | sufficient for description tuning | any `complete()` |

Default to `simulated` (works everywhere, proven effective at catching body-only capabilities that can't trigger). Split 60/40 train/holdout, evaluate, let a strong model propose improved descriptions from failures, re-evaluate, iterate ≤5 rounds, select by **holdout** score, apply the best. When the host model itself must not judge its own description, swap judges via the runner layer.

### Step 9: Package & Integrate

Adapt to the host's conventions: provenance metadata (`GENERATION.md`), changelog (`CHANGES.md`), bilingual mirrors, skill catalogs/README indexes. Update the index wherever the host lists its skills — including install commands. Optionally package with `scripts/package_skill.py` (see NOTICE) when a `.skill` artifact is wanted.

## Prohibitions

- Do not hard-code a vendor CLI in the workflow or scripts — every external invocation goes through a runner adapter or the in-host path.
- Do not let without_skill runs discover the skill (baseline contamination invalidates the benchmark).
- Do not stuff usage instructions or long tables into the `description`.
- Do not let a SKILL.md grow past ~500 lines by inlining reference material.
- Do not ship skills whose content would surprise the user relative to their stated intent.
- Do not claim "verified" without real command output or review evidence behind it.

## When Unsure

- Which trigger strategy → `simulated` for description work; `inhost`/`run_agent` for execution quality; `native` only when the backend's transcript parsing is actually implemented.
- Whether a section is needed → omit it; add back when a real case demands it.
- How rigorous testing should be → default 2–3 prompts with recorded expectations; deeper benchmarks when the user asks or the skill is high-traffic.
- Which backend is available → probe per [references/backends.md](references/backends.md) detection order; if none, everything above still works in-host.
