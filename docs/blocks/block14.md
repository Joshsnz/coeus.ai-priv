Block 14 — Analysis and Summary

Block 14 is the headless execution bridge: it lets the same CLE runtime run from stdin/stdout JSONL without the GUI. It is mainly an eval/integration surface, not a separate agent architecture. The important design intent is that headless should reuse canonical project, tool, runtime, policy, validation, and export seams instead of creating a second semantic path.

1. Headless mode runs as a JSONL process

headless_main.cpp is intentionally simple:

stdin  = JSONL commands
stdout = JSONL protocol events only
stderr = logs / diagnostics

That stdout/stderr separation is critical. Eval harnesses usually parse stdout as a machine protocol, so any diagnostic text on stdout would corrupt the harness. This file correctly pushes logging to stderr during bootstrap.

Startup flow:

BootstrapHeadlessConfig()
  -> set logging sink to stderr
  -> load providers
  -> load profiles
  -> log active provider/profile diagnostics to stderr
  -> create HeadlessSession
  -> emit ready event
  -> read JSONL commands from stdin
  -> parse command
  -> dispatch via session.HandleCommandStreaming
  -> emit events to stdout

The ready event uses:

version = "headless_v2"

That matches the eval logs we’ve been seeing.

2. Protocol model

The protocol is deliberately small and stable.

Commands:

hello
new_project
set_root
bootstrap_repo
send_turn
get_state
shutdown

Events:

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

Errors are structured:

InvalidJson
UnknownCommand
MissingField
InvalidField
InvalidState
ProjectNotInitialized
RootNotSet
TurnAlreadyRunning
TurnExecutionFailed
BootstrapFailed
InternalError

This is exactly what the eval harness needs: deterministic request/response shape, explicit request IDs, and no GUI dependency.

3. Session layer: command gate and streaming wrapper

headless_session.* is the protocol/session coordinator. It owns:

HeadlessRuntimeBridge bridge_;
bool turn_in_flight_;

Its main jobs:

- reject commands before project/root exists
- prevent concurrent bootstrap/turn execution
- emit started events before long-running operations
- convert bridge results into protocol events
- convert exceptions into structured error events

The streaming behavior matters. send_turn emits turn_started before AgentFlow or model execution begins. That prevents the eval harness from seeing silence during a slow model call.

This is a good separation:

headless_protocol = parse/serialize schema
headless_session  = stateful command protocol
runtime_bridge    = core runtime seam
headless_main     = process IO loop
4. Runtime bridge: not a second policy engine

headless_runtime_bridge.h explicitly states the intended rule:

This is an operator/transport bridge, not a second policy/runtime engine.

That is the most important line in this block.

The bridge calls canonical CLE seams:

ProjectManager
ToolRunner
fc.set_root_v1
fc.scan_folder_async_v1
AgentFlow::RunTurn
Agent::singleShotChat
OutputValidator::ValidateAndPostprocess
AgentFlow::ApplyPostCommit
OutputValidator::EnqueueFinalizeExport
ProjectActions / ProjectManager persistence

So headless is not supposed to invent separate routing, retrieval, policy, validation, or postprocess behavior.

5. Project/session model

Headless keeps one active project/thread inside the bridge:

active_project_id_
active_thread_id_
project_name_

new_project creates a normal project via ProjectManager::createProject, resolves the active conversation UUID, and loads the thread tail if available.

get_state reports:

project_created
root_set
integration_stub
project_id
thread_id
project_name
root_path
project_folder

This gives the harness enough state to assert the session is initialized and rooted before sending turns.

Important: this is a single-session mutable bridge. It is not a multi-project/multi-thread protocol yet. That is fine for eval harnesses.

6. Root/bootstrap path

set_root is implemented through the built-in tool:

fc.set_root_v1

That is good. It means headless uses the same tool substrate as the GUI/app path for file-context root mutation.

bootstrap_repo then runs:

fc.scan_folder_async_v1

with default scan extensions including C/C++, Python, JS/TS, JSON/YAML, Markdown, HTML/CSS, SQL, shell scripts, Java/Kotlin/Rust/Go/Swift/PHP/Ruby/Lua, etc.

Then it waits for quiescence:

WaitForFileContextQuiescence(...)
  timeout = 8000ms
  poll = 50ms
  stable_samples_required = 3

It only succeeds if:

saw_change = true
ingest_idle = true
pending_cleared = true
stabilized = true

This is important for eval correctness. Without this, the harness could send the first question before file context has actually stabilized.

One detail: embed_tool is currently skipped with the message:

skipped: fc.scan_folder_async_v1 is the authoritative full bootstrap ingest path

So public eval reports should not imply a separate embed command ran during bootstrap. The scan path is the authoritative bootstrap path here.

7. Turn execution path

send_turn does the following:

1. optionally set root from root_override
2. validate non-empty turn text
3. ensure thread tail loaded
4. call AgentFlow::RunTurn(...)
5. if deterministic/no-LLM, use rr.served_text as raw_text
6. otherwise call Agent::singleShotChat(rr.request)
7. build PostprocessContext
8. run OutputValidator::ValidateAndPostprocess
9. apply AgentFlow::ApplyPostCommit
10. enqueue finalize export
11. append committed user/assistant turn to thread tail
12. return TurnFinishedEvent

The most important detail:

Stage1 export is owned by AgentFlow.
Headless bridge only performs finalize/postprocess export.

That is good. It avoids duplicate Stage1 export ownership and keeps the turn bundle pipeline aligned with normal runtime.

The returned turn_finished event exposes:

final_text
request_tag
turn_tier
answer_kind
effective_output_mode
canonical_target_file
markers
artifact_refs
post_commit

That is the public/eval-facing summary of the turn.

8. Relation to eval harness

The eval harness likely treats headless as a subprocess:

start headless exe
wait for ready
send new_project
send set_root / bootstrap_repo
send send_turn
read turn_started / turn_finished
score final_text + metadata
inspect artifacts/logs separately if needed

Public eval reports are mainly backed by the headless protocol outputs, especially:

bootstrap_finished:
  items_total
  items_included
  items_pending
  scan_tool
  scan_message

turn_finished:
  final_text
  answer_kind
  effective_output_mode
  canonical_target_file
  markers
  artifact_refs
  post_commit

Private eval/debug can additionally inspect:

.cle_traces turn bundles
retrieval debug artifacts
lattice trace artifacts
packed prompt artifacts
raw/final output artifacts
stderr logs
marker truth export
post_commit metadata

So the split is:

public report = protocol-visible behavior and final answer
private debug = internal artifacts, prompt markers, packed prompt, traces, raw outputs
9. What public eval reports are backed by

A public eval report should be safe to back with:

- scenario id
- command sequence
- ready/bootstrap/turn protocol events
- final_text
- answer_kind
- effective_output_mode
- canonical_target_file
- high-level pass/fail result
- coarse file counts from bootstrap
- sanitized marker summaries if intentionally exposed

It should not be backed by dumping:

- packed_prompt_markdown
- raw model output before validator/postprocess
- full prompt markers
- provider API key details
- private project folder absolute paths unless intentionally local-only
- full retrieval debug JSON
- internal artifact paths
- exact tool args/outputs if they may contain private data
10. What private headless code should not be shown

The private pieces that should generally stay out of public reports:

provider config:
  provider paths
  base URLs
  apiKeyName
  apiKeyValuePresent

runtime internals:
  full prompt markers
  request tags if they encode internal routing
  packed prompt markdown
  retrieval debug JSON
  lattice trace JSON
  raw model output
  postprocess mutation internals
  trace bundle filesystem paths

local environment:
  absolute project folder paths
  Windows usernames / storage paths
  provider/profile config paths
  local model/process details

eval internals:
  exact harness fixtures/workspaces
  private scenario implementation details
  hidden expected answers
  scoring heuristics

Public reports can cite that the run used headless mode and passed a scenario, but should avoid showing the machinery that would leak prompt construction, private file paths, or hidden scoring.

11. Main drift risks
A. Headless appends committed turns itself

The bridge manually calls AppendCommittedTurnToTail(...) after validation/postprocess. This is necessary because it is not using the GUI conversation manager, but it is a potential drift point.

Risk:

GUI commit path and headless commit path diverge.

Mitigation:

Keep this function tiny.
Do not add policy/retrieval/validation behavior here.
Prefer extracting a shared commit helper later if drift appears.
B. Headless manually invokes singleShotChat

AgentFlow::RunTurn prepares the request, then headless calls Agent::singleShotChat(rr.request) when not served by deterministic path.

This is okay if GUI/runtime does the same. But if the normal runtime later wraps model execution with additional tracing, cancellation, streaming, tool loops, or retry logic, headless can drift.

Recommended watchpoint:

Ensure headless model execution path remains equivalent to the primary non-headless execution path.
C. Bootstrap timeout can create flaky evals

WaitForFileContextQuiescence uses an 8-second timeout. On large fixtures, slow disks, or debug builds, that could cause false failures.

Possible improvement:

Make timeout configurable from command JSON or environment for eval harnesses.
D. Public markers may expose too much

turn_finished returns the full marker map from rr.prompt_markers, plus headless markers. This is very useful for private evals, but too much for public reports.

Recommended distinction:

Protocol can return markers to harness.
Public report generator should redact/filter them.
12. Architecture verdict

Block 14 is mostly aligned with the CLE doctrine.

It correctly makes headless a thin transport/runtime bridge, not a second agent. It uses canonical tools for root/bootstrap, canonical AgentFlow for turn setup, canonical model execution, canonical validator/postprocess, canonical post-commit mutation, and canonical exporter finalize.

The strongest pieces are:

- stdout reserved for JSONL protocol only
- stderr used for diagnostics
- explicit ready/start/finish/error events
- single in-flight guard
- bootstrap waits for file context quiescence
- Stage1 export remains owned by AgentFlow
- headless exposes final answer metadata without requiring GUI

The main thing to protect going forward:

Headless must stay a protocol bridge.
It must not become a second runtime, second commit path, second retrieval path, or second policy interpreter.
Carry-forward summary
Block 14 defines the headless JSONL execution surface. headless_main boots provider/profile config, forces diagnostics to stderr, reserves stdout for protocol JSONL, emits ready=headless_v2, then reads commands from stdin and emits structured events. headless_protocol defines the small command/event schema: hello, new_project, set_root, bootstrap_repo, send_turn, get_state, shutdown; and ready/ack/project/root/bootstrap/turn/state/shutdown/error events. headless_session owns protocol state, rejects invalid command ordering, prevents concurrent turns, and emits turn_started/bootstrap_started before long operations. headless_runtime_bridge is the important seam: it is explicitly a transport bridge, not a second policy/runtime engine. It creates projects via ProjectManager, sets root through fc.set_root_v1, bootstraps via fc.scan_folder_async_v1, waits for file-context quiescence, runs turns through AgentFlow::RunTurn, calls Agent::singleShotChat only when not deterministic/no-LLM, validates/postprocesses through OutputValidator, applies AgentFlow post-commit, enqueues finalize export, and appends the committed turn to the thread tail. Public eval reports should be backed by headless protocol events and final turn metadata. Private eval/debug may inspect CLE traces, packed prompts, raw/final outputs, retrieval/lattice traces, markers, and stderr logs, but those internals should not be shown in public reports unless deliberately redacted.
Highest-value follow-up

Audit parity between the GUI commit/model execution path and HeadlessRuntimeBridge::SendTurn. That is the main place headless could accidentally become a parallel runtime instead of a faithful eval transport.