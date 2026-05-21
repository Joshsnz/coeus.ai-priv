# Turn Compiler, Policy, and Canonical Truth

## Why this is the most important architecture

The turn compiler is the strongest differentiator in Coeus. It turns raw user input into canonical runtime truth before retrieval, prompt packing, validation, tools, and telemetry.

Without this, the app would be prompt-first:

```text
user message → prompt → model response
```

Coeus aims to be policy-first:

```text
user message
→ turn context
→ turn plan
→ resolved turn state
→ answer contract
→ retrieval/prompt/validator/tool/export projections
→ model or deterministic runtime path
```

This is how the app avoids many common LLM assistant failure modes:

- generic answers with lost repo context
- hallucinated implementation details
- semantic questions misrouted into patch/diff mode
- internal trace/vector storage leaking into answers
- missing-context refusals when the project is already loaded
- deterministic commands bypassing canonical truth

## Key files

```text
core/agent/planner/turn_plan_builder.cpp
core/agent/policy/intent.h
core/agent/policy/request_tag.h
core/agent/policy/turn_context.cpp
core/agent/policy/turn_context_heuristics.cpp
core/agent/policy/turn_context_scoping.cpp
core/agent/policy/turn_target_selection.cpp
core/agent/policy/resolved_turn_state.cpp
core/agent/policy/resolved_turn_state_io.cpp
core/agent/policy/resolved_turn_contract.cpp
core/agent/policy/resolved_turn_extras_builder.cpp
```

## Core concepts

### TurnContext

TurnContext captures what the current user turn appears to be asking for, plus relevant state needed to route it. It is an early semantic/policy representation, not the final prompt.

### TurnPlan

TurnPlan decides how the turn should be handled at a high level: repo QA, command, patch/diff, plan/apply, inventory, conversation recall, missing target, etc. It also shapes retrieval admission and context packing.

### ResolvedTurnState

ResolvedTurnState is canonical runtime truth. Downstream layers should consume this instead of re-guessing user intent. It carries answer family, output mode, target, authority, repo scope, and other resolved values.

### ResolvedTurnExtras

Extras add supporting fields needed by downstream packers/validators/exporters without making the core turn state messy.

### AnswerContract

The answer contract states what kind of answer is allowed. This is important for validation/postprocess and for preventing wrong output modes.

## Simplified pipeline

```text
Raw user turn
→ detect command / host control / repo context / semantic compare / patch intent / inventory / recall / absence query
→ build TurnContext
→ build TurnPlan
→ resolve target and authority
→ build ResolvedTurnState
→ derive AnswerContract
→ project to retrieval, prompt, validator, telemetry
```

## The policy truth doctrine

A core doctrine from the source review:

> Prompting, retrieval, validation, tools, and telemetry should consume resolved policy truth. They should not each independently decide what the user meant.

This prevents drift. For example:

- Prompt builder should not decide that a semantic comparison is a patch request.
- Retrieval should not scan globally if resolved truth says the turn is a host command or missing target.
- Validator should not infer an evidence-table requirement merely from answer kind if the user/eval did not request it.
- Tool execution traces should not become model evidence by default.

## Important answer families / modes

The exact enum names may vary, but the review points to these answer families or modes:

- repo-scoped QA
- semantic comparison
- patch/diff/edit request
- inventory/listing
- no-repo/general chat
- host command
- deterministic command answer
- missing target / impossible request
- conversation recall
- absence/negative grounding

## Why semantic vs patch/diff matters

One real eval issue involved the phrase “implementation path” accidentally triggering patch/diff mode. That shows why turn policy matters. A user can ask a semantic source-role question using implementation-ish language. The policy layer must distinguish:

- “Explain how this is implemented today” = semantic/source QA
- “Change this implementation” = edit/patch request
- “Produce a diff” = patch/diff mode

This is why semantic-to-diff routing discipline is a key benchmark dimension.

## Deterministic commands still need truth

Slash/host commands bypass the normal LLM path, but they should not bypass canonical truth. The review notes that deterministic command turns now get explicit local truth. Inventory commands become inventory truth; ordinary host-control commands become no-repo/host-control truth.

Why this matters:

- exporter/validator can understand what happened
- eval harness can classify the turn
- command turns do not accidentally reuse stale repo truth
- maintenance commands do not falsely clear recent mutation information

## Retrieval gating

Retrieval should be admitted or forbidden by planner/resolved truth.

Examples:

- repo QA → allow source retrieval
- inventory listing → may use inventory truth but not broad QA retrieval
- missing repo target → forbid retrieval/global scan
- host command → deterministic command path, not model retrieval
- conversation recall → continuity/memory, not source proof

This is one of the main controls that makes the system disciplined.

## Target selection

Turn target selection decides what the turn is “about”: active project, selected file, current context, recent mutation, plan target, or no repo target. This matters because retrieval and answer authority depend on target.

## Reviewer answer: what is the turn compiler?

> The turn compiler is the policy/runtime layer that converts a raw user message into canonical turn state before retrieval and prompting. It decides whether the turn is repo QA, semantic comparison, patch/diff, inventory, command, recall, or missing-context, then projects that decision into retrieval options, prompt policy, answer contracts, validation, and telemetry. This reduces drift between what the user asked, what context was packed, and what the assistant is allowed to claim.

## Risk and audit points

Watch for:

- prompt layer reclassifying the turn
- validator inferring requirements from answer kind alone
- retrieval running despite hard forbids
- stale recent mutation state leaking into new turns
- commands bypassing resolved truth
- weak distinction between “source role map” and “implementation patch” language

## Why this helps portfolio explanation

This lets you explain Coeus as an engineered AI runtime:

> Coeus compiles each turn into policy and context contracts before invoking the model. That architecture is what lets it test context retention, source grounding, semantic-vs-diff routing, and internal artifact suppression.
