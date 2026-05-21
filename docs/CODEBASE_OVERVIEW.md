# Coeus AI — Codebase Overview

## Summary

Coeus AI is a native C++ project-aware AI desktop workspace. The codebase contains a custom GUI, project context management, retrieval/context assembly, tool/workflow execution, model/provider integration, validation/post-processing, telemetry, and a headless runtime used for private automated evaluation.

This private review package exists to support code-level understanding without exposing secrets, traces, local project data, or build artifacts.

## Clean metrics

The clean review scan includes native C/C++ application source and headers.

Excluded from the clean scan:

- generated Tree-sitter parser code,
- static assets and fonts,
- build outputs,
- cache directories,
- vendor/external directories,
- environment/secrets files.

| Metric | Value |
|---|---:|
| Files analyzed | 364 |
| Total lines | 181,781 |
| Code lines | 144,110 |
| Reported complexity | 36,717 |
| C++ files | 177 |
| Header files | 187 |

## What the codebase proves

The codebase supports the public claim that Coeus is not just a chatbot wrapper. It contains dedicated subsystems for:

- native desktop GUI,
- project state and project IO,
- agent policy and turn orchestration,
- prompt construction and policy projection,
- retrieval and context assembly,
- context lattice and memory,
- file context and source inventory,
- command/tool execution,
- telemetry and structured export,
- provider/model transport,
- and headless evaluation support.

## Main subsystem map

| Subsystem | Purpose |
|---|---|
| `gui/` | Native GUI, rendering, panels, conversation surfaces, evidence/context display, telemetry UI. |
| `core/agent/` | Turn policy, prompting, runtime orchestration, state, validation, post-processing. |
| `core/cle/` | File context, retrieval, memory, lattice/context assembly, summaries, inventory. |
| `core/tools/` | Tool registry, tool runner, built-ins, planning helpers, connectors. |
| `core/telemetry/` | Run traces, markers, exporters, schemas, activity feed. |
| `core/project/` | Project model, project IO, project actions, project manager. |
| `core/commands/` | Slash/chat command dispatch, command effects, command operations. |
| `core/llm/` | Provider management and model runner abstraction. |
| `headless/` | Private non-GUI runtime surface for automation/evaluation. |

## Important interpretation

The `scc` complexity figure is a relative indicator of branching/control-flow complexity, not a quality score. In Coeus, complexity is expected in orchestration, retrieval, validation, telemetry, GUI state, command/tool routing, and policy logic.

The `scc` COCOMO-style effort estimates are heuristic estimates derived from code size. They are included in raw metrics but should not be presented as actual project cost.
