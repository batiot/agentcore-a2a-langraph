---
description: Design review of the current OpenSpec change — Socratic questions, principle violations (SOLID, KISS, DRY, YAGNI), and simpler/cleverer alternatives. Writes findings to review-design.md in the change directory.
---

Perform a design review of the current OpenSpec change.

**IMPORTANT: Review mode is for thinking and critiquing, not implementing.** Do NOT write code or implement features. Read artifacts, reason about the design, and write findings to `review-design.md`.

**Input**: The argument after `/opsx:review-design` is an optional change name (e.g., `/opsx:review-design a2a-haiku-agent`). If omitted, auto-detect from context or prompt the user.

---

## Steps

### 1. Select the change

If a name is provided, use it. Otherwise:
- Infer from conversation context if the user mentioned a change
- Auto-select if only one active change exists
- If ambiguous, run `openspec list --json` and use the **AskUserQuestion tool** to let the user select

Announce: "Reviewing change: <name>"

### 2. Read all design artifacts

Read each available artifact from `openspec/changes/<name>/`:
- `proposal.md` — the *why* and *what*
- `design.md` — the *how* (decisions, architecture)
- `tasks.md` — the implementation plan
- Any specs under `specs/*/spec.md`

Do not skip specs — they often reveal hidden assumptions and coupling.

### 3. Analyze the design

Think like a senior engineer pairing with the author: respectful, specific, grounded in the actual artifacts.

#### 3a. Socratic Questions

Surface assumptions that have not been validated. Frame as genuine open questions, not rhetorical traps. Probe:
- Environment and deployment assumptions
- Protocol and SDK assumptions ("is that actually required, or just convention?")
- Scaling / extension assumptions ("what breaks first if a second agent type is added?")
- Operational assumptions ("who owns token rotation on-call?")

#### 3b. Principle Violations

Evaluate against: **SRP, OCP, LSP, ISP, DIP** (SOLID), **DRY**, **KISS**, **YAGNI**, Separation of Concerns, and Coupling.

For each violation: name the principle, cite the artifact, explain the risk.

#### 3c. Duplication

Look for repeated responsibilities across components, overlapping specs, and config values defined in multiple places.

#### 3d. Alternatives

For each significant concern, propose alternatives rated as **Simpler**, **Cleaner**, or **Cleverer** (flag clever ones honestly). Prefer simpler unless clever provides a clear, disproportionate benefit.

### 4. Write review-design.md

Write output to `openspec/changes/<name>/review-design.md` using this structure:

```markdown
# Design Review: <change-name>

> Generated: <date>

## Socratic Questions
### Q1: <title>
<question>

---

## Principle Analysis
### <Principle> — <OK | Minor concern | Major concern>
<Finding with citation and risk.>

---

## Duplication
### D1: <title>
<Description, location, impact.>

---

## Proposed Alternatives
### Alt-1: <title>
**Addresses**: <concern(s)>
**Type**: Simpler | Cleaner | Cleverer
**Description**: <1–3 sentences>
**Trade-offs**: <what you give up>

---

## Summary
| Area | Severity | Note |
|------|----------|------|
| ... | Low / Medium / High | ... |
```

Severity: **Low** = revisit later, **Medium** = address before shipping, **High** = will cause problems.

### 5. Confirm output

After writing, report:
- Number of Socratic questions raised
- Number of principle concerns (by severity)
- Number of alternatives proposed
- File path written

Do NOT re-paste the full content — just the summary counts.
