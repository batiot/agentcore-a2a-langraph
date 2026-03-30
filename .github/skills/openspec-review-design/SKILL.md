---
name: openspec-review-design
description: Design review of the current OpenSpec change. Asks Socratic questions, identifies design flaws and principle violations (SOLID, KISS, DRY, YAGNI), and proposes simpler or cleverer alternatives. Outputs a review-design.md file in the change directory.
license: MIT
compatibility: Requires openspec CLI.
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.2.0"
---

Perform a design review of the current OpenSpec change.

**IMPORTANT: Review mode is for thinking and critiquing, not implementing.** You MUST NOT write code or implement features. You read artifacts, reason about the design, and write your findings to `review-design.md`. Stay in reviewer mode throughout.

**Input**: Optionally specify a change name. If omitted, auto-detect from context or list available changes.

---

## Steps

### 1. Select the change

If a name is provided, use it. Otherwise:
- Infer from conversation context
- Auto-select if only one active change exists
- If ambiguous, run `openspec list --json` and use the **AskUserQuestion tool** to let the user select

Announce: "Reviewing change: <name>"

### 2. Read all design artifacts

Read each available artifact from `openspec/changes/<name>/`:
- `proposal.md` — the *why* and *what*
- `design.md` — the *how* (decisions, architecture)
- `tasks.md` — the implementation plan
- Any specs under `specs/*/spec.md`

Do not skip specs — they often reveal assumptions and coupling that are not visible in the design alone.

### 3. Analyze the design

For each concern below, collect findings in memory before writing. Think like a senior engineer pairing with the author: respectful, specific, grounded in the actual artifacts.

#### 3a. Socratic Questions

Surface assumptions that have not been validated. Frame as genuine questions, not rhetorical traps.

Probe areas such as:
- **Assumptions about the environment**: "The design assumes X — what happens if X is not true at deploy time?"
- **Assumptions about the protocol**: "Decision D2 chooses pattern Y — is that actually required by the A2A spec, or is it convention?"
- **Assumptions about scale / future use**: "If a second agent type were added tomorrow, which parts break first?"
- **Assumptions about the operator**: "The pre-generated JWT approach requires manual token rotation — who owns that on-call?"

Keep questions open-ended and genuinely curious. Do not answer them yourself.

#### 3b. Principle Violations

Evaluate the design against:

| Principle | What to look for |
|-----------|-----------------|
| **SRP** (Single Responsibility) | Modules / components doing more than one thing; classes that change for multiple reasons |
| **OCP** (Open/Closed) | Extension points hard-coded, requiring source changes to add new behavior |
| **LSP** (Liskov Substitution) | Subclass contracts that break parent promises |
| **ISP** (Interface Segregation) | Fat interfaces that force clients to depend on methods they don't use |
| **DIP** (Dependency Inversion) | High-level modules directly instantiating low-level modules |
| **DRY** (Don't Repeat Yourself) | Same logic duplicated across tasks, specs, or decisions |
| **KISS** (Keep It Simple) | Unnecessary abstraction layers; complex solutions where a trivial one exists |
| **YAGNI** (You Aren't Gonna Need It) | Features or flexibility built for hypothetical future requirements stated as non-goals |
| **Separation of Concerns** | Config, business logic, and I/O tangled together |
| **Coupling** | Components that cannot be tested or deployed independently |

For each violation found: name the principle, quote or cite the relevant part of the design, explain the risk.

#### 3c. Duplication

Look for:
- Repeated responsibilities across multiple components (e.g., both the executor and the server layer handling the same error case)
- Overlapping specs that describe the same behavior from two angles without a single source of truth
- Config values defined in multiple places (task + spec + design decision)

#### 3d. Artifact Drift

Compare all artifacts against each other to detect inconsistencies introduced by manual edits that were not propagated across the full set.

For each artifact pairing, check:

| Pair | What to look for |
|------|------------------|
| **proposal → tasks** | Scope, goals, or constraints described in the proposal that have no corresponding task; tasks that implement something not mentioned in the proposal |
| **proposal → design** | Decisions in the proposal ("we will use X") that contradict or are absent from the design |
| **design → specs** | Architectural decisions or interfaces described in the design that are not reflected in any spec, or specs that diverge from design decisions |
| **specs → tasks** | Spec requirements (e.g., a security constraint, an API contract) that have no task covering them; tasks referencing spec behavior that no longer exists in the spec |
| **tasks → tasks** | Duplicate tasks, tasks that contradict each other, or missing dependency ordering |

For each drift found: identify the source artifact, the target artifact, quote the diverging content from both sides, and classify the drift:
- **Missing**: content exists in one artifact but is absent from another where it should appear
- **Contradictory**: content in two artifacts actively conflicts
- **Stale**: content in one artifact references an older version of a decision that has since changed elsewhere

#### 3e. Alternatives

For each significant concern or violation, propose one or more alternatives. Rate each alternative:

- **Simpler**: Lower complexity, fewer moving parts, less to maintain
- **Cleaner**: Better separation, more principled
- **Cleverer**: Technically elegant but potentially harder to understand (flag this honestly)

Prefer simpler over clever unless clever provides a clear, disproportionate benefit.

### 4. Write review-design.md

Write the output to `openspec/changes/<name>/review-design.md`.

Structure:

```markdown
# Design Review: <change-name>

> Generated: <date>

## Socratic Questions

<!-- Open questions to validate assumptions. Not rhetorical — genuinely unanswered. -->

### Q1: <short title>
<question body>

### Q2: ...

---

## Principle Analysis

### <Principle Name> — <Verdict: OK | Minor concern | Major concern>
<Finding. Quote or cite the artifact. Explain the risk. Be specific.>

---

## Duplication

### D1: <short title>
<Description of duplication. Where it appears. Why it matters.>

---

## Artifact Drift

### Drift-1: <short title>
**Source**: `<artifact-A>` | **Target**: `<artifact-B>` | **Type**: Missing | Contradictory | Stale
<Quote the relevant content from both artifacts. Explain what is out of sync and why it matters.>

---

## Proposed Alternatives

### Alt-1: <short title>
**Addresses**: <which concern(s) above>
**Type**: Simpler | Cleaner | Cleverer
**Description**: <1–3 sentences on the alternative>
**Trade-offs**: <what you give up>

---

## Summary

| Area | Severity | Short note |
|------|----------|-----------|
| <finding> | Low / Medium / High | <one-liner> |
```

Use severity consistently:
- **Low**: Cosmetic or stylistic, revisit later
- **Medium**: Should be addressed before shipping
- **High**: Likely to cause problems in integration or production

Drift entries also appear in the summary table with type (Missing / Contradictory / Stale) noted in the short note column.

### 5. Confirm output

After writing the file, briefly summarize:
- How many Socratic questions raised
- How many principle concerns found (and at what severity)
- How many artifact drift issues found (and which artifact pairs are affected)
- How many alternatives proposed

Do NOT re-paste the full content — just the summary counts and the file path.
