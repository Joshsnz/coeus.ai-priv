# Telemetry, Evals, and Headless Runtime

## Why this matters

Telemetry and headless runtime are what make Coeus testable and debuggable. They provide the private evidence behind the public RepoGrounding reports.

The key doctrine:

> Telemetry records what happened. It is not model authority by default.

## Telemetry files

```text
core/telemetry/activity_feed.cpp
core/telemetry/artifact_store.cpp
core/telemetry/run_trace.cpp
core/telemetry/run_index.cpp
core/telemetry/tool_exporter.cpp
core/telemetry/trace.cpp
core/telemetry/cle_exporter.cpp
core/telemetry/cle_exporter_marker_truth.cpp
core/telemetry/schema/*.h
```

## ActivityFeed

ActivityFeed is a UI-visible, model-invisible progress stream. It writes JSONL and keeps a bounded in-memory ring per thread. It is like a terminal/status feed for the user/operator.

Important:

- useful for UI progress
- useful for debugging
- not source proof
- not model-visible by default

## ArtifactStore

ArtifactStore is a thread-scoped, content-addressed private artifact helper. It stores payloads as exact refs and rejects unsafe filenames/traversal.

Key doctrine:

- artifacts are model-invisible by default
- tools return refs/metadata
- no directory scanning
- prompt packing must explicitly admit anything model-visible

This is a major privacy/debug boundary.

## RunTrace

RunTrace records agentic/Tier-4 execution spines: planner calls, actions, caps, final refs/digests, termination reason, and timing. It is a replay/debug spine.

It records what happened, not what the model may claim.

## RunIndex

RunIndex is a thread-local catalog of runs: tool runs, chains, agentic runs. It lets the UI display details without directory scanning.

## ToolExporter and tool traces

ToolExporter writes tool traces, caps inline args/outputs, offloads large payloads to artifacts, and preserves the model-invisible boundary. Tool traces are debug records, not prompt context.

## ToolCall and ToolResult schemas

These schemas separate:

- invocation identity
- replay digests
- outputs JSON
- UI summaries
- artifacts
- model blocks

Only model blocks should be pack-eligible.

## CLE exporter

CLE exporter writes per-turn bundles and thread manifests. It is the bridge between runtime facts and private eval/debug artifacts.

It helps compare:

- what policy thought the turn was
- what got packed
- what validator saw
- what runtime executed
- what final answer was produced

## Marker truth

`cle_exporter_marker_truth` is one of the most important eval-support files. It parses prompt/transport/runtime markers into structured truth categories such as:

- policy truth
- packed truth
- validator truth
- runtime truth
- recent mutation truth
- semantic truth
- packed plane disambiguation

Why this matters:

- prevents legacy/ambiguous plane markers from substituting for strict runtime truth
- lets evals detect whether a turn actually packed the right source/context
- helps detect route loss, diff misroutes, missing context leaks, artifact leaks, and false claims

## Headless runtime

Headless files:

```text
headless/headless_main.cpp
headless/headless_protocol.cpp
headless/headless_session.cpp
headless/headless_runtime_bridge.cpp
```

Headless mode runs as a JSONL process:

```text
stdin  = JSONL commands
stdout = JSONL protocol events only
stderr = logs / diagnostics
```

This stdout/stderr separation is critical for eval harnesses.

## Headless protocol

Commands include:

```text
hello
new_project
set_root
bootstrap_repo
send_turn
get_state
shutdown
```

Events include:

```text
ready
ack
project_created
root_set
bootstrap_finished
turn_started
turn_finished
state
shutdown_complete
error
```

Errors are structured: invalid JSON, unknown command, missing field, invalid state, project not initialized, root not set, turn already running, turn execution failed, bootstrap failed, internal error.

## Headless runtime bridge

The bridge is explicitly a transport bridge, not a second policy engine. It should reuse the same runtime path as GUI:

```text
set project/root
→ bootstrap file context
→ wait for quiescence
→ AgentFlow::RunTurn
→ Agent::singleShotChat if needed
→ OutputValidator::ValidateAndPostprocess
→ AgentFlow::ApplyPostCommit
→ finalize export
→ append committed turn
→ emit turn_finished
```

## Public eval reports

The public RepoGrounding reports are backed by this private headless/runtime/eval infrastructure. Publicly, we show scorecards and summaries. Privately, we can inspect:

- protocol events
- packed prompts
- raw/final outputs
- retrieval/lattice traces
- marker truth
- run traces
- stderr diagnostics

## Internal artifact suppression

Telemetry and artifacts are powerful, but they must not leak to user answers. The architecture’s boundary is:

- telemetry/artifacts/debug traces are private by default
- model-visible evidence must be deliberately packed
- public reports should summarize behavior without exposing raw private trace machinery

## Reviewer answer: how are eval reports generated?

> Coeus has a headless JSONL runtime that drives the same agent/runtime concepts without the GUI. The eval harness creates a project, sets a root, bootstraps repository context, sends turns, receives structured events, and inspects private telemetry/export bundles. Public reports summarize whether source grounding, route discipline, context retention, negative grounding, and artifact suppression passed.

## Risk and audit points

Watch for:

- GUI/runtime and headless runtime drift
- telemetry aliases leaking private files
- artifact refs exposed to user-facing answers
- trace helpers becoming evidence authority
- eval relying on markers that are too weak or ambiguous
- stdout contamination in headless mode

## Public/private split

Public:

> private headless evaluation harness and structured telemetry used to generate RepoGrounding reports.

Private:

> headless protocol, marker truth, trace schemas, run bundles, retrieval/lattice debug snapshots, exact eval marker logic.
