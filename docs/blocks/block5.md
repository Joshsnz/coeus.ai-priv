Block 5 Summary — Slash Commands, Command Effects, Retrieval Probe, and Editor Bridge

Block 5 defines Coeus’s deterministic command layer: slash commands, tool macros, quick pins, task commands, retrieval diagnostics, and editor bridge integration. This layer is important because it separates explicit operator commands from normal repo-aware chat/QA.

The strongest reviewer-facing point is:

Coeus has a command-first dispatch path for slash commands. Valid commands are handled deterministically, invalid slash commands are rejected deterministically, and only non-command user messages continue into normal chat/LLM repo-QA flow. Command effects are host-side/model-invisible state updates, not model-authored claims.

1. Batch-level architecture role

Block 5 separates command responsibilities into clear layers:

chat_command_dispatcher.*
  top-level command-first routing
  command disposition policy
  handled / rejected / not-command distinction

chat_command_ops.*
  implementation helpers for /tool, /task, /inventory, /fs, quick pins, autocomplete

chat_command_effects.*
  extraction of host-side effects from tool outputs
  active hint / working set / recent mutation continuity

retrieval_prober.*
  deterministic retrieval diagnostic probe

editor_bridge/*
  editor-agnostic bridge types
  concrete Sublime localhost HTTP adapter

The key architectural distinction:

Normal repo QA
  user asks a normal question
  → not a slash command
  → proceeds to agent/runtime/LLM flow

Explicit operator command
  user starts with /
  → deterministic dispatcher
  → handled or rejected
  → does not fall through to normal LLM chat

That is exactly the behavior needed to avoid commands blurring into ordinary repo QA semantics.

2. File-by-file role map
core/commands/chat_command_dispatcher.cpp/h

Role: Authoritative slash-command dispatcher.

This file owns top-level command classification and disposition.

It returns a ChatCommandResult with:

disposition:
  NotACommand
  Handled
  RejectedUnknown
  RejectedInvalid

assistant_text
effects
command_head
command_kind

The important correctness point is in the header:

invalid or unknown slash commands MUST return a rejecting disposition, never NotACommand

That means /apple, /badcmd, malformed /tool, or invalid /apply usage should not pass into normal chat as if the user asked a natural-language question.

Supported command families

The dispatcher recognizes:

/inventory
/listfiles
/list_files

/plan
/apply

/tool ...

/task
/tasks

/goal
/fact
/constraint
/pref
/decision

/fs

/probe-retrieval
/plan and /apply

These do not implement planning directly in the dispatcher. They are routed through the internal tool path:

/plan "<goal>"
  → plan.make_v1

/apply [--yes]
  → plan.apply_v1

That is a good boundary: planning/apply behavior remains in the tool/runtime layer, while the command dispatcher owns syntax and deterministic routing.

/tool

/tool commands are classified into useful kinds:

tool_help
tool_discovery
toolchain_discovery
tool_run
tool_scan
tool_embed
tool_search
tool_list
tool_show
tool_probe
tool_ctx
tool_command
Unknown/malformed handling

The dispatcher has user-friendly deterministic errors and “did you mean” hints:

/apple → Did you mean /apply --yes?
/inv   → Did you mean /inventory?
/tol   → Did you mean /tool help?

What this proves:
Slash commands are treated as a first-class deterministic command surface with explicit stop/continue semantics.

Reviewer-facing wording:
“chat_command_dispatcher is the command-first boundary. It decides whether a user message is not a command, a handled command, an unknown slash command, or an invalid slash command, ensuring malformed slash input never falls through into LLM chat handling.”

core/commands/chat_command_effects.cpp/h

Role: Extract model-invisible host effects from tool outputs.

This file translates JSON outputs from deterministic tools into CommandEffects.

Effects include:

clear_active_hint
clear_working_set
set_active_hint_rel
set_working_set_rel

clear_recent_mutation
set_recent_mutation_files
set_recent_mutation_primary_rel
set_recent_mutation_preferred_primary_rel
set_recent_mutation_source

set_recent_mutation_prior_version_artifact_exists
set_recent_mutation_exact_diff_artifact_exists
set_recent_mutation_structural_summary_available

This is important because deterministic command/tool turns can affect follow-up context. Example: after /apply --yes, the user may ask “what changed?” or “open that file.” The command layer can preserve host-side continuity truth without asking the model to infer it from chat text.

Recent mutation discipline

The file contains several mature safeguards:

preferredPrimary is metadata only
preferredPrimary must not silently promote into primary
single-file bundles may synthesize primary from the sole file
multi-file bundles must not synthesize a single primary
recent_mutation_present=false does not clear previous mutation state
only explicit clear keys may clear recent mutation continuity

This directly matches the broader CLE principle:

continuity is not proof
support-only host state should not silently upgrade into evidence truth
Path safety

The effects layer normalizes repo-relative paths and rejects traversal:

NormalizeCommandRelativePath
IsSafeRelNoTraverse
MergeUniqueRels

What this proves:
The command layer has a careful, model-invisible continuity bridge. It does not just print command output; it extracts safe host-side state for later turn resolution.

Reviewer-facing wording:
“chat_command_effects converts tool result JSON into safe host-side command effects such as active hints, working sets, and recent-mutation continuity. It explicitly prevents preferred-primary metadata from becoming mutation proof and only clears recent mutation state on explicit clear signals.”

core/commands/chat_command_ops.cpp/h

Role: Command implementation helpers.

This file implements the actual behavior behind many command families.

/tool macro support

HandleToolMacro supports:

/tool help
/tool listcommands
/tool tools
/tool chains
/tool chain <name>
/tool run <name> [json]
/tool <registeredToolName> [json]

/tool list <path>
/tool show <path>
/tool scan <folder> [ext:...]
/tool embed
/tool search "query"
/tool ctx show <path>
/tool probe

A key design detail: direct filesystem/list/show operations are bounded and path-resolved inside the project or file-context base.

Registered tool execution

Registered tools are run through:

ToolRegistry::EnsureBuiltins()
ToolRegistry::FindSpec()
ToolRunner::Run(...)

The tool run is correlated with:

project
thread_id
epoch
tier = 1
tool name
args

Then outputs are summarized into transcript-friendly text and passed through ExtractCommandEffectsFromToolOutputs.

This is a clean separation:

ToolRunner produces structured result
chat_command_ops summarizes for transcript
chat_command_effects extracts host continuity effects
/task manager

Task commands are thread-scoped and model-invisible:

/tasks
/task help
/task list
/task show <id>
/task create "<goal>"
/task cancel <id>
/task delete <id>
/task resume <id>

/task resume uses:

RunController::ResumeTask(...)

So command/task execution connects to runtime control without becoming ordinary chat.

Quick pins

Quick pins include:

/goal
/task <text>
/fact
/constraint
/pref
/decision

They append to User Context and Memory. /task <text> remains a quick pin unless the second token is a manager subcommand like list, show, create, etc.

/inventory

/inventory builds deterministic inventory text from included file-context items:

ProjectActions::BuildInventoryPayload
inventory::BuildInventoryStateV1IncludedDomain
/fs ls

A project-bounded filesystem listing command.

Search/probe

Legacy project search calls VectorMemoryService::Search or SearchFaissOnly.

/tool probe and /probe-retrieval perform retrieval diagnostics.

Catalog/autocomplete

The command layer exposes:

Catalog()
Autocomplete()
RegisterTool()
ClearRegisteredTools()
ListCommandsAlphaVec()

This supports UI dropdowns/help surfaces.

Important no-op compatibility exports

The file says deterministic command truth is runtime-owned in agent_flow.cpp:

HostCommandRequestTag now returns empty.
AppendDeterministicCommandMarkers now performs no marker mutation.

That is a very important architectural point. It avoids this command layer becoming a second truth engine for runtime markers.

What this proves:
Command operations are host-side deterministic helpers, while runtime command truth/markers are centralized elsewhere.

Reviewer-facing wording:
“chat_command_ops implements command helpers for tools, tasks, inventory, filesystem listing, file-context inspection, search, quick pins, help, and autocomplete. It intentionally leaves top-level disposition to the dispatcher and runtime marker truth to agent_flow.”

core/commands/commands.h

Role: Router header.

This is a small convenience include for:

chat_command_dispatcher.h
retrieval_prober.h

It is intentionally not the command implementation.

core/commands/retrieval_prober.cpp/h

Role: In-app retrieval diagnostic probe.

RunRetrievalProbeTranscript creates a markdown transcript for diagnostics.

It does the following:

load persisted state
create/open probe project
write sample files:
  hello.cpp
  utils.cpp
  page.html
scan + embed sample source files
write a gist summary
preload gists
run hybrid searches
run offset lookup
save/reload project state
run post-reload search
optional cleanup
return markdown transcript

This is direct evidence of a private diagnostic/eval-style capability inside the app.

Important nuance:
The comment says it creates a dedicated project and does not hijack the current active project. However, the project manager’s createProject() path sets the created project active. So this should be described as a diagnostic probe project, but we should review whether it temporarily changes active project state when run in a live UI session.

What this proves:
The command layer includes a deterministic retrieval smoke test that exercises file scanning, embedding, gist preload, hybrid search, offset lookup, save/reload, and post-reload query behavior.

Reviewer-facing wording:
“retrieval_prober provides an in-app retrieval diagnostic transcript. It creates a sample project, writes sample source files, scans and embeds them, preloads a gist, runs searches and offset lookup, saves/reloads, and verifies post-reload retrieval.”

core/integrations/editor_bridge/editor_bridge_types.h

Role: Editor-agnostic bridge contract.

Defines shared types:

EditorBridgeError
EditorBridgeState
EditorBridgeApplyRequest
EditorBridgeResult

Important fields:

file_name
view_id
is_dirty
is_read_only
change_count
history_depth
base_change_count precondition
save_after

This gives higher layers a stable contract independent of Sublime-specific transport.

What this proves:
The editor integration is designed as an adapter boundary, not hardcoded directly into workflow layers.

core/integrations/editor_bridge/sublime_bridge_client.cpp/h

Role: Concrete Sublime Text bridge adapter.

This is a Windows WinHTTP localhost client for a Sublime AgentBridge plugin.

Supported operations:

IsAvailable()
GetState()
ApplyFullText()
RevertLastApply()

Endpoints:

GET  /state
POST /apply_full_text
POST /revert_last_apply

Important safety details:

only http:// URLs are supported
default base URL is http://127.0.0.1:8765
requests are UTF-8 sanitized
apply requests can include base_change_count
target file mismatch is detected
read-only / no active view / target mismatch / precondition failure are mapped into typed errors
no hidden fallback to filesystem writes
on non-Windows, client reports unavailable/not implemented

This is a clean external editor bridge: it acts through the editor’s native apply/revert mechanism and validates state.

What this proves:
Coeus has an editor-native actuation boundary for Sublime Text, separate from repo file writes and planning logic.

Reviewer-facing wording:
“SublimeBridgeClient is a thin localhost HTTP adapter for the Sublime bridge. It can check availability, read active editor state, apply approved full-text edits with optional change-count preconditions, and revert the last bridge-applied edit. It has no planning logic, no approval logic, and no hidden fallback-to-filesystem behavior.”

3. Main command control flow
Normal user text
User message does not start with "/"
  → DispatchChatCommand returns NotACommand
  → caller continues into normal agent/runtime/LLM flow
Slash command
User message starts with "/"
  → DispatchChatCommand
      → classify command head/kind
      → route to plan/tool/task/quick-pin/explicit handler
      → return Handled or Rejected*
      → caller stops normal chat flow
Tool command
/tool run <name> [json]
  → HandleToolMacro
  → ToolRegistry::EnsureBuiltins
  → ToolRunner::Run(project, thread_id, epoch, tier, name, args)
  → ToolResultV1
  → transcript-friendly summary
  → ExtractCommandEffectsFromToolOutputs
  → ChatCommandResult.effects
Plan/apply command
/plan "<goal>"
  → dispatcher packages args
  → HandleToolMacro run plan.make_v1
  → ToolRunner path

/apply [--yes]
  → dispatcher packages confirm flag
  → HandleToolMacro run plan.apply_v1
  → ToolRunner path
  → recent mutation effects extracted if present
Task command
/task list/show/create/cancel/delete/resume
  → TaskStore
  → for resume:
      RunController::ResumeTask
Quick pin
/goal or /fact etc.
  → append to Project.contextData.userSection
  → optional UserContext aggregator
  → MemoryStore::addChunk
  → save project context
Retrieval probe
/probe-retrieval or /tool probe
  → retrieval diagnostics
  → file-context stats
  → hybrid/dense search checks
4. Command effects vs runtime operations

This distinction is important.

Runtime operations

Runtime operations actually do work:

ToolRunner::Run
RunController::ResumeTask
ProjectManager::saveProjectContext
MemoryStore::addChunk
VectorMemoryService::Search
FileContextService::ScanFolder / EmbedPending
SublimeBridgeClient HTTP calls
Command effects

Command effects are host-side metadata returned to AgentFlow:

active_hint
working_set
recent_mutation files/source/artifact flags
clear flags

They are:

model-invisible
post-commit continuity truth
not source evidence
not prompt-authority by themselves
not claim proof

This is exactly the correct conceptual boundary.

The command layer can say:

a deterministic repo.write_v1 touched src/foo.cpp

But that does not mean:

the model may claim anything about src/foo.cpp without retrieval/source evidence

That latter discipline belongs to retrieval, prompt packing, and validation.

5. Does this blur normal repo QA semantics?

The source mostly shows the opposite: it protects the boundary.

Boundary protections
1. Slash commands are explicit

Only messages starting with / enter the command path.

2. Invalid slash commands stop deterministically

Unknown or malformed slash commands return RejectedUnknown or RejectedInvalid, not NotACommand.

So /tool bad does not become a vague chat request.

3. /tool is described as operator-directed

The help text says /tool … runs explicit local/host actions instead of asking the LLM.

4. Runtime marker truth is not duplicated here

HostCommandRequestTag and AppendDeterministicCommandMarkers are no-ops because deterministic command truth is owned by agent_flow.cpp.

That avoids command ops becoming a second classifier.

5. Effects are model-invisible

The header explicitly frames CommandEffects as host-side continuity state.

6. Recent mutation rules are conservative

Preferred primary cannot silently become primary; multi-file bundles do not collapse into fake single-file truth.

Remaining caveat

Some /tool macros are transcript-visible and produce assistant text summaries. That is fine because they are explicit operator commands. The important thing is that their effects remain host state and do not automatically become source-evidence proof.

6. External/editor bridge role

The editor bridge is not part of normal repo QA. It is an external actuation adapter.

Architecture:

Project editor bridge config
  editorBridgeEnabled
  editorBridgeKind
  editorBridgeBaseUrl

EditorBridge shared types
  editor-agnostic request/result/state

SublimeBridgeClient
  concrete Windows localhost HTTP adapter
  /state
  /apply_full_text
  /revert_last_apply

This is distinct from repo tools:

repo.write_v1 / repo.apply_patch_v1
  direct repo file mutation through tool runner

SublimeBridgeClient
  editor-native mutation through active editor buffer

The source explicitly says Sublime bridge non-goals:

No planning logic
No approval logic
No hidden fallback-to-filesystem behavior

That is a strong boundary.

7. What this proves about Coeus
Proven: deterministic command surface

Coeus has a dedicated command dispatcher with explicit handled/rejected/not-command semantics.

Proven: tool commands are host/runtime operations

/tool commands route through ToolRegistry, ToolRunner, and structured ToolResultV1.

Proven: command outputs can preserve host continuity

Tool outputs can update active hints, working sets, and recent mutation continuity without injecting model-visible assumptions.

Proven: command truth is not duplicated

Command operation helpers explicitly avoid appending deterministic markers because runtime command truth is owned by agent_flow.cpp.

Proven: retrieval diagnostics exist

The retrieval probe exercises scanning, embedding, gist loading, search, offset lookup, save/reload, and post-reload search.

Proven: editor bridge is adapter-based

The shared editor bridge types are editor-agnostic; Sublime is one concrete implementation.

8. Complexity / risk notes
chat_command_ops.cpp is broad

It handles many command families: tool macros, task manager, quick pins, inventory, filesystem listing, search, retrieval probe, toolchain discovery, catalog/autocomplete. This is understandable for a command helper layer, but it is a high-density file.

Some legacy macro commands bypass ToolRunner

Commands like /tool list, /tool show, and legacy /tool search are implemented directly in chat_command_ops, while scan/embed/run use ToolRunner. That is not necessarily wrong, but for trace completeness, the review should distinguish:

ToolRunner-backed commands
  traced structured tool calls

Legacy macro helpers
  deterministic host helpers, not necessarily ToolRunner traces
Retrieval probe may alter active project

The probe creates a dedicated project, but createProject() sets the new project active. If this is run in the UI, check whether active project restoration is needed or already handled elsewhere.

retrieval_prober.h includes stdout helper

printResults writes to stdout, but RunRetrievalProbeTranscript uses markdown output and does not use stdout. For headless/protocol cleanliness, use the transcript path.

Sublime bridge is Windows-only

The concrete client is implemented through WinHTTP and returns unavailable on non-Windows. That is fine if described accurately.

9. Reviewer questions and suggested answers
Q: What happens if a user types an invalid slash command?

A:
It is rejected deterministically. The dispatcher returns RejectedUnknown or RejectedInvalid and produces an assistant-facing error message. It does not fall through into normal LLM chat handling.

Q: Are /tool commands the same as normal repo QA?

A:
No. /tool is an explicit operator command surface. It runs deterministic host/tool actions. Normal repo QA only happens when the message is not a slash command.

Q: Where do /plan and /apply run?

A:
The dispatcher routes them into registered tools: plan.make_v1 and plan.apply_v1. The dispatcher owns syntax/routing; planning/apply behavior lives in the tool/runtime path.

Q: How does the system remember what a command changed?

A:
Tool outputs are parsed into model-invisible CommandEffects, including working set and recent mutation fields. These can update host continuity after a deterministic command turn.

Q: Does a recent mutation effect prove future claims?

A:
No. It is continuity truth, not evidence truth. Source claims still require retrieval/source evidence and validation.

Q: What is the editor bridge?

A:
An adapter boundary for editor-native actuation. Shared bridge types define state/apply/result contracts, and SublimeBridgeClient implements a Windows localhost HTTP adapter for Sublime Text.

10. Notes to carry into CODEBASE_OVERVIEW.md
The command layer gives Coeus an explicit deterministic operator surface separate from normal LLM chat. `chat_command_dispatcher` is the command-first boundary: non-slash messages continue to normal agent flow, handled slash commands stop the turn with deterministic output, and malformed/unknown slash commands are rejected without falling through to the model. `chat_command_ops` implements helpers for inventory, tools, tasks, quick pins, file-system listing, file-context inspection, search, toolchain discovery, help, and autocomplete. Tool-backed commands route through `ToolRegistry` and `ToolRunner`, while `/plan` and `/apply` route to `plan.make_v1` and `plan.apply_v1`. `chat_command_effects` extracts model-invisible host continuity effects from tool outputs, including active hints, working sets, and recent mutation metadata. Retrieval probing and the Sublime editor bridge provide deterministic diagnostics and external editor actuation without becoming normal repo-QA semantics.
11. Notes to carry into ARCHITECTURE_NOTES.md
Block 5 command architecture

Dispatcher:
  DispatchChatCommand(project, thread_id, epoch, userText)
    → NotACommand
    → Handled
    → RejectedUnknown
    → RejectedInvalid

Command families:
  /inventory
  /plan
  /apply
  /tool ...
  /task /tasks
  /goal /fact /constraint /pref /decision
  /fs
  /probe-retrieval

Tool flow:
  /tool run <name> [json]
    → ToolRegistry::EnsureBuiltins
    → ToolRunner::Run(project, tid, epoch, tier=1, name, args)
    → ToolResultV1
    → transcript summary
    → CommandEffects extraction

Plan/apply:
  /plan "<goal>" → plan.make_v1
  /apply [--yes] → plan.apply_v1

Command effects:
  active_hint
  working_set
  recent_mutation files / primary / preferred_primary / source / artifact flags
  model-invisible continuity
  not source evidence

Task flow:
  /task manager commands
    → TaskStore
    → RunController::ResumeTask for resume

Quick pins:
  /goal etc.
    → Project.contextData.userSection
    → MemoryStore
    → save project context

Retrieval probe:
  sample project
  scan/embed/gist/search/reload check

Editor bridge:
  shared editor-agnostic types
  Sublime localhost HTTP client
  no planning/approval/filesystem fallback
12. Portfolio-facing summary
Block 5 shows Coeus’s deterministic command layer. Slash commands are routed through a command-first dispatcher that cleanly distinguishes normal chat, handled commands, invalid commands, and unknown commands. Explicit operator commands such as `/tool`, `/plan`, `/apply`, `/task`, `/inventory`, and `/probe-retrieval` run host/tool operations instead of being treated as ordinary LLM prompts. Tool outputs can return model-invisible command effects such as active hints, working sets, and recent mutation continuity, while source claims still depend on retrieval and validation. The block also includes an in-app retrieval probe and an editor-bridge abstraction with a concrete Sublime localhost adapter for editor-native state, apply, and revert operations.

Stronger version:

The Block 5 source proves that Coeus separates deterministic operator commands from normal repository QA. Slash commands are handled or rejected before the LLM path, tool-backed commands route through the ToolRunner/runtime layer, and command effects preserve host-side continuity without becoming evidence truth. This gives the app a controlled command surface for file context, tools, planning, tasks, retrieval diagnostics, and editor bridge actuation while keeping ordinary source-grounded questions on the normal retrieval/agent path.