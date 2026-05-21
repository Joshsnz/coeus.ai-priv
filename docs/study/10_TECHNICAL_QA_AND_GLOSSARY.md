# Technical Q&A and Glossary

## Technical Q&A

### What makes Coeus AI more than a chatbot wrapper?

Coeus has a native C++ desktop GUI, project manager, file context service, retrieval/indexing, memory/summaries, policy-first turn compiler, prompt projection, tool execution, validation/postprocess, structured telemetry, and headless eval runtime. The model is one part of a larger application/runtime architecture.

### Where does source grounding happen?

Source grounding happens through active project state, file-context ingestion, chunking/slicing, inventory/indexing, retrieval controller/hybrid retrieval, context/lattice packing, prompt policy projection, and validation. A claim should be supported by packed project evidence, not merely by model memory.

### What is the context lattice?

The context lattice is the internal context assembly substrate that separates and coordinates context planes: source evidence, inventory, summaries, memory, tool observations, user context, recent mutation state, and policy markers. Its purpose is to prevent all available context from being treated as equal proof.

### What is the turn compiler?

The turn compiler is the policy/runtime layer that converts raw user input into canonical turn state before retrieval and prompting. It builds turn context, turn plan, resolved turn state, target selection, and answer contracts. Downstream layers consume this truth rather than reclassifying the user’s request.

### What is deterministic vs model-driven?

Deterministic:

- slash/host commands
- tool execution
- file scans
- project IO
- telemetry export
- validation checks
- many plan/apply preflight/effects

Model-driven:

- natural language generation
- some intent/semantic interpretation
- plan synthesis or rewrite proposals where applicable
- source-grounded explanation after packed context

The key is that model-driven parts are wrapped in deterministic policy, retrieval, validation, and telemetry.

### How does headless relate to GUI?

Headless is a JSONL protocol surface that reuses the same core runtime concepts without the GUI. It is used for automation/evals. It should not be a second policy engine. The GUI is the product surface; headless is the eval/integration surface.

### What prevents internal artifacts from leaking?

Telemetry and artifacts are model-invisible by default. Tool traces, run traces, activity feed, artifact store, marker truth, and CLE bundles are private/debug surfaces. Only explicitly packed model blocks/source context should be model-visible. Validation/postprocess also checks for internal artifact pollution.

### Why native C++?

The project was designed as a native desktop AI workspace, not a web app. Native C++ gave direct control over rendering, local filesystem/project integration, runtime state, telemetry, and desktop packaging. It also demonstrates systems/desktop engineering skill.

### What are the hardest parts?

Likely hardest areas:

- turn compiler correctness
- semantic vs patch/diff routing
- source evidence vs memory/summary distinction
- context packing under token limits
- tool/plan/apply mutation tracking
- telemetry marker truth
- GUI/runtime state coupling
- headless/GUI runtime drift prevention

### What would you refactor next?

Good answers:

- reduce large GUI orchestration files
- centralize tab/panel identifiers if not already centralized
- simplify high-complexity validation/postprocess files
- strengthen canonical tool names over aliases
- improve performance measurement notes
- keep headless and GUI runtime paths aligned
- continue isolating prompt policy projection from reclassification

## Glossary

### Answer contract

The resolved rules for what kind of answer is allowed for a turn. Used by prompt projection and validation.

### ArtifactStore

Private content-addressed store for debug/run/tool artifacts. Model-invisible by default.

### CLE

Internal context/evidence layer. In practice it includes file context, retrieval, lattice, memory, summaries, telemetry bundles, and related runtime context surfaces.

### Context lattice

Context assembly structure that coordinates different context planes without treating them all as equal source proof.

### File context

The app’s representation of project files/chunks/slices that can be searched, retrieved, summarized, and packed.

### Headless runtime

JSONL stdin/stdout runtime surface used for automated evals/integration. It runs without GUI.

### Marker truth

Structured telemetry/export truth parsed from runtime/prompt/transport markers. Used by private evals to check what actually happened.

### Model-visible

Information deliberately included in the model request/context.

### Model-invisible

Information stored for UI/debug/eval that is not automatically included in the model context.

### Packed truth

The context/evidence actually included in the prompt/request. Availability alone does not mean packed.

### Policy truth

Canonical turn semantics and allowed behavior decided before retrieval/prompting.

### Prompt policy projection

The layer that projects resolved runtime policy into prompt-facing instructions/markers.

### RepoGrounding

Private evaluation suite for repository context, source-grounded QA, negative grounding, multi-turn continuity, route discipline, and artifact suppression.

### ResolvedTurnState

Canonical runtime representation of the current turn after policy/planner/target resolution.

### Tool observation

A selected host tool result that may become model-visible if explicitly packed. Distinct from raw tool trace.

### Tool trace

Private/debug record of tool execution. Model-invisible by default.

### Turn compiler

The policy-first pipeline that compiles user input into canonical turn state, answer contract, retrieval options, prompt policy, validation policy, and telemetry markers.
