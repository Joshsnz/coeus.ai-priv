# Tools, Commands, and Planning

## Purpose

The tools layer is where Coeus executes host-side operations. This is not “the model magically runs tools.” The model may request or describe operations, but the host runtime resolves and executes them.

The key doctrine from the review:

> Tools are host-executed. Tool traces are model-invisible by default. Tool observations only become model-visible through an explicit pack-eligible seam.

## Key files

```text
core/commands/chat_command_dispatcher.cpp
core/commands/chat_command_effects.cpp
core/commands/chat_command_ops.cpp
core/commands/retrieval_prober.cpp
core/tools/tool_registry.cpp
core/tools/tool_runner.cpp
core/tools/tool_spec.h
core/tools/tool_observation_store.cpp
core/tools/toolchain_registry.cpp
core/tools/toolchain_executor.cpp
core/tools/toolchain_spec.cpp
core/tools/non_transcript_tool_api.cpp
core/tools/builtins/*.cpp
core/tools/connectors/connector_registry.cpp
core/tools/connectors/secret_store.cpp
core/tools/planning/*.cpp
```

## Commands vs tools

Commands are user/operator-facing instructions such as slash/chat operations. Tools are host-executed capabilities exposed to runtime/planning.

They overlap, but the boundary matters:

- commands route user intent or host controls
- tools execute deterministic host operations
- tool results may be logged/traced
- only selected observations should be packed into model context

## ToolRegistry

The registry owns tool registration, aliases, built-in seeding, and lookup. It centralizes capability names and compatibility aliases.

Good aspects:

- idempotent builtin registration
- central alias map
- legacy/compatibility aliases
- metadata/spec separation

Risk:

- aliases can hide planner sloppiness if overused
- canonical names should still be preferred

## ToolRunner

ToolRunner executes host handlers and records traces/activity/run-index entries. It should not call the model. It should not automatically make tool output source proof.

Execution creates things like:

- call id
- details ref
- tool trace JSON
- activity feed event
- run-index entry
- artifact refs
- outputs JSON

But the important boundary:

> a tool trace exists for debugging; it is not automatically model-visible evidence.

## ToolObservationStore

This is the explicit seam for tool observations that may become pack-eligible. This is safer than dumping tool traces or raw outputs into prompts.

## Built-in tools

The `builtins/` folder includes tools for:

- artifacts
- core/status
- file context operations
- file context tools/write
- indexing
- inventory
- network connector
- planning
- repo operations
- retrieval
- search
- summary
- toolchains
- workspace

Representative files:

```text
builtin_plan.cpp
builtin_fc.cpp
builtin_fc_ops.cpp
builtin_fc_tools.cpp
builtin_fc_write.cpp
builtin_repo.cpp
builtin_retrieval.cpp
builtin_search.cpp
builtin_summary.cpp
builtin_ws.cpp
```

## File-context and repo operations

File-context tools are especially important because they can modify or refresh project context.

Important concepts:

- root setting
- folder scanning
- embedding/rescan
- write operations
- before/after artifacts
- active hint relations
- working set relations
- auto-ingest handoff

Repo/workspace tools should remain bounded under active roots/workspace boundaries.

## Planning helpers

Relevant files:

```text
plan_target_resolution.cpp
plan_json_extract.cpp
plan_rewrite_fallback.cpp
plan_preflight.cpp
plan_apply_effects.cpp
plan_store_local.cpp
```

These files are higher-risk because planning may involve LLM-derived JSON or rewrite instructions. The review notes meaningful validation/preflight exists around this path.

Important steps in plan/apply workflows:

1. extract JSON from model output
2. normalize/canonicalize tool/action names
3. resolve target paths
4. perform preflight checks
5. apply effects deterministically
6. create before/after artifacts
7. update recent mutation state
8. update run telemetry

## Workspace tools

Workspace operations should be quarantined under a controlled workspace root. They should only become file-context mutation truth if they map back under the active file-context root.

## Connector and secret boundary

`connector_registry` and `secret_store` are private boundary surfaces. Public docs should not expose secret/provider configuration details. Private review can discuss host-owned auth injection and disabled-by-default network access.

## Model-invisible traces

Tool traces are private/debug evidence. They do not automatically authorize user-facing claims.

Model-visible tool information should go through:

```text
tool result → model_blocks / observation store → context packer → prompt
```

## Reviewer answer: how do tools work?

> Tools are declared in a registry, resolved by the host runtime, executed by host handlers, and logged through traces/activity/run-index. The model never directly executes tools. Tool outputs only become model-visible when the runtime explicitly constructs pack-eligible observations. This keeps tool execution deterministic and prevents private tool traces from becoming accidental source proof.

## Risk and audit points

Watch for:

- alias drift hiding wrong canonical tool names
- raw tool outputs becoming prompt context
- workspace writes crossing root boundaries
- plan JSON extraction accepting malformed actions
- rewrite fallback creating broad/unintended edits
- recent mutation bundles being over/under-promoted
- network connector exposing unsafe fetch/auth behavior

## Public/private split

Public:

> Coeus has a deterministic command/tool layer and planning/apply-style workflows.

Private:

> Exact built-in tools, write/apply semantics, preflight behavior, secret boundary, tool observation/trace model, and mutation bundle handling.
