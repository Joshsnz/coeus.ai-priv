# Coeus AI — Start Here System Guide

## One-sentence understanding

Coeus AI is a native C++ desktop AI workspace that turns a local software project into an active context surface: it loads project files, indexes and summarizes them, routes user turns through a policy-first runtime, retrieves source evidence, assembles a prompt/context package, validates the output, records telemetry, and can run either through a custom GUI or a headless JSONL eval harness.

## The project is not just a chatbot wrapper

The source review shows that Coeus has several authored subsystems beyond the model call:

- native SDL2/Skia/OpenGL GUI shell
- custom pane compositor, frame scheduler, and render backend abstraction
- project manager and project IO
- file-context ingestion, chunking, indexing, and retrieval
- memory, summaries, inventory gists, and context lattice runtime
- policy-first turn compiler
- prompt builder and prompt policy projection
- deterministic commands and host-executed tools
- plan/apply and rewrite/preflight helpers
- output validation and post-processing
- structured telemetry, marker truth, run traces, tool traces, and private artifacts
- headless JSONL protocol used for automated evals

This is the core answer when a reviewer asks: “What did you engineer beyond calling an LLM?”

## Two runtime surfaces

The codebase has two primary runtime surfaces.

### 1. Native GUI desktop application

The GUI path begins at `main.cpp`, initializes paths, logging, provider/profile configuration, project state, curl, and then enters the GUI lifecycle:

```text
main.cpp
→ MyApp::mainAppEntry()
→ InitGuiMain()
→ RunGuiMain()
→ ShutdownGuiMain()
```

The GUI is a native C++ desktop surface, not a browser shell. It uses SDL2 for window/events, Skia for drawing, CPU/OpenGL rendering paths, custom window chrome, cached pane composition, UI widgets, text editing, rich code rendering, panels, telemetry UI, and project/context surfaces.

### 2. Headless JSONL runtime

The headless path begins at `headless/headless_main.cpp` and exposes a stdin/stdout JSONL protocol. It is mainly for eval/integration, not a separate agent architecture.

```text
headless_main.cpp
→ HeadlessSession
→ HeadlessRuntimeBridge
→ AgentFlow::RunTurn
→ Agent::singleShotChat when needed
→ OutputValidator
→ telemetry/export/finalize
```

The protocol uses stdout for machine-readable JSONL events only, while diagnostics/logging go to stderr. This makes it suitable for automated eval harnesses.

## The strongest architecture idea: policy-first compiled turns

The most important internal architecture is the compiled-turn pipeline. The app does not simply pass the user message into a prompt. Instead, a turn is normalized into canonical runtime truth before retrieval, prompting, validation, and telemetry.

A simplified lifecycle:

```text
User input
→ TurnContext
→ TurnPlan
→ ResolvedTurnExtras
→ TargetSeed
→ ResolvedTurnState
→ AnswerContract
→ RetrievalOptions / PromptPolicyProjection / ValidatorPolicy / TelemetryMarkers
→ Prompt + source/context package
→ Model or deterministic command/tool path
→ OutputValidator + AnswerPostprocess
→ AgentFlow post-commit
→ Telemetry/export/finalize
```

The key doctrine:

- downstream layers should consume resolved/canonical turn truth
- downstream layers should not independently reclassify the user’s intent
- retrieval is admitted or forbidden by turn policy and resolved truth
- source evidence is different from summaries, continuity, tools, and traces
- internal traces/artifacts are useful for debugging but are not model-visible proof by default

## Four “truth planes” to remember

The source review repeatedly surfaced a useful mental model:

### 1. Policy truth

What the app decided the user is asking and what the turn is allowed to do. This lives around turn context, turn planning, resolved turn state, target selection, answer contract, and prompt policy projection.

### 2. Packed truth

What was actually included in the model context/prompt. It is not enough that evidence was available; for the model to rely on it, it must be packed or otherwise deliberately made model-visible.

### 3. Retrieval/source truth

What exists in the project/file surface, what was indexed, and what retrieval found. This is the source-grounding substrate.

### 4. Claim truth

What the assistant is allowed to claim after validation/postprocessing. Claim truth should be supported by the allowed answer contract, packed evidence, and validation policy.

## End-to-end turn lifecycle

A typical repository question flows like this:

1. User enters a question in the GUI or headless protocol.
2. GUI/headless forwards it into the shared runtime path.
3. Agent policy builds turn context and a turn plan.
4. Turn target selection and resolved turn state decide whether the turn is repo-scoped, semantic QA, patch/diff, inventory listing, host command, missing target, conversation recall, etc.
5. Retrieval policy is applied from planner/resolved truth.
6. File context, retrieval, summaries, memory, inventory, or lattice context are packed as appropriate.
7. Prompt policy projection turns canonical runtime truth into prompt-facing instructions/markers.
8. Provider/runner dispatch calls the model if the turn is not deterministic.
9. Output validator and answer postprocess enforce answer-family contracts, prevent invalid modes, and suppress internal artifact leaks.
10. AgentFlow post-commit updates state, recent mutation information, telemetry, and conversation tail.
11. Telemetry/export records the run for private debugging/evals.

## What to study first

If you forget how the system works, start with these concepts:

1. **GUI/headless dual surface** — proves product and eval/runtime surfaces.
2. **Turn compiler** — explains how Coeus avoids prompt-only behavior.
3. **Retrieval/context lattice** — explains project-aware source grounding.
4. **Tools and commands** — explains deterministic host operations.
5. **Telemetry/marker truth** — explains how evals/debugging know what happened.
6. **Validation/postprocess** — explains how user-facing answers are constrained.

## Likely reviewer questions

You should be ready to answer:

- What makes Coeus more than a chatbot wrapper?
- Where does source grounding happen?
- What is the context lattice?
- What is deterministic and what is model-driven?
- How does the headless eval runtime relate to the GUI runtime?
- What prevents internal traces and vector/project storage from leaking to users?
- What prevents semantic questions from being misrouted into patch/diff mode?
- Why native C++ instead of a web app?
- Where are the hardest/riskier parts of the architecture?
