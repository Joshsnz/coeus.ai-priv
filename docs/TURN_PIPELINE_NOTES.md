# Coeus AI — Turn Pipeline Notes

## Purpose

The turn pipeline is the core architecture that makes Coeus more than a prompt wrapper. It converts a raw user message into canonical turn state before retrieval, prompt construction, validation, and output.

## Conceptual pipeline

```text
Raw user message
→ TurnContext
→ TurnPlan
→ ResolvedTurnExtras
→ TargetSeed
→ ResolvedTurnState
→ AnswerContract
→ retrieval / prompt / validation / telemetry projection
```

## Important responsibilities

### Turn context

Captures the scoped user input and relevant cheap signals:

- explicit file mentions,
- current file or pinned focus,
- recent mutation state,
- working set,
- request hints,
- inventory references,
- lightweight feature extraction.

### Turn plan

Defines high-level plan/routing intent:

- repository question vs general chat,
- source-grounded answer vs patch/diff request,
- retrieval permission,
- output mode,
- command or tool needs.

### Resolved turn state

Represents the canonical post-seed state that downstream layers should consume.

This is the layer that should decide:

- answer kind,
- effective output mode,
- evidence requirements,
- diff requirements,
- whether needs-more-context is allowed,
- canonical target file,
- target strength and source.

### Answer contract

Controls what the assistant is allowed to output:

- direct answer,
- source-grounded answer,
- patch/diff,
- needs-more-context,
- impossible diff request,
- artifact suppression,
- evidence requirements.

## Why this matters

Without a compiled turn pipeline, every downstream layer can drift:

- prompt builder may reinterpret the user,
- retrieval may search the wrong surface,
- validator may repair semantics after the fact,
- postprocess may hide underlying routing problems,
- telemetry may export inconsistent truth.

The desired design is:

```text
Classify once upstream.
Consume canonical truth downstream.
Validate against the contract.
Do not silently reclassify later.
```

## Review questions this answers

- Where is turn meaning derived?
- How does the app decide direct answer vs retrieval vs diff?
- How does it avoid semantic repo questions being misrouted into patch mode?
- How does it control needs-more-context?
- What prevents summaries/continuity from becoming proof?
