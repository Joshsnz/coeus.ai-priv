# Coeus AI — Technical Q&A Notes

## What makes Coeus different from a chatbot wrapper?

Coeus has dedicated subsystems for project state, file inventory, retrieval, context assembly, policy-first turn routing, tool execution, validation, telemetry, and headless evaluation. The model is only one component in a larger application runtime.

## What is the compiled-turn architecture?

The compiled-turn architecture normalizes the user message into canonical turn state before downstream layers act. Retrieval, prompt building, validation, telemetry, and output should consume this canonical state instead of independently reclassifying the user request.

## Where does source grounding happen?

Source grounding begins with file context and retrieval. Project files are scanned, indexed, retrieved, and packed into prompt context. Validation and output policy then constrain what the assistant may claim.

## What is the context lattice?

The context lattice is the conceptual layer that assembles different context/evidence planes into a prompt-ready form. It helps distinguish what exists in the project, what was retrieved, what was packed, and what the model may claim.

## What is deterministic vs model-driven?

Deterministic:

- turn-policy construction,
- routing decisions,
- command/tool dispatch,
- validation checks,
- telemetry/export,
- project/file indexing mechanics.

Model-driven:

- natural language generation,
- some semantic interpretation,
- answer synthesis using packed context.

## How does headless relate to GUI?

The GUI is the product surface. Headless is the automation/evaluation surface. Headless allows the system to be tested through scenarios without manually driving the GUI.

## Why native C++?

The project is a native desktop workspace with custom rendering, local project management, tooling, telemetry, and performance-sensitive UI/runtime concerns. C++ gives control over native GUI, integration, and local runtime behaviour.

## What are the hardest parts?

- Maintaining canonical turn truth across many layers.
- Keeping source evidence distinct from summaries and continuity.
- Avoiding semantic-to-diff misrouting.
- Keeping retrieval surfaces coherent.
- Preventing internal artifact leakage.
- Coordinating GUI, runtime, tools, telemetry, and headless evaluation.

## What would be refactored next?

Likely areas:

- reducing complexity hotspots,
- simplifying GUI component boundaries,
- tightening policy/retrieval/validator seams,
- improving performance measurement,
- expanding repeatable eval coverage,
- and hardening private review docs and source packaging.
