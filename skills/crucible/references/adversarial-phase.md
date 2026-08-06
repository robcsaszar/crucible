# Adversarial & Verification Phase — Agent Contracts

## Independence Boundary

Both adversarial and verification agents receive:

- The structured finding JSON
- `codebase-map.md`

They do NOT receive:

- The recon agent's prose reasoning
- Other findings from the same run
- Any context about what the recon agent "intended"

Violating the independence boundary collapses the adversarial review into a rubber stamp.

---

## Adversarial Agent Prompt Template

Launch one **general** agent per finding (or per domain batch). Replace `[FINDING_JSON]` and `[CODEBASE_MAP]`.

```text
You are an adversarial reviewer. Your job is to DISPROVE this finding.
Read the actual source code at every step cited.

Codebase context:
---
[CODEBASE_MAP]
---

Finding to disprove:
[FINDING_JSON]

Apply all four tests:

1. EXISTENCE TEST — Read the file at the exact path and line(s) cited.
   Does the described code actually exist there? If the file or line is wrong, the finding is rejected.

2. MITIGATION TEST — Is there another layer that already handles this?
   Middleware, framework defaults, database constraints, a wrapper that the recon agent missed.
   If yes, the finding is rejected or downgraded.

3. IMPACT TEST — What does a user actually experience if this issue is present?
   If the answer is "nothing visible" or "a slightly ugly error message", severity is lower than claimed.

4. DESIGN TEST — Is this intended behaviour? Read the surrounding code for intent.
   If the code is doing what the developer designed, it is not a bug.

Return exactly one of:
- "CONFIRMED: [code evidence — file:line — why it's real]"
- "REJECTED: [what specifically is wrong — file:line evidence]"
- "DOWNGRADED to [new_severity]: [why impact or likelihood is lower]"
```

## Verdict Mapping

| Verdict | Action |
|---------|--------|
| CONFIRMED | Set `adversarial_verdict: "confirmed"` in findings.json |
| REJECTED | Set `adversarial_verdict: "rejected"`, move to rejected array in findings.json |
| DOWNGRADED | Set `adversarial_verdict: "downgraded"`, update `severity` field |

After all verdicts are written, run the schema validator. Rejected findings must still appear in findings.json (with `verdict: "rejected"`) — the record of what was disproved is part of the audit trail.

---

## Verification Agent Prompt Template (Phase 4)

Launch one **research** agent per P1 + P2 confirmed finding. Replace `[FINDING_JSON]`.

```text
You did NOT write this finding. Your job is to verify every factual claim against current source code.

Finding:
[FINDING_JSON]

Check each of these, in order:

1. PATH — Does the file exist at the exact path in "files"?
2. LINE — Read the cited line number. Does it match the described code?
   If the line is wrong, find the correct line and return the correction.
3. SCOPE — Is the function/component name in root_cause correct?
4. PATTERN — Does the pattern described in fix_approach actually appear in the current code?
   (The codebase may have changed since recon ran.)

Return exactly one of:
- "VERIFIED"
- "CORRECTED [field]: [wrong value] → [correct value]" (one line per correction)
- "REJECTED: [reason — the finding is fundamentally wrong]"
```

## Reconciliation Rule

After applying all Phase 4 corrections:

- `PLAN.md` and `findings.json` must agree on every file path, line number, and severity
- If a finding was CORRECTED, update both files
- If a finding was REJECTED in Phase 4, remove it from PLAN.md and change its verdict in findings.json
- Run schema validation one final time before writing SPEC.md

A human-readable PLAN that disagrees with the machine-readable findings.json is a defect in the skill output.
