# Agent Runtime, Validation, and Postprocess

## Purpose

The runtime is the layer that actually turns resolved policy and packed context into an answer, deterministic command result, tool operation, or plan/apply flow. It should coordinate, not reclassify. The validation/postprocess layer then ensures the answer respects the contract and does not leak private internals.

## Key files

```text
core/agent/runtime/agent_flow.cpp
core/agent/runtime/run_controller.cpp
core/agent/runtime/agent_request.cpp
core/agent/runtime/agentic_runner.cpp
core/agent/runtime/agent_single_shot.cpp
core/agent/runtime/action_request_v1.cpp
core/agent/runtime/plan_store.cpp
core/agent/runtime/sprint_plan_v1.h
core/agent/state/conversation_manager.cpp
core/agent/state/ephemeral_state_store.cpp
core/agent/state/task_store.cpp
core/agent/validation/output_validator.cpp
core/agent/validation/output_validator_policy.cpp
core/agent/validation/answer_postprocess.cpp
```

## AgentFlow

`agent_flow.cpp` appears to be the central orchestration file. It is one of the highest-value files in the codebase. It likely owns the path from user turn to runtime result:

```text
input turn
→ turn context / plan / resolved state
→ retrieval options
→ prompt/request construction
→ deterministic/tool/model path
→ validation/postprocess
→ post-commit state updates
→ telemetry/export
```

AgentFlow should enforce prior decisions, not make every decision itself.

## RunController

RunController appears to manage run-level execution, especially for agentic/plan/tool sequences. It likely coordinates tool execution, action application, plan/apply flows, and post-commit effects.

## Agent request construction

`agent_request.cpp` / `.h` likely construct the provider request from prompt/context and runtime state. This is the boundary between app truth and the LLM transport layer.

## Single-shot model calls

`agent_single_shot.cpp` and `core/llm` runner files indicate the project supports direct model calls when needed. Headless uses `Agent::singleShotChat` only when the turn is not deterministic/no-LLM.

## State stores

### Conversation manager

Owns persistent or in-memory conversation/tail state. This is needed for multi-turn continuity.

### Ephemeral state store

Holds transient state that should not necessarily become persistent source truth.

### Task store

Supports tasks/workflow concepts.

### Plan store

Stores planning or apply-related state. This matters when a turn involves patching, tool application, recent mutation, or multi-step changes.

## Validation

Relevant files:

```text
core/agent/validation/output_validator.cpp
core/agent/validation/output_validator_policy.cpp
core/agent/validation/answer_postprocess.cpp
```

Validation is a key safety/quality layer. It should check for:

- missing-context leaks
- internal artifact pollution
- provider/config errors
- turn contract failures
- compare misroutes
- repo QA grounding failures
- policy blocking
- anchor leaks
- required coverage failures
- evidence-shaped answer contract failures when explicitly required

A recurring lesson from the eval work: do not infer too much purely from answer kind. Evidence-table requirements should be explicit, not assumed just because a turn is repo QA.

## Answer postprocess

Postprocess should clean, normalize, or constrain the final answer. It should not invent new facts. It should enforce the answer contract, suppress internals, and handle routing/mode outputs.

## Deterministic paths

Not every turn needs a model call. Deterministic commands, inventory listings, status operations, or host-control commands can be served by the runtime. But even deterministic turns need canonical truth so telemetry/exporter/validator can classify them.

## Post-commit

After a turn completes, runtime may need to:

- append conversation tail
- update recent mutation state
- update task/plan stores
- emit telemetry
- queue export/finalize
- update GUI state
- record run index/activity feed

Post-commit matters because many bugs happen after answer generation, especially stale state, recent mutation drift, or incorrect eval markers.

## Runtime drift risks

Watch for:

- runtime reclassifying a turn after policy already resolved it
- model/no-model deterministic paths not producing marker truth
- postprocess changing meaning rather than formatting/validating
- recent mutation state being cleared or preserved incorrectly
- headless runtime diverging from GUI runtime
- provider failures being mistaken for answer failures

## Reviewer answer: how does Coeus control answers?

> Coeus does not rely only on prompt wording. The runtime compiles the turn into resolved state and answer contracts, retrieves/packs admitted context, dispatches deterministic or model paths, then validates and postprocesses the result before committing state and telemetry. This creates a control loop around the model rather than leaving the model as the only authority.

## Why this matters for evals

The RepoGrounding eval reports are meaningful because validation/postprocess and telemetry/marker truth can detect specific failure modes:

- “need more context” leakage despite loaded project
- patch/diff misroute on semantic questions
- absent-feature hallucination
- internal trace artifact leakage
- follow-up route/context loss
- missing expected source terms or coverage

## Public/private split

Public:

> The runtime coordinates turn processing, request construction, retrieval/context assembly, answer handling, validation, telemetry, and headless evaluation.

Private:

> Specific files, contracts, marker truth, post-commit behavior, validator policy, and failure modes.
