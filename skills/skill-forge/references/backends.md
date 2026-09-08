# Runner & Backend Reference

Normative reference for skill-forge's runner abstraction. The SKILL.md workflow is
runner-agnostic; this file maps each primitive onto concrete tools. Status notes
reflect 2026-09-02 probing on the author's machine — re-probe before relying on them.

## Runner contract

```python
class Runner:
    name: str

    def available(self) -> bool:
        """CLI on PATH (or host capability present) AND authenticated."""

    def complete(self, prompt: str, model: str | None = None, timeout: int = 120) -> str:
        """One-shot text generation. Powers: description improvement, LLM grading,
        simulated trigger judge. No skill awareness required."""

    def run_agent(self, task: str, cwd: str, timeout: int = 600,
                  skill_dirs: list[str] | None = None) -> RunResult:
        """One agent session, optionally with skills mounted.
        Powers: trigger probing (native strategy), executor runs.
        RunResult = {output_text, skill_used: bool | None, transcript_path}"""
```

`skill_used` is `None` when the backend cannot observe skill usage — callers must
treat that as "unknown", never as "not triggered".

## Backend table

| Backend | `complete()` | `run_agent()` | Skills live at | `skill_used` detection | Status 2026-09-02 |
|---|---|---|---|---|---|
| `inhost` | host model itself (the agent running this skill) | host's native subagent tool (pi subagents, Claude Code Task tool, …) | already installed by host | n/a (explicit with_skill runs) | **always available** — default path |
| `claude` | `claude -p "<prompt>" --output-format json` | `claude -p "<task>"` | `~/.claude/skills/` | parse transcript for skill invocation | installed+authed but `claude -p` **hangs** (exit 124 after 90s) — probe before use |
| `codex` | `codex exec "<prompt>"` | `codex exec --cd <cwd> "<task>"` | no native skills — inject via prompt (explicit strategy only) | not observable → `None` | available |
| `opencode` | `opencode run "<prompt>"` | `opencode run "<task>"` | AGENTS.md-ecosystem conventions | depends on version | available (also the active `PI_PROVIDER` backend) |
| `pi` | pi headless/SDK one-shot | pi SDK script or `pi` non-interactive | native skills system (like this host) | native `available_skills` + transcript | available as host; headless adapter = P2 |

Detection order for `--runner auto`: `inhost` (always usable) → `codex` → `opencode` → `claude` → `pi`. Skip backends whose probe (a 10-second `complete("reply ok")` with timeout) fails.

## Trigger-probing strategies

| Strategy | Mechanism | Measures | Fidelity | Requires |
|---|---|---|---|---|
| `native` | skill installed in the tool's own skill dir; real sessions; parse transcripts for skill usage | real triggering behavior | highest | backend with observable `skill_used` |
| `explicit` | task instructs "read skill X first" | execution quality with skill present | medium | any `run_agent` |
| `simulated` | judge sees ONLY the name+description list + one query; returns which skill(s) it would consult | whether the description wins the consult decision | sufficient for description tuning | any `complete()` |

Fidelity caveat: `simulated` cannot see what happens *after* a skill is loaded — execution-quality questions belong to executor runs (Step 6), not to trigger evals. Empirical basis: a simulated-judge pass (20 queries/skill, fresh-context judge) correctly identified 20/20 trigger behaviors for one skill and surfaced 3 real false negatives in another, each caused by capabilities documented only in the body (invisible at trigger time).

### Simulated judge prompt template

```text
You are simulating the skill-triggering decision of an AI coding assistant.
The assistant has the following available_skills list (name + description ONLY —
bodies are not visible at decision time):

- <name>: <description>
...

For EACH user query below, decide which skill(s) the assistant would consult
based strictly on the descriptions. If no skill clearly matches, answer ["none"].
Return ONLY a JSON array: [{"id": 1, "consult": ["skill-name"]}, ...]

Queries:
1. <query>
...
```

Rules: judge runs with fresh context (never inherits the authoring conversation);
query order shuffled; the judge never sees `should_trigger` labels; borderline
"would this trigger?" queries are the valuable ones — discard obviously-irrelevant
negatives from the set before running.

## Trigger eval set contract

`trigger-evals.json` at the skill root:

```json
[
  {"id": 1, "query": "realistic user query", "should_trigger": true},
  {"id": 2, "query": "near-miss query", "should_trigger": false}
]
```

20 queries: 8–10 should-trigger (varied phrasings, some never naming the skill) +
8–10 near-miss should-not (shared keywords, adjacent domains, ambiguous phrasing).

## Porting contract for the optimization loop (P2)

Upstream `run_eval.py` / `run_loop.py` / `improve_description.py` (anthropics/skills)
hard-code `claude -p`. Port plan — schemas and directory contracts stay unchanged:

1. Add `scripts/runners/{base,claude,codex,opencode,pi,inhost}.py` implementing the contract above.
2. Thread a `--runner auto|inhost|claude|codex|opencode|pi` flag through the three scripts; `auto` uses the detection order above.
3. `run_loop.py` gains `--strategy native|explicit|simulated` (default `simulated`); `native` requires a backend with observable `skill_used`.
4. Keep `--holdout` 60/40 split, `--max-iterations`, and selection-by-holdout semantics identical to upstream.
5. Until ported, run the loop manually per SKILL.md Steps 6/8 — every artifact lands in the same schemas, so results stay comparable.

## Contamination rule (applies to every strategy)

Executor `without_skill` runs must not be able to discover the skill. When the
skill lives in the same tree the agent searches (e.g. `skills/<name>/SKILL.md`),
run executors from a pruned copy or neutral cwd. Confirmed failure mode: baseline
agents found and read the skill file in-repo, cited it in their output, and
produced fully conforming results — silently zeroing the measured delta.
