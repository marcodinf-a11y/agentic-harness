# ADR-011: LEARNINGS.md Extraction After Final Verdict

## Status

Accepted

## Context

Rein seeds a `LEARNINGS.md` file into the sandbox at session start (FR-091). The agent appends operational knowledge during execution — build commands, test quirks, naming conventions, environment requirements. This knowledge is valuable across sessions, but the design docs do not specify **when** learnings are extracted from the sandbox back to the project root, or **how** the extraction interacts with the quality gate's retry mechanism.

The quality gate may run multiple rounds. Each round gets a fresh agent context but the same sandbox. The agent may write learnings in any round, including rounds that ultimately fail. This creates a timing question: should rein extract learnings after every round, only after the final verdict, or accumulate across rounds but write once?

Three options were considered:

**Option A — Extract after final verdict only.** Read LEARNINGS.md from the sandbox after the last round completes (regardless of verdict), diff against the project root file, merge validated new entries.

**Option B — Extract after every round, accumulate.** After each quality gate round, merge sandbox learnings into the project root immediately. Failed rounds contribute learnings alongside successful ones.

**Option C — Buffer across rounds, write once.** Collect sandbox learnings from each round into a buffer, merge and deduplicate, write to the project root after the final verdict.

## Decision

**Option A — extract after final verdict only.**

The extraction step runs once, after the quality gate has produced its final verdict (pass, warn, or fail after exhausting max_rounds). Rein reads `LEARNINGS.md` from the sandbox, diffs it against `.rein/LEARNINGS.md` in the project root, validates new entries structurally, and appends them.

### Extraction flow (post-session)

```
After final verdict (pass, warn, or fail after max_rounds):

1. Read LEARNINGS.md from sandbox
2. Read .rein/LEARNINGS.md from project root (create if missing)
3. Parse both into line sets
4. Compute new entries = sandbox lines - project lines (set difference)
5. Structural validation on each new entry:
   - Must start with "- "
   - Must be <= 200 characters
   - Must not duplicate an existing entry (exact match)
6. Append validated new entries to project file
7. If total exceeds 80 lines, warn operator (do not truncate)
8. Write back to .rein/LEARNINGS.md
```

### Injection flow (session start)

```
During SANDBOX step (step 2 in session lifecycle):

1. Check if .rein/LEARNINGS.md exists in project root
2. If yes: copy into sandbox as LEARNINGS.md
3. If no: create empty LEARNINGS.md in sandbox with "# Learnings" header
```

### Structural validation rules

Rein validates new entries with code — no LLM validator. The learnings are operational facts (build commands, test flags, conventions) that are verifiable by running the commands, not by asking another model to judge prose quality. An LLM validator cannot reliably distinguish "correct but surprising" from "wrong," and the per-session cost compounds across multi-session workflows.

Validation rules:
- Entry must be a single line starting with `- `
- Entry must be 200 characters or fewer (forces conciseness)
- Entry must not exactly duplicate an existing line
- Total file capped at 80 lines (warn, do not truncate — operator curates)

### Prompt convention

The existing work protocol instruction (PROMPTS.md §3) already includes:

> If LEARNINGS.md exists, read it before starting. If you discover reusable insights (gotchas, patterns, constraints), append them — keep the file under 80 lines.

An additional category prefix convention is recommended:

> Prefix entries with a category: `build:`, `test:`, `lint:`, `env:`, `convention:`, `quirk:`.

Example:
```
- build: `cargo build --release` (debug builds fail timing tests)
- test: `cargo test -- --test-threads=1` (DB tests not parallelizable)
- quirk: auth module uses custom JWT, not the jsonwebtoken crate
```

Categories make the file scannable for operators and future sessions without requiring the agent to rank importance.

### Code location

In `runner.py` (orchestrator), two call sites:

```python
# After sandbox setup, before agent invocation
learnings.inject(sandbox_path, project_root)

# After final verdict, before cleanup
learnings.extract(sandbox_path, project_root)
```

A `learnings.py` module provides:
- `inject(sandbox_path, project_root)` — copy or create
- `extract(sandbox_path, project_root)` — diff, validate, merge
- `validate_entry(line) -> bool` — structural checks

## Rationale: Why Option A Over B and C

**Failed rounds are the most likely source of wrong facts.** If a round fails because the agent's code was incorrect, the agent may have written a learning like "this test is flaky" when the test was actually fine — the agent's code was the problem. Extracting only from the final sandbox state avoids propagating these misattributions.

**Successful retries rediscover real operational facts.** If round 1 fails but discovers that tests need `--test-threads=1`, round 2 (a fresh context with the same sandbox) will rediscover this fact when it runs the tests. The knowledge isn't lost — it's confirmed by a round that also produced correct work.

**Simpler implementation.** One extraction point, one merge, one diff. No buffering, no cross-round deduplication, no partial rollback if the final verdict is "fail."

**Option B's risk: wrong learnings compound.** If round 1 writes a wrong learning and it's immediately merged to the project root, round 2 reads it as a seed and may be misled. Option A avoids this by only extracting from the final state.

**Option C's marginal benefit doesn't justify complexity.** Buffering across rounds captures learnings from failed rounds that might be correct, but structural validation can't distinguish correct-from-failed-round vs. wrong-from-failed-round. The only reliable filter is "did the round succeed?" — and the simplest version of that filter is "use the final sandbox state."

## Consequences

**Positive:**

- Learnings persist across sessions with no manual copy step.
- Structural validation prevents format drift and unbounded growth without an LLM call.
- Single extraction point keeps the implementation simple and the data flow predictable.
- Warn-don't-truncate respects operator agency — the operator knows which learnings to keep.

**Negative:**

- Learnings from failed-only runs (all rounds fail) still get extracted. These may contain incorrect facts. Mitigation: the operator can curate `.rein/LEARNINGS.md` between sessions.
- Exact-match deduplication won't catch semantically equivalent entries with different wording. Mitigation: acceptable for a file capped at 80 lines — operator curation handles semantic duplicates.

**Neutral:**

- This is the first "write-back" pattern in rein — all other data flows are sandbox → report. The `.rein/LEARNINGS.md` write-back sets a precedent for future artifact persistence (e.g., DEFERRED.md). The interface (`inject`/`extract` pair) is designed to be reusable.
- No interaction with context pressure monitoring — extraction is a post-execution concern.
