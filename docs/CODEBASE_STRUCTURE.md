# Coeus AI — Codebase Structure

This document summarizes the private Coeus AI source tree at a review level. It is intended to help a reviewer understand the major subsystems without reading every file in the raw file manifest.

The full clean source file list should be kept in:

```text
structure/source_file_manifest_clean.txt
```

The clean local codebase metrics should be kept in:

```text
metrics/scc_summary_clean.txt
metrics/scc_by_file_clean.txt
metrics/scc_summary_clean.json
metrics/scc_by_file_clean.json
```

## Top-level source layout

```text
src/
├── main.cpp
├── core/
│   ├── agent/
│   ├── cle/
│   ├── commands/
│   ├── integrations/
│   ├── llm/
│   ├── project/
│   ├── telemetry/
│   └── tools/
├── gui/
└── headless/
```

## `core/agent`

Agent policy, prompting, runtime orchestration, state, and validation.

```text
core/agent/
├── io/
├── planner/
├── policy/
├── prompting/
├── runtime/
├── state/
└── validation/
```

Important areas:

- `policy/` — turn context, intent/request tags, resolved turn state, targeting, and canonical answer contracts.
- `planner/` — turn plan construction.
- `prompting/` — prompt building, prompt policy projection, attention construction, locked prompts, profile handling, and tiered prompt contracts.
- `runtime/` — agent flow, run controller, agent request construction, bridge logic, single-shot runner, and plan storage.
- `state/` — conversation state, ephemeral state, and task store.
- `validation/` — output validation, answer post-processing, and validation policy.

Representative files:

```text
core/agent/runtime/agent_flow.cpp
core/agent/runtime/run_controller.cpp
core/agent/policy/resolved_turn_state.cpp
core/agent/policy/turn_context.cpp
core/agent/prompting/prompt_builder.cpp
core/agent/prompting/prompt_policy_projection.cpp
core/agent/validation/output_validator.cpp
core/agent/validation/answer_postprocess.cpp
```

## `core/cle`

Context, retrieval, memory, summary, inventory, and context-lattice infrastructure.

```text
core/cle/
├── docs/
├── file_context/
├── index/
├── inventory/
├── lattice/
├── memory/
├── prompt_ir/
├── retrieval/
├── summary/
└── user_context/
```

Important areas:

- `file_context/` — file scanning, chunking, file context management, and file-context search.
- `retrieval/` — BM25, hybrid search, retrieval policy, retrieval detail, and retrieval controller.
- `lattice/` — context lattice and lattice runtime.
- `memory/` — vector memory, memory store, memory graph, embed cache, snapshots, and memory seed building.
- `summary/` — file summaries, gists, meta summaries, summary queues, and summary worker logic.
- `inventory/` — project inventory state and inventory gist construction.
- `index/` — graph index builder and store.

Representative files:

```text
core/cle/file_context/file_context_service.cpp
core/cle/file_context/file_context_search.cpp
core/cle/retrieval/retrieval_controller.cpp
core/cle/retrieval/hybrid_search.cpp
core/cle/lattice/lattice_runtime.cpp
core/cle/memory/vector_memory_service.cpp
core/cle/summary/summary_service.cpp
core/cle/inventory/inventory_gist_builder.cpp
```

## `core/tools`

Tool execution, built-in command tools, planning helpers, toolchains, connectors, and observation storage.

```text
core/tools/
├── builtins/
├── connectors/
└── planning/
```

Important areas:

- `builtins/` — built-in file context, retrieval, repo, index, plan, workspace, search, summary, artifact, and tool commands.
- `planning/` — plan application, preflight, rewrite fallback, target resolution, local plan store, and JSON extraction.
- `connectors/` — connector registry and secret-store boundary.
- root tool files — tool registry, tool runner, toolchain executor, tool observations, and tool specs.

Representative files:

```text
core/tools/tool_runner.cpp
core/tools/tool_registry.cpp
core/tools/toolchain_executor.cpp
core/tools/builtins/builtin_plan.cpp
core/tools/builtins/builtin_fc_ops.cpp
core/tools/builtins/builtin_repo.cpp
core/tools/planning/plan_apply_effects.cpp
core/tools/planning/plan_rewrite_fallback.cpp
```

## `core/telemetry`

Structured telemetry, run traces, exported bundles, tool traces, and evaluation-facing schemas.

```text
core/telemetry/
└── schema/
```

Representative files:

```text
core/telemetry/cle_exporter.cpp
core/telemetry/cle_exporter_marker_truth.cpp
core/telemetry/run_trace.cpp
core/telemetry/run_index.cpp
core/telemetry/trace.cpp
core/telemetry/schema/cle_bundle_v1.h
core/telemetry/schema/run_trace_v1.h
core/telemetry/schema/lattice_trace_v1.h
```

## `gui`

Native desktop GUI, rendering, panels, conversation surfaces, context/evidence displays, telemetry UI, and user input.

```text
gui/
├── components/
├── conv/
├── panels/
├── platform/
├── render/
├── services/
├── sidebar/
└── telemetry/
```

Important areas:

- `components/` — app bar, conversation output, context output, dashboard, evidence box, file context manager, memory map, profile editor, provider editor, side bar, trace viewer, user input.
- `conv/` — conversation rendering, syntax, minimap, painter, and fenced block parsing.
- `panels/` — prompt panel, tools panel, monitor panel, settings panel.
- `render/` — Skia/SDL/OpenGL render backend and frame scheduler.
- `telemetry/` — telemetry user interface.
- `platform/` — Windows-specific window chrome.

Representative files:

```text
gui/gui_main.cpp
gui/ui_framework.cpp
gui/ui_compositor.cpp
gui/components/conversation_output.cpp
gui/components/context_output.cpp
gui/components/evidence_box.cpp
gui/components/user_input.cpp
gui/telemetry/telemetry_ui.cpp
gui/render/render_backend_gl.cpp
```

## `headless`

Private headless runtime support used for automation and evaluation.

Representative files:

```text
headless/headless_main.cpp
headless/headless_protocol.cpp
headless/headless_runtime_bridge.cpp
headless/headless_session.cpp
```

## Architecture reading path

A reviewer who wants to understand the implementation should start here:

1. Desktop entry and GUI shell

```text
main.cpp
gui/gui_main.cpp
gui/ui_framework.cpp
gui/ui_compositor.cpp
```

2. Conversation and user interaction surfaces

```text
gui/components/user_input.cpp
gui/components/conversation_output.cpp
gui/components/context_output.cpp
gui/components/evidence_box.cpp
```

3. Agent runtime

```text
core/agent/runtime/agent_flow.cpp
core/agent/runtime/run_controller.cpp
core/agent/runtime/agent_request.cpp
core/agent/runtime/agentic_runner.cpp
```

4. Turn policy and canonical turn state

```text
core/agent/policy/turn_context.cpp
core/agent/planner/turn_plan_builder.cpp
core/agent/policy/resolved_turn_state.cpp
core/agent/policy/resolved_turn_contract.cpp
core/agent/policy/turn_target_selection.cpp
```

5. Prompt projection and context packing

```text
core/agent/prompting/prompt_builder.cpp
core/agent/prompting/prompt_policy_projection.cpp
core/agent/prompting/context_prompt_service.cpp
core/agent/prompting/attention_builder.cpp
```

6. Project context and retrieval

```text
core/cle/file_context/file_context_service.cpp
core/cle/file_context/file_context_search.cpp
core/cle/retrieval/retrieval_controller.cpp
core/cle/retrieval/hybrid_search.cpp
core/cle/retrieval/bm25_db.cpp
```

7. Context lattice and memory

```text
core/cle/lattice/lattice.cpp
core/cle/lattice/lattice_runtime.cpp
core/cle/memory/vector_memory_service.cpp
core/cle/memory/memory_store.cpp
core/cle/memory/memory_graph.cpp
```

8. Validation and post-processing

```text
core/agent/validation/output_validator.cpp
core/agent/validation/output_validator_policy.cpp
core/agent/validation/answer_postprocess.cpp
```

9. Tools and planning

```text
core/tools/tool_runner.cpp
core/tools/tool_registry.cpp
core/tools/toolchain_executor.cpp
core/tools/builtins/builtin_plan.cpp
core/tools/planning/plan_apply_effects.cpp
```

10. Telemetry and headless evaluation support

```text
core/telemetry/cle_exporter.cpp
core/telemetry/cle_exporter_marker_truth.cpp
core/telemetry/run_trace.cpp
headless/headless_runtime_bridge.cpp
headless/headless_session.cpp
```
