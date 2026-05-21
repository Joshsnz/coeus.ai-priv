Clean source scope

The clean review scan includes the native C/C++ application source and headers.

Excluded from the clean scan:

generated Tree-sitter parser code
static assets and fonts
build outputs
cache directories
vendor/external directories
environment/secrets files

Clean scan summary:

Metric  Value
Files analyzed  364
Total lines 181,781
Code lines  144,110
Reported complexity 36,717
C++ files   177
Header files    187
Top-level source layout
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
Major subsystems
core/agent

Agent policy, prompting, runtime orchestration, state, and validation.

core/agent/
├── io/
├── planner/
├── policy/
├── prompting/
├── runtime/
├── state/
└── validation/

Important areas:

policy/ — turn context, intent/request tags, resolved turn state, targeting, and canonical answer contracts.
planner/ — turn plan construction.
prompting/ — prompt building, prompt policy projection, attention construction, locked prompts, profile handling, and tiered prompt contracts.
runtime/ — agent flow, run controller, agent request construction, bridge logic, single-shot runner, and plan storage.
state/ — conversation state, ephemeral state, and task store.
validation/ — output validation, answer post-processing, and validation policy.

Representative files:

core/agent/runtime/agent_flow.cpp
core/agent/runtime/run_controller.cpp
core/agent/policy/resolved_turn_state.cpp
core/agent/policy/turn_context.cpp
core/agent/prompting/prompt_builder.cpp
core/agent/prompting/prompt_policy_projection.cpp
core/agent/validation/output_validator.cpp
core/agent/validation/answer_postprocess.cpp
core/cle

Context, retrieval, memory, summary, inventory, and context-lattice infrastructure.

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

Important areas:

file_context/ — file scanning, chunking, file context management, and file-context search.
retrieval/ — BM25, hybrid search, retrieval policy, retrieval detail, and retrieval controller.
lattice/ — context lattice and lattice runtime.
memory/ — vector memory, memory store, memory graph, embed cache, snapshots, and memory seed building.
summary/ — file summaries, gists, meta summaries, summary queues, and summary worker logic.
inventory/ — project inventory state and inventory gist construction.
index/ — graph index builder and store.

Representative files:

core/cle/file_context/file_context_service.cpp
core/cle/file_context/file_context_search.cpp
core/cle/retrieval/retrieval_controller.cpp
core/cle/retrieval/hybrid_search.cpp
core/cle/lattice/lattice_runtime.cpp
core/cle/memory/vector_memory_service.cpp
core/cle/summary/summary_service.cpp
core/cle/inventory/inventory_gist_builder.cpp
core/tools

Tool execution, built-in command tools, planning helpers, toolchains, connectors, and observation storage.

core/tools/
├── builtins/
├── connectors/
└── planning/

Important areas:

builtins/ — built-in file context, retrieval, repo, index, plan, workspace, search, summary, artifact, and tool commands.
planning/ — plan application, preflight, rewrite fallback, target resolution, local plan store, and JSON extraction.
connectors/ — connector registry and secret-store boundary.
root tool files — tool registry, tool runner, toolchain executor, tool observations, and tool specs.

Representative files:

core/tools/tool_runner.cpp
core/tools/tool_registry.cpp
core/tools/toolchain_executor.cpp
core/tools/builtins/builtin_plan.cpp
core/tools/builtins/builtin_fc_ops.cpp
core/tools/builtins/builtin_repo.cpp
core/tools/planning/plan_apply_effects.cpp
core/tools/planning/plan_rewrite_fallback.cpp
core/telemetry

Structured telemetry, run traces, exported bundles, tool traces, and evaluation-facing schemas.

core/telemetry/
└── schema/

Important areas:

run index and run trace
CLE exporter
marker truth export
tool exporter
artifact store
telemetry schemas

Representative files:

core/telemetry/cle_exporter.cpp
core/telemetry/cle_exporter_marker_truth.cpp
core/telemetry/run_trace.cpp
core/telemetry/run_index.cpp
core/telemetry/trace.cpp
core/telemetry/schema/cle_bundle_v1.h
core/telemetry/schema/run_trace_v1.h
core/telemetry/schema/lattice_trace_v1.h
core/project

Project state, project actions, project IO, and project manager.

Representative files:

core/project/project_manager.cpp
core/project/project_io.cpp
core/project/project_actions.cpp
core/project/project_data.cpp
core/commands

Slash/chat command dispatch, command effects, command operations, and retrieval probing.

Representative files:

core/commands/chat_command_dispatcher.cpp
core/commands/chat_command_effects.cpp
core/commands/chat_command_ops.cpp
core/commands/retrieval_prober.cpp
core/llm

Provider management and local/remote model runner abstraction.

Representative files:

core/llm/provider_manager.cpp
core/llm/runners/http_json_runner.cpp
core/llm/runners/local_process_runner.cpp
core/integrations

Editor bridge integration.

Representative files:

core/integrations/editor_bridge/sublime_bridge_client.cpp
core/integrations/editor_bridge/editor_bridge_types.h
gui

Native desktop GUI, rendering, panels, conversation surfaces, context/evidence displays, telemetry UI, and user input.

gui/
├── components/
├── conv/
├── panels/
├── platform/
├── render/
├── services/
├── sidebar/
└── telemetry/

Important areas:

components/ — app bar, conversation output, context output, dashboard, evidence box, file context manager, memory map, profile editor, provider editor, side bar, trace viewer, user input.
conv/ — conversation rendering, syntax, minimap, painter, and fenced block parsing.
panels/ — prompt panel, tools panel, monitor panel, settings panel.
render/ — Skia/SDL/OpenGL render backend and frame scheduler.
telemetry/ — telemetry user interface.
platform/ — Windows-specific window chrome.

Representative files:

gui/gui_main.cpp
gui/ui_framework.cpp
gui/ui_compositor.cpp
gui/components/conversation_output.cpp
gui/components/context_output.cpp
gui/components/evidence_box.cpp
gui/components/user_input.cpp
gui/telemetry/telemetry_ui.cpp
gui/render/render_backend_gl.cpp
headless

Private headless runtime support used for automation and evaluation.

Representative files:

headless/headless_main.cpp
headless/headless_protocol.cpp
headless/headless_runtime_bridge.cpp
headless/headless_session.cpp
Architecture reading path

A reviewer who wants to understand the implementation should start here:

Desktop entry and GUI shell
main.cpp
gui/gui_main.cpp
gui/ui_framework.cpp
gui/ui_compositor.cpp

Conversation and user interaction surfaces
gui/components/user_input.cpp
gui/components/conversation_output.cpp
gui/components/context_output.cpp
gui/components/evidence_box.cpp

Agent runtime
core/agent/runtime/agent_flow.cpp
core/agent/runtime/run_controller.cpp
core/agent/runtime/agent_request.cpp
core/agent/runtime/agentic_runner.cpp

Turn policy and canonical turn state
core/agent/policy/turn_context.cpp
core/agent/planner/turn_plan_builder.cpp
core/agent/policy/resolved_turn_state.cpp
core/agent/policy/resolved_turn_contract.cpp
core/agent/policy/turn_target_selection.cpp

Prompt projection and context packing
core/agent/prompting/prompt_builder.cpp
core/agent/prompting/prompt_policy_projection.cpp
core/agent/prompting/context_prompt_service.cpp
core/agent/prompting/attention_builder.cpp

Project context and retrieval
core/cle/file_context/file_context_service.cpp
core/cle/file_context/file_context_search.cpp
core/cle/retrieval/retrieval_controller.cpp
core/cle/retrieval/hybrid_search.cpp
core/cle/retrieval/bm25_db.cpp

Context lattice and memory
core/cle/lattice/lattice.cpp
core/cle/lattice/lattice_runtime.cpp
core/cle/memory/vector_memory_service.cpp
core/cle/memory/memory_store.cpp
core/cle/memory/memory_graph.cpp

Validation and post-processing
core/agent/validation/output_validator.cpp
core/agent/validation/output_validator_policy.cpp
core/agent/validation/answer_postprocess.cpp

Tools and planning
core/tools/tool_runner.cpp
core/tools/tool_registry.cpp
core/tools/toolchain_executor.cpp
core/tools/builtins/builtin_plan.cpp
core/tools/planning/plan_apply_effects.cpp

Telemetry and headless evaluation support
core/telemetry/cle_exporter.cpp
core/telemetry/cle_exporter_marker_truth.cpp
core/telemetry/run_trace.cpp
headless/headless_runtime_bridge.cpp
headless/headless_session.cpp

