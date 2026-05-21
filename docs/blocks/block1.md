Block 1 Summary — Entry Points, Core Utilities, and App Shell

Block 1 establishes that Coeus AI has a conventional native C++ application boot path, with early initialization for paths, logging, provider/profile config, project state, background task infrastructure, and GUI handoff.

The important source-proven point for the portfolio overview is:

Coeus AI is not launched as a script or web wrapper. It has a native C++ main.cpp entrypoint that initializes runtime folders, logging, libcurl, provider/profile configuration, project state, and then hands control to a separate GUI translation unit through InitGuiMain(), RunGuiMain(), and ShutdownGuiMain().

This block also shows several foundational systems that support the larger architecture: deterministic runtime paths, project-local artifacts, CLE trace/workspace directories, thread-safe logging, background task lanes, main-thread UI dispatch, UTF-8 sanitization, and token-budget helpers.

1. File-by-file role map
main.cpp

Role: Native application entrypoint and top-level boot coordinator.

main.cpp keeps the real main() tiny and delegates to MyApp::mainAppEntry(argc, argv). Inside mainAppEntry, the app performs a clear startup sequence:

Initialize runtime paths via PathUtils::Init(argv[0]).
Initialize file logging under PathUtils::LogsDir() / "myapp_log.txt".
On Windows, disable QuickEdit and request UTF-8 console codepages.
Initialize libcurl globally.
Set current working directory to PathUtils::ProjectRoot(), which resolves to the executable/runtime root.
Ensure build/config exists.
Load providers from providers.json.
Load agent profiles from profiles.json.
Log SDL video driver environment state.
Load project state using ProjectManager::instance().loadFromDisk().
Parse basic CLI window flags:
--fullscreen
--size <w> <h>
Initialize GUI with InitGuiMain(width, height, fullscreen).
Enter the GUI event loop with RunGuiMain().
Shut down the task engine.
Save project state, profiles, and providers.
Shut down GUI, libcurl, and logger.

Portfolio significance:
This is the cleanest source-level evidence for the app shell. It proves the application has a native lifecycle, config loading, project persistence, provider/profile persistence, and a GUI runtime boundary.

Important detail:
The file explicitly states that there is no agent_defaults.json config file by design. Runtime defaults live in compiled code and/or profiles. That helps explain the configuration boundary professionally: the app loads providers and profiles, not arbitrary hidden default-agent config.

core/logger.cpp / core/logger.h

Role: Thread-safe logging subsystem with runtime log levels and console routing.

The logger provides:

InitLogger(filepath)
ShutdownLogger()
SetLevel() / GetLevel()
SetConsoleSink() / GetConsoleSink()
File logging with directory creation.
Optional console mirroring to stderr/stdout/disabled.
Environment configuration:
MYAPP_LOG_LEVEL=error|warn|info|debug|verbose|0..4
MYAPP_LOG_CONSOLE=stderr|stdout|none|off|0|1|2

A distinctive implementation detail is that MyLogging::Log(...) is implemented as a macro:

In debug builds, it lazily evaluates logging arguments and respects runtime log level.
In release builds with NDEBUG, it compiles to a no-op.

Portfolio significance:
This supports the claim that the app has structured diagnostic infrastructure rather than casual printf debugging. It also matters for headless/operator use because console routing can keep stdout protocol-clean while preserving file logging or stderr logs.

Reviewer-facing wording:
“Coeus uses a small thread-safe logging layer with runtime log-level and console-sink controls. In debug builds, log calls are lazily evaluated; in release builds, most log calls compile away to avoid formatting overhead.”

Nuance / risk:
Because Log(...) compiles out in release builds, release artifacts will not emit the same runtime diagnostics unless built without NDEBUG or unless specific non-macro logging paths are used. That is not a flaw by itself, but it is worth being precise about.

core/path_utils.cpp / core/path_utils.h

Role: Centralized runtime path and artifact-location utility.

This is more important than a normal path helper. It defines the runtime file layout used by several larger subsystems.

It owns paths for:

Runtime root
ProjectRoot()
ConfigDir()
AssetsDir()
LogsDir()
ProjectsDir()
Config files
ProvidersJson()
ProfilesJson()
PromptsJson()
Legacy traces
TracesDir()
TraceThreadDir()
TraceArtifactPath()
Canonical CLE traces
CleTracesDir()
CleArtifactsDir()
CleToolsDir()
CleToolsTurnsDir()
ToolTracePath()
Agent/tool workspace
CleWorkspaceDir()
ResolveWorkspacePath()
WorkspaceToolArtifactPath()
Semantic summaries
SummariesDir()
SemanticSummaryArtifactPath()
SemanticSummaryStatePath()
SemanticSummaryCompatMirrorPath()
Repo-relative path normalization
NormalizeRelRepoPath()

The implementation includes strong path-safety logic:

rejects empty/invalid paths
strips quotes and outer whitespace
normalizes slashes
rejects absolute/rooted paths
rejects ..
rejects NUL bytes
limits component length
uses deterministic fallback names for invalid semantic keys
ensures workspace paths remain inside .cle_workspace

Portfolio significance:
This file proves several deeper product claims before we even inspect retrieval/runtime:

The app is project-aware.
It writes project-scoped summaries.
It has trace/artifact infrastructure.
It has CLE-specific trace roots.
It has tool trace paths.
It has a constrained workspace for agentic/tool mutations.
It distinguishes project-local artifacts from shared fallback artifacts.

This is strong evidence for the “local-first project workspace” and “debuggable source-grounded runtime” positioning.

Reviewer-facing wording:
“PathUtils centralizes the runtime filesystem contract. It defines where config, logs, projects, summaries, traces, CLE artifacts, tool traces, and workspace files live, and includes defensive normalization to prevent workspace-relative paths from escaping the project-controlled root.”

core/task_manager.cpp / core/task_manager.h

Role: Background task engine plus main-thread UI dispatcher.

This subsystem has two major parts.

1. Tasking::MainThread

A UI-agnostic main-thread dispatcher:

Post(fn) can be called from background threads.
Drain() / DrainCount() / DrainBudget() are intended to be called by the UI thread.
SetWakeFn() allows the UI/event loop to be woken immediately after background work posts back to the main thread.

This is important for a native GUI app because background tasks need a safe way to return results to the UI thread without directly touching UI state.

2. Tasking::TaskEngine

A lane-based worker-pool engine with:

lanes:
UIHigh
IO
CPU
LLM
priorities:
High
Normal
Low
cooperative cancellation
task handles with status/error
per-lane queues
serial keys for mutual exclusion
queued-task coalescing
lane-local and global resource gates
configurable worker counts
graceful shutdown with cooperative cancellation

The LLM lane is especially relevant. It indicates the codebase treats model/provider work as a distinct task class rather than mixing everything into generic background threads.

Portfolio significance:
This is evidence that Coeus has a real application runtime substrate. It is not just one blocking request/response loop. The app has explicit concurrency lanes for UI-sensitive work, IO, CPU work, and LLM work.

Reviewer-facing wording:
“Coeus includes a small lane-based task engine that separates UI-high-priority tasks, IO tasks, CPU tasks, and LLM tasks. It supports cooperative cancellation, per-key serialization, queued-work coalescing, and resource gates for shared resources such as local model runtimes or cloud-provider limits.”

Nuance / risk:
The shutdown comment in main.cpp says “Ensure all background tasks have completed before persisting,” but the actual call is TaskEngine::Shutdown() with default cancelRunning=true. That means queued tasks are cancelled and running tasks receive cooperative cancellation before join. The accurate wording is “cooperative task shutdown before persistence,” not “guaranteed completion of every queued task.”

core/tokenizer.h

Role: Lightweight header-only token estimation and prompt-budget helper.

This file provides:

rough token counting
model-aware token estimates
chat-message token estimates
context-window inference from model names
prompt budget helpers
prompt-fit checks
trim helpers
chunk-by-token helpers
optional backend hooks for a real tokenizer

The important architectural point is that this is a budgeting utility, not a tokenizer engine in itself. It defaults to heuristics, but it can be wired to a real backend through function pointers.

Portfolio significance:
This supports the prompt/context-packing story. Even before reviewing prompt assembly, we can see the codebase has utilities for prompt budgeting, context windows, trimming, and chunking.

Reviewer-facing wording:
“Coeus includes lightweight token-budget utilities used to estimate prompt size, reserve completion space, trim context, and split text into budgeted chunks. The default implementation is heuristic, with hooks for a real tokenizer backend.”

Nuance:
From this block alone, we should not claim that a real BPE tokenizer is currently wired in. The source here proves that the hook exists, not that it is used.

core/utf8.h

Role: Header-only UTF-8 sanitizer and validator.

This file provides:

Utf8::Sanitize()
Utf8::IsValidUtf8()
replacement of invalid UTF-8 with U+FFFD
rejection of UTF-16 surrogate halves encoded as UTF-8
optional mapping of stray Windows-1252 bytes to proper Unicode punctuation
optional JSON-tree sanitization when UTF8_WITH_NLOHMANN_JSON is enabled

Portfolio significance:
This is basic but useful infrastructure for a project-aware assistant, because source files, logs, prompts, traces, and UI rendering may encounter mixed or malformed text encodings.

Reviewer-facing wording:
“Coeus includes a robust UTF-8 sanitization utility to keep logs, prompts, JSON artifacts, and UI text safe when source files or external inputs contain invalid byte sequences.”

tinyfiledialogs.cpp/h

Not included in the provided batch, so we cannot assess the actual source here.

Based on the planned review order, this should be described cautiously as likely bundled/support code only after verifying the files. Do not present it as core authored architecture unless the source shows meaningful project-specific integration.

2. Main control flow proven by Block 1

The top-level boot flow is:

main()
  → MyApp::mainAppEntry(argc, argv)
      → PathUtils::Init(argv[0])
      → MyLogging::InitLogger(build/logs/myapp_log.txt)
      → Windows console setup, if _WIN32
      → curl_global_init()
      → current_path(PathUtils::ProjectRoot())
      → ensure build/config exists
      → ProvidersManager::loadProviders(build/config/providers.json)
      → AgentProfilesManager::loadProfiles(build/config/profiles.json)
      → ProjectManager::loadFromDisk()
      → parse --fullscreen / --size
      → InitGuiMain(width, height, fullscreen)
      → RunGuiMain()
      → TaskEngine::Shutdown()
      → ProjectManager::saveToDisk()
      → AgentProfilesManager::saveProfiles()
      → ProvidersManager::saveProviders()
      → ShutdownGuiMain()
      → curl_global_cleanup()
      → ShutdownLogger()

This is a clean application shell. The important separation is:

main.cpp owns boot/shutdown coordination.
gui_main.cpp owns actual GUI lifecycle.
ProjectManager owns project persistence.
ProvidersManager owns provider config persistence.
AgentProfilesManager owns profile persistence.
TaskEngine owns background worker shutdown.
PathUtils owns filesystem layout.

That is a professional architecture boundary.

3. What this block proves about Coeus AI

This block provides source-level support for several portfolio claims:

Proven claim: Native C++ desktop app

main.cpp is a native C++ entrypoint. It initializes a GUI lifecycle through separate native functions:

bool InitGuiMain(int width, int height, bool fullscreen);
int  RunGuiMain();
void ShutdownGuiMain();

This supports the claim that Coeus is a native desktop application, not a web-only wrapper.

Proven claim: Project-aware runtime

main.cpp loads project state through:

ProjectManager::instance().loadFromDisk();

and saves it on exit:

ProjectManager::instance().saveToDisk();

PathUtils also defines project-scoped summaries, traces, CLE traces, and workspace directories. That supports the claim that the app works around local projects, not stateless chat sessions.

Proven claim: Provider/profile configuration

The app loads and saves:

providers.json
profiles.json

through provider and agent-profile managers.

That supports the claim that model/provider/profile configuration is part of the application shell.

Proven claim: Background runtime substrate

TaskEngine provides distinct worker lanes for:

UIHigh
IO
CPU
LLM

This supports the claim that Coeus has a native application runtime capable of asynchronous/background work.

Proven claim: Trace/artifact-oriented design

PathUtils defines:

.cle_traces/
.cle_traces/artifacts/
.cle_traces/tools/
.cle_workspace/
summaries/
traces/

This supports claims about telemetry, debugging, tool traces, workspace isolation, summaries, and reproducible internal artifacts.

4. Important implementation details to carry forward
Runtime root is executable-relative

PathUtils::Init(argv[0]) sets the runtime root from the executable location. In practice, comments indicate this is expected to be the build directory.

This matters for deployment/review because runtime files are expected under:

build/
  config/
  logs/
  projects/
  assets/
Project-local artifacts are intentionally separated

The source distinguishes between:

<project>/summaries/
<project>/traces/
<project>/.cle_traces/
<project>/.cle_workspace/

That suggests a layered artifact model:

summaries are semantic/context artifacts
traces are diagnostic artifacts
.cle_traces are canonical CLE observability artifacts
.cle_workspace is controlled tool/workspace output

This distinction will matter later when reviewing CLE, tools, telemetry, and headless eval.

Workspace path safety is explicit

ResolveWorkspacePath() rejects unsafe relative paths and verifies the resolved path remains under CleWorkspaceDir().

That is important reviewer evidence because agent/tool systems that can write files need bounded write surfaces.

Tasking separates execution concerns

The task system does not just run arbitrary background work. It classifies work by lane and allows concurrency limits. That is relevant later for:

file indexing
summaries
model calls
local/remote provider execution
UI updates
tool execution
5. Architecture strengths visible in this block
Clear boot ownership

main.cpp is not doing retrieval, prompting, validation, or GUI rendering directly. It coordinates startup and shutdown only.

That is a good sign for maintainability.

Centralized path contract

PathUtils avoids scattered string paths across the app. It also creates a single place to reason about what artifacts are written where.

This will help explain the private review boundary, because we can point to the exact artifact roots that should be excluded or included.

Defensive filesystem handling

Path normalization and workspace resolution show attention to path traversal and unsafe artifact writes.

Asynchronous foundation

The task engine is non-trivial. It includes priorities, lanes, cancellation, resource limits, and main-thread posting. That is meaningful native-app infrastructure.

Text robustness

UTF-8 sanitization and token budgeting indicate the app expects to process messy source text and large prompt/context payloads.

6. Complexity / risk areas to mention carefully

These are not refactor demands, just reviewer-aware notes.

Logging is build-mode dependent

Most MyLogging::Log(...) calls compile out under NDEBUG. That is performance-friendly, but release/demo diagnostics will differ from debug/headless diagnostics unless configured intentionally.

Shutdown wording should be precise

TaskEngine::Shutdown() cancels queued work and cooperatively requests cancellation from running tasks. It then joins workers. So the portfolio should say:

“The app performs cooperative background-task shutdown before persisting state.”

Avoid saying:

“The app waits for all background jobs to complete.”

That is not exactly what the code does.

PathUtils already carries many subsystem concerns

PathUtils includes general config paths, traces, CLE traces, semantic summaries, tool traces, and workspace paths. This is useful centralization, but it also means this file is a key architecture boundary. Later reviews should check whether downstream modules consistently use these helpers rather than inventing parallel paths.

Tokenizer is heuristic unless backend is wired

Do not oversell tokenizer.h as exact token accounting from this file alone. It is a budgeting utility with optional real-backend hooks.

7. Reviewer questions and suggested answers
Q: What is the actual application entrypoint?

A:
The real C++ entrypoint is main() in main.cpp, which immediately delegates to MyApp::mainAppEntry(). That function initializes runtime paths, logging, curl, config directories, providers, profiles, project state, CLI window options, and then hands control to the GUI lifecycle through InitGuiMain(), RunGuiMain(), and ShutdownGuiMain().

Q: Is this a native app or a web shell?

A:
Block 1 proves a native C++ application shell. It has a normal compiled C++ main(), direct Windows console handling, libcurl global initialization, filesystem-backed runtime directories, and a separate native GUI lifecycle. The rendering details are reviewed in Block 2.

Q: Where does project state enter the application?

A:
Project state is loaded during startup through ProjectManager::instance().loadFromDisk() and saved during shutdown through ProjectManager::instance().saveToDisk(). PathUtils also defines project-scoped directories for summaries, traces, CLE artifacts, tools, and workspace files.

Q: How are provider settings handled?

A:
The app loads providers from build/config/providers.json and profiles from build/config/profiles.json, then saves them back on shutdown. main.cpp explicitly notes that there is no agent_defaults.json; runtime defaults live in code and/or profiles.

Q: Does the app have background processing infrastructure?

A:
Yes. TaskEngine provides lane-based worker pools for UI-high-priority work, IO, CPU, and LLM tasks. It supports task priorities, cancellation, serial-key mutual exclusion, queued-task coalescing, and global/lane-local resource limits.

Q: How does background work safely update the GUI?

A:
Tasking::MainThread provides a UI-agnostic dispatcher. Background threads can call Post(), and the UI thread drains posted work once per frame. A wake hook can notify the UI event loop immediately when work is posted.

Q: How are internal traces and tool artifacts organized?

A:
PathUtils defines deterministic roots for legacy traces, canonical CLE traces, tool traces, CLE artifacts, and workspace artifacts. Tool traces are stored under .cle_traces/tools/turns, and workspace writes are constrained under .cle_workspace.

8. Notes for CODEBASE_OVERVIEW.md

Use this wording:

The application starts from a native C++ entrypoint in `main.cpp`. Startup initializes executable-relative runtime paths, logging, libcurl, provider/profile configuration, project state, and then transfers control into the GUI lifecycle through `InitGuiMain()`, `RunGuiMain()`, and `ShutdownGuiMain()`.

Core utilities establish the base runtime contract: `PathUtils` centralizes config, logs, project folders, summaries, CLE traces, tool traces, and controlled workspace paths; `TaskEngine` provides lane-based background execution for UI, IO, CPU, and LLM work; `MyLogging` provides thread-safe logging with runtime sink/level controls; UTF-8 and token-budget utilities support safe text handling and prompt/context budgeting.
9. Notes for ARCHITECTURE_NOTES.md

Carry forward this architecture map:

main.cpp
  owns boot/shutdown coordination

PathUtils
  owns runtime filesystem contract
  config/logs/projects/assets
  project summaries
  legacy traces
  CLE traces
  tool traces
  controlled workspace paths

Logger
  owns thread-safe diagnostics
  supports debug/headless/operator output routing

TaskEngine
  owns background execution substrate
  lanes: UIHigh / IO / CPU / LLM
  supports cancellation, priorities, coalescing, serial keys, resource gates

GUI lifecycle
  declared in main.cpp, implemented later in gui_main.cpp
  InitGuiMain → RunGuiMain → ShutdownGuiMain

ProjectManager / ProvidersManager / AgentProfilesManager
  loaded at startup
  saved at shutdown
10. Short portfolio-facing summary

For a high-level portfolio overview, use:

Block 1 shows the native application shell and core runtime utilities. Coeus AI starts from a compiled C++ entrypoint, initializes executable-relative runtime paths, logging, provider/profile configuration, project state, libcurl, and then enters a separate GUI lifecycle. The supporting core utilities define the project/runtime filesystem contract, including config, logs, summaries, CLE traces, tool traces, and controlled workspace paths. A lane-based task engine provides asynchronous execution for UI, IO, CPU, and LLM workloads, while UTF-8 and token-budget helpers support safe source-text handling and prompt/context budgeting.

A slightly stronger version:

The first source block establishes Coeus AI as a real native application rather than a thin chatbot wrapper. The app has a C++ boot sequence, project persistence, provider/profile loading, deterministic runtime directories, controlled trace/workspace paths, background task lanes, and a clean handoff into the GUI runtime. These foundations support the later project-aware retrieval, prompt assembly, tool execution, telemetry, and headless evaluation layers.
