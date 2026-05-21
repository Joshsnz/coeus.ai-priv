# Coeus AI — Architecture Notes

## Core idea

Coeus AI is structured around a policy-first compiled-turn pipeline. The user turn is normalized into canonical state before retrieval, prompt assembly, validation, telemetry export, and response handling.

This avoids having every layer independently infer what the user asked.

## End-to-end turn lifecycle

```text
User input
→ GUI user input surface
→ command/runtime dispatch
→ turn context extraction
→ turn plan construction
→ resolved turn state
→ answer contract
→ retrieval/context selection
→ context lattice / prompt packing
→ model/provider call or deterministic path
→ validation
→ post-processing
→ GUI response
→ telemetry/export/headless trace
```

## Key architecture seams

| Seam | Purpose |
|---|---|
| GUI ↔ runtime | Keeps presentation separate from turn orchestration. |
| Runtime ↔ policy | Prevents model transport from owning turn meaning. |
| Policy ↔ retrieval | Determines what evidence should be searched without treating retrieval as policy. |
| Retrieval ↔ prompt packing | Distinguishes what exists/searches from what is actually packed. |
| Prompting ↔ model transport | Keeps prompt construction separate from provider execution. |
| Validation ↔ postprocess | Validation enforces contract; postprocess handles presentation cleanup. |
| GUI ↔ headless | Allows product UI and automated evals to share runtime concepts without being the same surface. |

## Truth/state boundaries

| Boundary | Meaning |
|---|---|
| Policy truth | What the turn means and allows. |
| Retrieval truth | What source/project material is indexed or retrieved. |
| Packed truth | What context was actually included in the prompt. |
| Continuity truth | Conversation/history state that may route the turn but is not proof by itself. |
| Claim truth | What the assistant is allowed to assert. |

## GUI surface

The GUI provides the native product experience:

- conversation input/output,
- context/evidence display,
- file context management,
- provider/settings panels,
- telemetry/debug panels,
- memory/context views,
- command palette and tool panels.

## Headless surface

The headless runtime provides non-GUI automation and evaluation:

- JSONL/session protocol,
- runtime bridge,
- scenario execution support,
- private eval harness integration.

This is how public RepoGrounding reports are produced without relying on manual GUI testing.

## Engineering significance

The architecture demonstrates:

- native C++ GUI engineering,
- source/project context management,
- policy-first turn routing,
- retrieval-backed answers,
- deterministic tool/workflow operations,
- structured telemetry,
- validation and output discipline,
- and repeatable headless evaluation.
