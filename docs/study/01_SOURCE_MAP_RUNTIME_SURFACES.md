# Source Map and Runtime Surfaces

## Clean source scope

The private review scan includes the native C/C++ application source and headers. It excludes generated Tree-sitter parser code, static assets/fonts, build outputs, cache directories, vendor/external directories, and environment/secrets files.

| Metric | Value |
|---|---:|
| Files analyzed | 364 |
| Total lines | 181,781 |
| Code lines | 144,110 |
| Reported complexity | 36,717 |
| C++ files | 177 |
| Header files | 187 |

This is the number set to use when describing the private codebase. Do not use the raw scan that includes generated parser C code.

## Top-level layout

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

## Entry point and boot path

The GUI application starts from `main.cpp`. The real `main()` is deliberately tiny and delegates to `MyApp::mainAppEntry(argc, argv)`. The important work happens in the application entry function.

The startup sequence is roughly:

1. Initialize executable-relative path handling.
2. Initialize logging.
3. Configure Windows console behavior and UTF-8 console codepage.
4. Initialize libcurl.
5. Set current working directory to the executable/runtime root.
6. Ensure runtime/config folders exist.
7. Load providers and agent profiles.
8. Load persisted project state.
9. Parse basic CLI flags such as fullscreen/window size.
10. Initialize GUI.
11. Run GUI loop.
12. Shutdown task engine.
13. Save project state, providers, and profiles.
14. Shutdown GUI, curl, and logger.

This is the first proof that Coeus is a real compiled C++ desktop application with a runtime lifecycle, not a notebook/script/web shell.

## Core runtime utilities

### Logging

`core/logger.cpp` and `core/logger.h` implement a thread-safe logging system with runtime log level and console sink controls. It supports environment controls such as log level and console target. In debug builds, logging arguments are lazily evaluated. In release builds with `NDEBUG`, many log calls compile away.

Important detail: headless stdout must remain protocol-clean, so diagnostics need to go to stderr or files. Logging sink control is part of that boundary.

### Path utilities

`core/path_utils.cpp` and `core/path_utils.h` are more important than ordinary helpers. They define the runtime filesystem contract: config paths, logs, project state, CLE traces, tool traces, workspace paths, provider/profile files, and summary/artifact roots.

This supports the “local-first project workspace” story. The project has a clear local runtime layout rather than ad hoc output files.

### Task manager

`core/task_manager.cpp` and `core/task_manager.h` provide application task infrastructure. The important concept is explicit lanes for background/UI/IO/CPU/LLM work and safe handoff to the main thread. This is important for a native GUI where background indexing, retrieval, provider calls, and filesystem work must not directly mutate GUI state.

### Token and text helpers

`core/tokenizer.h` provides token budgeting helpers. It is not a full tokenizer engine by itself; it is a budgeting utility that can estimate or use configured functions. `core/utf8.h` provides UTF-8 validation/sanitization helpers. Together they support safe source-text handling and prompt/context budgeting.

## Two runtime surfaces

### GUI surface

The GUI runtime proves Coeus is a native product surface. It includes SDL2 window/event handling, Skia drawing, render backends, panes, frame scheduling, input dock, app bar, sidebars, panels, telemetry UI, and custom Windows chrome.

### Headless surface

The headless runtime is protocol-driven and used for automation/evals. It reads JSONL commands from stdin, emits JSONL protocol events to stdout, and keeps diagnostics on stderr.

This supports the main private-review claim:

> Coeus has a product GUI and a separate automation/eval surface that share runtime concepts rather than being two unrelated applications.

## Major subsystem map

| Subsystem | Main role |
|---|---|
| `core/agent` | turn policy, prompting, runtime orchestration, state, validation |
| `core/cle` | context layer, file context, retrieval, lattice, memory, summaries |
| `core/tools` | host-executed tools, builtins, planning, connectors |
| `core/telemetry` | traces, artifacts, run index, marker truth, eval export |
| `core/project` | project state, project IO, project manager/actions |
| `core/commands` | slash/chat command dispatch and effects |
| `core/llm` | provider management and runner abstraction |
| `gui` | native desktop UI, render backend, panels, conversation surfaces |
| `headless` | JSONL eval/integration runtime |

## Review reading path

If a reviewer wants to understand implementation rather than marketing, start here:

1. `main.cpp`
2. `gui/gui_main.cpp`
3. `gui/ui_framework.cpp`
4. `gui/render/render_backend_gl.cpp`
5. `gui/components/user_input.cpp`
6. `core/agent/runtime/agent_flow.cpp`
7. `core/agent/policy/turn_context.cpp`
8. `core/agent/prompting/prompt_builder.cpp`
9. `core/cle/file_context/file_context_service.cpp`
10. `core/cle/retrieval/retrieval_controller.cpp`
11. `core/cle/lattice/lattice_runtime.cpp`
12. `core/tools/tool_runner.cpp`
13. `core/agent/validation/output_validator.cpp`
14. `core/telemetry/cle_exporter.cpp`
15. `headless/headless_runtime_bridge.cpp`

## Key review message

The codebase is organized around separate runtime concerns: GUI, project state, policy/turn compilation, retrieval/context, prompt packing, provider dispatch, tools, validation, telemetry, and headless evaluation. That separation is the technical proof behind the public claim that Coeus is a project-aware AI workspace rather than a generic chatbot UI.
