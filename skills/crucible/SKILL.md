---
name: crucible
description: "Targeted multi-domain codebase audit. Use when auditing a user flow end to end, investigating issues that span several domains (auth, data, API, UI), or running a structured find-then-fix cycle over a named area. Not for reviewing a single file or diff — that's code-review — or for fixing a bug whose cause is already known."
disable-model-invocation: true
---

# Crucible

Parallel domain recon → adversarial kill → plan → verify → SPEC → implement → PR.

## File Ownership Rule

**Subagents NEVER write files.** They return results as text. You (the orchestrator) write every artifact. Files are the cross-phase handoff — subagent memory is not.

## Setup

Parse the invocation for:

- **Domains** (required) — user-facing areas to audit, e.g. `player`, `curator`
- **Scope hint** (optional) — narrow focus within a domain, e.g. `lobby SSE`, `role selection`

Determine output directory: `.crucible/run-<N>/` where N is the next unused integer.

**Prior-run check**: scan `.crucible/` for existing run directories. Read each `findings.json`. Feed their confirmed IDs as a skip list into Phase 1 agents — don't re-discover known fixed issues; weight effort toward gaps.

Files written this run (all by orchestrator):

| File | Phase | Role |
|------|-------|------|
| `codebase-map.md` | 1 | Context injected verbatim into all downstream agents |
| `recon-<domain>.json` | 1 | Per-domain raw findings |
| `findings.json` | 1→2→4 | Merged findings; verdicts appended each phase |
| `PLAN.md` | 3 | Prioritized fix plan |
| `SPEC.md` | 5 | Before/after code + test cases per finding |

## Phase 1 — Recon

MANDATORY READ [`references/recon-phase.md`](references/recon-phase.md) before launching agents.

Launch one **research** agent per declared domain, all in parallel. Feed each: codebase path, domain name, scope hint (if any), prior known IDs to skip.

Agents return JSON text. If an agent returns prose instead of a JSON array, re-prompt once with the exact schema shape from `assets/findings-schema.json`. If it fails again, write an empty `recon-<domain>.json` (`[]`) and note the failure in `codebase-map.md`. Continue — one failed domain is recoverable; do not abort the run.

Collect valid responses, then:

1. Write `recon-<domain>.json` per agent response
2. Synthesize `codebase-map.md` — key file paths, domain boundaries, shared modules, conventions
3. Merge all domain findings into `findings.json`

Completion criterion: every domain has a `recon-<domain>.json`, `codebase-map.md` written, `findings.json` written. Do NOT load `references/adversarial-phase.md` until Phase 2.

## Phase 2 — Adversarial Review

MANDATORY READ [`references/adversarial-phase.md`](references/adversarial-phase.md) before launching agents.

Launch one **general** agent per finding (batch findings from the same domain). Each agent gets the finding JSON and `codebase-map.md` — NOT the recon agent's reasoning. Independence boundary is the quality gate.

Agents return `CONFIRMED` / `REJECTED` / `DOWNGRADED` with code evidence. Append each verdict to the corresponding entry in `findings.json`. Run `node scripts/validate-findings.cjs findings.json` — fix any schema errors before Phase 3.

**When the validator itself can't run** — `node` not on PATH, the script or its schema missing, a usage error, or any failure before validation begins (`Failed to load schema`, `Failed to parse JSON`) — this is not a pass. Note `⚠ findings unvalidated — validator unavailable: <reason>` in the findings output and at the top of the SPEC handoff, then continue. A schema *rejection* is the opposite case and always blocks: fix the findings, re-run, do not proceed on a rejection. Both currently exit `1`, so read the error text, not the exit code.

Completion criterion: every finding has an `adversarial_verdict` field. Schema validation passes. Do NOT reload `references/recon-phase.md` from this point forward.

## Phase 3 — Plan Synthesis

**Zero-findings check**: before synthesizing, count confirmed + downgraded entries in `findings.json`. If zero, write `.crucible/run-<N>/REPORT.md` with the single line "No actionable findings after adversarial review." Stop — do not open a PR.

Synthesize confirmed + downgraded findings from `findings.json` into `PLAN.md`:

- **P1 (critical/high)**: broken core flows, data loss, permanent disconnection
- **P2 (medium)**: user-visible errors, missing recovery paths
- **P3 (low)**: polish, doc drift, copy issues

Group findings that share a root cause — one fix may eliminate several. When grouping: the highest severity among the grouped findings sets the group's priority. If grouped findings have conflicting fix approaches, split them — shared root cause does not require a shared fix. Declare fix order where dependencies exist. Rejected findings excluded entirely.

Completion criterion: `PLAN.md` written; every confirmed finding appears exactly once.

## Phase 4 — Independent Verification

Launch one **research** agent per P1 + P2 finding, all in parallel. Each agent is told: "You did NOT write this finding. Verify every factual claim against current source code."

Each agent checks: file exists at path, line number matches described code, scope name is correct, pattern described actually appears. Returns `VERIFIED` / `CORRECTED [field: wrong → right]` / `REJECTED [reason]`.

Apply all corrections to `PLAN.md` and `findings.json`. Run schema validation again. `PLAN.md` and `findings.json` must agree — no disagreement between deliverables.

Completion criterion: all P1+P2 findings verified. All corrections applied. Schema passes.

## Phase 5 — SPEC

Write `SPEC.md` with one section per confirmed finding (P1 → P2 → P3 order):

- Exact file path + line range (verified)
- Before code (real source, not pseudocode)
- After code (concrete fix)
- Test cases: one sentence per behavior the test must verify

Run `node scripts/validate-findings.cjs findings.json` final check.

Completion criterion: every PLAN.md entry has a SPEC section. Validation passes.

## Phase 6 — Handoff

1. Update flow docs (`docs/flows/`) for any finding that changes a user-facing behaviour
2. Invoke the `implement` skill with `SPEC.md`
3. Open a PR — this is the HITL gate; no intermediate approval is requested

PR body: link to `PLAN.md`, finding count by severity, link to `SPEC.md`, list of flow docs updated.

## NEVER

- **NEVER ask a subagent to write a file**
  **Instead:** Have the agent return the content as text; write the file yourself.
  **Why:** Cross-phase state lives in files. If an agent writes directly, the orchestrator loses the canonical copy and cannot reconcile deliverables after Phase 4 corrections.

- **NEVER give an adversarial agent the recon agent's chain of reasoning**
  **Instead:** Pass only the structured finding (JSON) and `codebase-map.md`.
  **Why:** Seeing the recon agent's logic anchors the adversarial agent to the same assumptions — the independence boundary is what kills false positives.

- **NEVER proceed past Phase 2 with schema validation failures**
  **Instead:** Fix the failing fields in `findings.json` before synthesizing the plan.
  **Why:** A PLAN built on malformed findings produces a SPEC with wrong file paths and line numbers that the implement skill cannot use.

- **NEVER write `SPEC.md` while any P1/P2 finding has `verification: "pending"`**
  **Instead:** Run `node scripts/validate-findings.cjs findings.json --pre-spec` and resolve all failures before Phase 5.
  **Why:** The `--pre-spec` guard exists in the validator script but is invisible unless invoked — an agent that skips this call will SPEC unverified file paths and silently pass wrong line numbers to implement.
