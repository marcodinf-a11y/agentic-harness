# ADR-015: PROGRESS.md and DEFERRED.md Sandbox Seeding

## Status

Accepted

## Context

Rein's work protocol (FR-094) instructs the agent to maintain PROGRESS.md (track what's done, in progress, blocked) and DEFERRED.md (park out-of-scope discoveries). FR-062 says "update PROGRESS.md with task summary." FR-095's deviation rules direct the agent to log out-of-scope work to DEFERRED.md.

However, no FR covers *creating* these files at sandbox setup time. Without seeding, the agent must create them from scratch — with no template, no size constraint, and no consistent format across runs. FR-091 already establishes the seeding pattern for LEARNINGS.md; PROGRESS.md and DEFERRED.md need the same treatment.

Three questions drove this decision:

1. **Seed from project root or start empty?** LEARNINGS.md seeds from the project root because operational knowledge accumulates across sessions. PROGRESS.md tracks *this run's* progress. DEFERRED.md tracks *this run's* out-of-scope items. Both are per-run artifacts.

2. **Size cap?** GSD's STATE.md uses a < 100-line constraint to prevent progress files from becoming context pressure sources. Research across deep dives (Ralph Wiggum's `fix_plan.md`, GSD's STATE.md) confirms that unbounded progress files degrade agent performance.

3. **Carry-forward on retry?** FR-093b establishes sandbox state policy: carry forward on verdict "fail", fresh sandbox otherwise. These files should follow the sandbox — if the sandbox carries forward, so do the files.

## Decision

Seed both files at sandbox setup time with minimal templates. Always start empty (never seed from project root). Follow the sandbox state policy from FR-093b on retries.

### PROGRESS.md Template

```markdown
# Progress

## Completed


## In Progress


## Blocked


---
*Keep under 100 lines. Summarize, don't narrate. Remove completed items when space is needed.*
```

### DEFERRED.md Template

```markdown
# Deferred

Items discovered during work that are out of scope for this task.

---
*One line per item. Prefix with file path where relevant.*
```

### Design choices

**PROGRESS.md capped at 100 lines.** GSD uses 80–100 for STATE.md. 100 lines is enough for meaningful progress tracking without becoming a context pressure source. The cap is advisory (enforced by prompt instruction, not by Rein code) — consistent with LEARNINGS.md's 80-line advisory cap.

**DEFERRED.md uncapped.** It's a parking lot for out-of-scope items, typically short. It won't be injected into prompts across rounds (unlike LEARNINGS.md). If a task generates enough deferred items to matter, the operator should review task scope.

**Both start empty every run.** PROGRESS.md is per-run state — carrying progress notes from a previous task's run would confuse the agent. DEFERRED.md items from previous runs are operator concerns, not agent concerns. This contrasts with LEARNINGS.md, which seeds from the project root because operational knowledge transfers across runs.

**Carry-forward follows sandbox policy.** On retry with carried-forward sandbox (FR-093b: validation ran and failed), both files remain as-is — the agent sees its previous progress alongside the code it wrote. On fresh sandbox retry, both files reset to the empty template.

**No write-back to project root.** Unlike LEARNINGS.md (ADR-011), neither file is extracted back to `.rein/`. PROGRESS.md is ephemeral per-run state. DEFERRED.md *could* be extracted in the future (ADR-011 anticipated this), but for R1 the items are captured in the JSON report via the escalation report (FR-092) and that's sufficient.

### Seeding flow

```
During sandbox setup (after workspace creation, alongside FR-091 LEARNINGS.md injection):

1. Write PROGRESS.md template to sandbox root
2. Write DEFERRED.md template to sandbox root
3. (FR-091 handles LEARNINGS.md separately — seed from project root or create)
```

### Code location

In `sandbox.py`, called from `runner.py` during sandbox setup:

```python
# After workspace creation and file seeding (FR-013)
sandbox.seed_progress_md(sandbox_path)
sandbox.seed_deferred_md(sandbox_path)
learnings.inject(sandbox_path, project_root)  # existing, per ADR-011
```

## Rationale

**Why not seed from project root?** PROGRESS.md from a previous run describes different work. Injecting it would mislead the agent about what's already done in *this* run. DEFERRED.md items from previous runs are for the operator to triage — they may have been addressed, deprioritized, or assigned elsewhere.

**Why templates instead of empty files?** The template's section headers (Completed / In Progress / Blocked) give the agent a consistent structure without requiring the prompt to describe the format. The footer line reinforces the size cap at the point of use.

**Why advisory cap, not enforced?** LEARNINGS.md (ADR-011) uses the same pattern — warn at threshold, don't truncate. Enforcing a hard cap would require Rein to parse and truncate markdown mid-run, which is fragile and may discard the agent's most recent (and most relevant) entries.

## Consequences

**Positive:**

- Agent always has structured files ready — no "create the file yourself" friction.
- Size cap prevents progress files from becoming context pressure sources.
- Consistent format across all runs makes post-hoc analysis easier.
- Templates are trivially small (~8 lines each), negligible sandbox setup cost.

**Negative:**

- DEFERRED.md items are only captured in the sandbox and JSON report, not persisted to the project root. Operators who want a running deferred list must extract manually. Acceptable for R1; write-back can be added later following ADR-011's inject/extract pattern.

**Neutral:**

- Sets precedent for additional sandbox seed files. The pattern (template in code, seeded at setup, follows sandbox carry-forward policy) is reusable.
