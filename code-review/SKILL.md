---
name: code-review
description: Evidence-first review of a PR, branch, commit range, or working-tree diff. Pins the comparison base, reconstructs the originating spec and repository standards, reads tests first, runs independent review passes for Spec/Correctness, Standards/Maintainability, and Security/Performance/Reliability, then verifies and severity-ranks only actionable findings. Use when the user asks to review code, a PR, a branch, work since a ref, or whether a change is ready to merge.
---

# Evidence-First Code Review

Act as a Staff-level reviewer. The job is to find merge-relevant defects with evidence, not to produce a long list of plausible concerns.

This skill is **read-only by default**. Do not edit files, post review comments, approve a PR, or merge anything unless the user explicitly asks for that action after seeing the review.

## Core model

Keep these concerns independent so one cannot hide another:

1. **Spec and Correctness** — does the change implement the requested behavior, including edge cases and failure paths?
2. **Standards and Maintainability** — does it follow repository rules and preserve readable architecture?
3. **Security, Performance, and Reliability** — does it introduce boundary, resource, concurrency, or scaling risk?
4. **Verification** — do tests and executed checks support the claims?

A clean implementation of the wrong requirement still fails. A correct implementation that violates critical repository constraints still fails.

## Process

### 1. Pin the review scope

Resolve the review target in this order:

1. A PR URL or number supplied by the user: use the PR base and head.
2. A fixed point supplied by the user: commit, branch, tag, or expression such as `HEAD~5`.
3. The current branch's PR base, if one exists.
4. The repository default branch, normally `origin/main` or `origin/master`.

State any inferred base explicitly. Ask only when multiple plausible bases would materially change the review.

For branch review, prefer merge-base semantics:

```bash
git rev-parse <base>
git diff --stat <base>...HEAD
git diff <base>...HEAD
git log <base>..HEAD --oneline
```

For uncommitted work, also inspect:

```bash
git diff
git diff --cached
```

Fail early when the ref does not resolve or the diff is empty. Record:

- base and head;
- commits included;
- changed files;
- generated, vendored, lockfile, migration, and binary files;
- whether the review covers committed, staged, or unstaged changes.

### 2. Reconstruct intent

Find the originating specification in this order:

1. PR title and body.
2. Issue references in PR text or commit messages.
3. A user-supplied task, plan, or spec path.
4. Matching files under `docs/`, `specs/`, `.claude/plans/`, `.scratch/`, or project handoff files.
5. Tests that clearly encode intended behavior.

If no authoritative spec exists, state `No authoritative spec found`. Do not invent requirements. Review correctness against the change's stated intent and existing public behavior.

Find repository standards, including the most local rule file that applies to each changed path:

- `AGENTS.md`, `CLAUDE.md`, nested agent instructions;
- `CONTRIBUTING.md`, coding standards, style guides;
- architecture documents and module READMEs;
- lint, type-check, formatting, test, and CI configuration.

Repository rules override generic heuristics.

### 3. Read tests before implementation

Read changed and adjacent tests first. Determine:

- what behavior the author believes changed;
- which boundaries, error paths, and regressions are covered;
- whether assertions prove behavior rather than merely execute code;
- whether mocks bypass the failure mode;
- whether tests would fail on the pre-change implementation;
- which important behavior remains untested.

Then inspect implementation and surrounding call sites. Do not review isolated hunks without enough context to understand control flow, ownership, and invariants.

### 4. Run independent review passes

Run the passes in parallel when sub-agents are available. Give every pass the exact diff command, commit list, relevant source files, and a strict word budget. Do not let one pass's conclusions prime the others.

#### Pass A — Spec and Correctness

Check:

- missing, partial, or misinterpreted requirements;
- behavior outside the requested scope;
- null, empty, boundary, overflow, encoding, and error cases;
- state transitions, retries, idempotency, cleanup, and rollback;
- races, ordering errors, stale state, and off-by-one defects;
- API compatibility and data/schema migration behavior;
- tests that do not actually prove the claimed behavior.

Every finding must quote or identify the relevant requirement when one exists.

#### Pass B — Standards and Maintainability

Check documented repository rules first, then:

- readability and naming;
- control-flow complexity;
- module boundaries and dependency direction;
- unjustified new patterns or abstractions;
- coupling, circular dependencies, and misplaced responsibility;
- maintainability of public interfaces and configuration.

Use these code smells only as labelled judgement calls, never as automatic violations:

- Mysterious Name
- Duplicated Code
- Feature Envy
- Data Clumps
- Primitive Obsession
- Repeated Switches
- Shotgun Surgery
- Divergent Change
- Speculative Generality
- Message Chains
- Middle Man
- Refused Bequest

Suppress anything the repository explicitly permits or tooling already enforces.

#### Pass C — Security, Performance, and Reliability

Inspect only dimensions relevant to the diff.

Security:
- input validation and output encoding at trust boundaries;
- authentication and authorization;
- injection, path traversal, unsafe deserialization, SSRF, and command execution;
- secrets in code, logs, fixtures, or generated artifacts;
- dependency and supply-chain exposure.

Performance:
- asymptotic regressions, N+1 work, repeated I/O, and unnecessary copies;
- unbounded loops, queues, fetching, buffering, or allocation;
- blocking work on latency-sensitive or async paths;
- missing pagination, batching, backpressure, caching, or release of resources;
- GPU/CPU synchronization, memory traffic, kernel launch, or storage amplification where applicable.

Reliability:
- timeouts, cancellation, retries, partial failure, and resource leaks;
- concurrency hazards and lock ordering;
- corruption, silent wrong results, durability, and recovery behavior;
- observability needed to diagnose a new failure mode.

Do not call a theoretical concern a defect without a concrete triggering path.

### 5. Verify candidate findings

The lead reviewer must independently validate every candidate before reporting it.

For each candidate:

1. Read the complete function and relevant callers.
2. Confirm the issue is introduced or made materially worse by this diff.
3. Identify a concrete input, state, or execution path that triggers it.
4. Run the narrowest useful test, build, static check, or reproducer when feasible.
5. Downgrade uncertainty rather than guessing.

Discard findings that are:

- purely stylistic and formatter/linter-enforced;
- pre-existing and unaffected by the change;
- unsupported by an exact location or failure path;
- speculative security or performance claims without a plausible trigger;
- duplicate manifestations of the same root cause.

A `Critical` or `Important` finding requires strong evidence. Mark unresolved uncertainty as `Investigate`, not as a confirmed blocker.

### 6. Assign severity and verdict

**Critical** — Must fix before merge. Examples: exploitable vulnerability, data loss or corruption, silent wrong result, broken core behavior, unrecoverable migration, or a severe availability failure.

**Important** — Should fix before merge. Examples: reproducible correctness gap, missing validation on a meaningful boundary, broken error handling, substantial regression, unsafe architecture change, or missing tests for newly introduced core behavior.

**Suggestion** — Non-blocking improvement with a clear benefit.

**Investigate** — Plausible concern that could not be verified with available evidence. State the exact check needed to resolve it.

Verdict:

- `REQUEST CHANGES` when any confirmed Critical or Important finding exists.
- `COMMENT` when only Suggestions or Investigate items exist.
- `APPROVE` when no merge-blocking issue is found and verification is adequate.

Never approve merely because automated checks pass. Never request changes merely because the diff is unfamiliar.

## Output format

Put findings first, ordered by severity **within each review axis**. Keep the axes visible; do not collapse a Spec failure into a style score.

```markdown
## Review Summary

**Verdict:** APPROVE | COMMENT | REQUEST CHANGES
**Scope:** <base>...<head>, <commits/files>
**Spec sources:** <paths, issue/PR, or none>
**Standards sources:** <paths>
**Overview:** <one or two sentences>

### Critical
- **[Spec/Correctness | Standards/Maintainability | Security/Performance/Reliability] `path:line` — Title**
  - Evidence: <requirement, code path, or reproducer>
  - Impact: <what fails and for whom>
  - Fix: <specific minimal repair>
  - Verification: <test or check that should prove the repair>

### Important
...

### Suggestions
...

### Investigate
...

## Axis Results
- Spec and Correctness: <pass/fail/not fully assessable; worst issue>
- Standards and Maintainability: <pass/fail; worst issue>
- Security, Performance, and Reliability: <pass/fail/not applicable; worst issue>

## Verification Story
- Tests reviewed: <what was inspected>
- Tests run: <commands and result, or not run with reason>
- Build/type/lint: <commands and result>
- Security/performance checks: <what was actually checked>
- Unverified assumptions: <explicit list>

## Confirmed Strengths
- <at least one specific property supported by the diff, when present>
```

Each Critical and Important finding must include an actionable fix. Use exact `path:line` references. Quote only the minimum code or spec text needed.

If there are no findings, say so directly. Do not manufacture praise or nits to make the review look busy.

## Review rules

1. Findings are the primary output; summaries come second.
2. Review the change, not the author's presumed intent or ability.
3. Prefer one root-cause finding over several symptom-level duplicates.
4. State uncertainty and evidence limits explicitly.
5. Do not trust agent summaries, PR descriptions, or test claims without checking.
6. Do not expand scope into unrelated legacy cleanup.
7. Do not modify code or GitHub state unless the user separately authorizes it.
8. Keep the report concise enough that an author can act on it.

## Attribution

This skill is an original synthesis derived from:

- Matt Pocock's `code-review` skill: fixed-point diffing, independent Spec/Standards axes, and smell-baseline review.
- Addy Osmani's `code-reviewer` agent: five-dimension review, severity categories, tests-first reading, actionable fixes, and verification reporting.

See the repository's `THIRD_PARTY_NOTICES.md` for the corresponding MIT license notices.
