Block 4 Summary — Project Model, Persistence, Project Actions, and State Ownership

Block 4 is the project substrate. It defines what a Coeus “project” is, how projects and threads are persisted, how thread state is loaded without blocking the UI, how project actions mutate state, and how project state feeds the GUI, retrieval, memory, telemetry, agent profile selection, file context, and runtime.

The strongest reviewer-facing point is:

Coeus has a real project model, not just an in-memory chat session. A project owns durable identity, user-facing title, local folder path, agent profile assignment, editor bridge preferences, file context, project context, retrieval/debug trace state, and a multi-thread conversation model with branch/close/delete semantics, focused-file state, bounded live tails, archive sidecars, summary metadata, and telemetry warm-loading.

1. Batch-level role

Block 4 establishes the unit of work for the whole app:

Project
  identity / UUID / title / folder
  assigned agent profile
  editor bridge config
  file context root + file context items
  project context data
  retrieval/lattice trace cache
  conversation threads
  active thread index
  thread-scoped focus
  runtime recent-file-context latch

Then it separates responsibilities:

project_data.*
  data structures and pure helpers

project_io.*
  disk format, sidecar files, atomic writes, tail/archive/context/file-context IO

project_manager.*
  lifecycle, open/active project management, async warming/hydration, memory binding

project_actions.*
  user-intent operations: create, rename, branch, switch, close, delete, focus

This is a clean conceptual split: data, IO, manager, and actions.

2. File-by-file role map
core/project/project_data.h/cpp

Role: Core data model.

This defines the durable project structures:

FileContextItem
ConversationMessage / Message
SummaryBlock
ProjectContextData
ConversationThread
Project

The most important model decisions:

Project owns threads.
ConversationThread::tail is the authoritative live conversation state.
Project::messages is legacy compatibility only.
Focused file state is thread-scoped.
Thread epoch is durable and persisted.
Close is non-destructive.
Delete is destructive.
Project UUID is stable and persisted.
PID is local/internal.
Important structures
FileContextItem

Represents files known to the project file-context system:

relPath
contents
included
attachVerbatim
status Pending/Embedded
byteCount
sliceCount
estTokens

This feeds File Context Manager, retrieval, inventory payloads, and file-context persistence.

ConversationMessage

Supports normal messages and summary nodes:

author / senderName
content
isUser
isSummaryNode
summary metadata:
  fromPair
  toPair
  path
  approxTokens
  createdAt

The summary-node model matters because the conversation can contain compacted or summarized history as first-class renderable messages, with metadata pointing back to archive paths.

ConversationThread

This is the main per-thread state:

id
title
tail
blocks
archivePath
archivedCount
assistantOrdinalBase
lastTurnEpoch
lastActivity
focusedFile / focusEnabled / focusedRelPath
parentThreadId
branchOrdinal
createdAtUtc
isClosed
branchedAtUtc
branchDepth
inheritPending
tailLoaded
tailDirty

Key semantics:

lastTurnEpoch is durable and prevents per-thread epoch reset after restart.
focusedFile is canonical; focusedRelPath is a compatibility alias.
isClosed means hidden/non-destructive close, not delete.
parentThreadId, branchOrdinal, and branchDepth define branch identity.
tailLoaded and tailDirty separate runtime hydration from persistence.
Project

The project model contains:

uniqueID
project_uuid
title
folderPath
createdAtUtc
lastModifiedAtUtc
assignedProfileIndex
editorBridgeEnabled / Kind / BaseUrl
threads / activeThreadIdx
fileContextRootFolder / fileContextItems
recentFileCtxRels
contextData
legacy focus fields
isDirty
lastActivityTime
projMutex

The code comment is explicit that Project::messages is a compatibility shim and not authoritative. The authoritative conversation state lives in ConversationThread::tail.

What this proves:
Coeus has a durable project/thread model with branchable conversation state, source-context state, focused-file state, prompt/profile binding, editor bridge preferences, and trace/context metadata.

core/project/project_io.cpp/h

Role: Disk persistence and sidecar IO.

This file defines how project state is stored on disk.

Major persisted artifacts:

index.json
threads_meta.json
conversations/<threadId>.tail.json
conversations/<threadId>.jsonl
conversations/<threadId>.blocks.json
context.md
context_meta.json
agent_config.md
file_context.json
index.json

Stores project registry and open/active project state:

projects:
  id
  title
  folderPath
  isDirty
  lastActivityTime
  assignedProfileIndex
  fileContextRootFolder
  project_uuid
  createdAtUtc
  lastModifiedAtUtc
  editorBridgeEnabled
  editorBridgeKind
  editorBridgeBaseUrl

openIDs
activeProjectID
threads_meta.json

Stores thread metadata only, not large tails:

activeThreadIdx
threads:
  id
  title
  archivePath
  archivedCount
  assistantOrdinalBase
  lastTurnEpoch
  lastActivity
  focusedFile
  focusEnabled
  parentThreadId
  branchOrdinal
  createdAtUtc
  isClosed
  branchedAtUtc
  branchDepth

This is important: thread metadata is lightweight and separate from conversation content.

.tail.json

Stores a bounded live tail cache for fast startup/thread switch:

version: 1
tail: [...]

The code caps persisted tails:

max messages: 240
max chars: 160,000

And treats .tail.json as a bounded cache, not full history.

.jsonl archive

The full conversation archive is preserved as JSONL:

role
content
time

This allows older conversation content to live outside the fast tail.

.blocks.json

Stores summary block metadata:

from
to
path
tokens
created
context.md and context_meta.json

context.md stores user context and summary context separated by:

---

context_meta.json stores:

lastRetrievalTrace
traceMode

That lastRetrievalTrace field is used by GUI debug surfaces, dashboard context bars, retrieval trace views, and Evidence/UI reconstruction paths.

file_context.json

Stores file context root and included/attached file list:

rootFolder
items:
  relPath
  included
  attach
IO safety

project_io has a global IO mutex and atomic-ish write strategy:

WriteTextAtomic:
  write temp sibling
  flush
  rename temp over target

AppendJsonlAtomic:
  mutex-protected append

This is a serious persistence detail.

Tail safety behavior

If a .tail.json is huge, malformed, corrupt, or too slow to parse, ProjectIO rebuilds a bounded tail from the archive JSONL instead of blocking or crashing the app.

That is a strong responsiveness design:

huge/corrupt tail
  → read last archive lines
  → rebuild bounded tail
  → rewrite compact tail
  → continue

What this proves:
Coeus has a sidecar persistence model designed for large/long-running conversation projects: metadata is separated from live tail caches and full JSONL archives, with atomic writes and recovery from pathological tail files.

core/project/project_manager.cpp/h

Role: Project lifecycle manager and state coordinator.

This is the central manager for:

loadFromDisk
saveToDisk
createProject
openProject
closeProject
deleteProject
active project selection
project lookup
thread tail hydration
memory-store binding
profile sync
file-context hydration tracking
project warming
telemetry warm-loading
Important ownership

ProjectManager owns pointer-stable storage:

m_byId: unordered_map<int, shared_ptr<Project>>
m_allIDs: stable UI/index ordering
m_openIDs
m_activeProjectID
m_nextID

This gives immediate raw pointer access for local use and shared-pointer access for async jobs.

Startup behavior

loadFromDisk() does:

loadIndexFile()
for open projects:
  loadConversationHistory()
  loadAgentConfig()
  loadProjectContext()

bind memory store to active project
sync active agent profile
defer file-context load unless explicitly hydrated
schedule project warm

Important: file context is not eagerly hydrated. The code deliberately logs that file-context load is deferred to explicit file-context operations.

Save behavior

saveToDisk():

snapshot projects
for dirty/open projects:
  SaveThreadsMeta
  Save dirty loaded tails
  SaveProjectContext
  SaveAgentConfig
  SaveFileContext only if hydrated
  save index

That “save file context only if hydrated” rule prevents wiping old file_context.json with an empty in-memory vector when file context was deliberately not loaded.

Tail hydration

loadActiveThreadTail() is non-blocking:

if active tail loaded:
  return true

else:
  scheduleThreadTailHydrateAsync
  return true

This is important for UI responsiveness. Tail hydration is scheduled on the IO lane.

ensureThreadTailLoaded() is the synchronous/truthful loader for explicit thread ID cases, used from IO/background paths.

Project warming

scheduleWarmAsync_() warms:

startupReconcileForProject_
VectorMemoryService::WarmProject(... ManifestOnly)

But file-context reconcile is skipped unless file context is hydrated.

Memory binding

When the active project changes, ProjectManager binds MemoryStore to:

<project>/.memory

This connects project lifecycle to context/memory subsystems.

Agent profile sync

syncActiveAgentProfile_() makes the active AgentProfilesManager profile match the project’s assigned profile, or uses a fallback active/default profile if project is unassigned.

Telemetry warm-loading

The manager schedules IO-lane warm loads for:

ActivityFeed
RunIndex

This feeds the GUI telemetry layer and fixes “activity disappeared after restart” behavior.

File-context hydration LRU

The manager tracks a small set of hydrated file-context projects:

kMaxHydratedProjects = 5
m_fctxLRU
m_fctxHydrated

When a project is evicted, it persists file context if needed and clears in-memory file contents/items.

What this proves:
Coeus has a professional project manager that separates active/open project registry, lazy file-context hydration, non-blocking thread tail loading, memory-store binding, profile sync, background warming, and persisted project state.

core/project/project_actions.cpp/h

Role: User-intent operations over project/thread state.

This file contains user-facing actions that higher-level UI and conversation manager calls.

It owns:

project CRUD:
  new
  close
  delete
  rename
  move
  save/save as

thread CRUD:
  new
  switch
  rename
  delete
  close
  reopen
  branch

thread tail hydration:
  ensure/hydrate

focused file:
  set
  clear
  get

inventory payload:
  BuildInventoryPayload
Async save behavior

Project actions schedule saves on the IO lane:

ScheduleSaveProjectAsync_
  lane: IO
  priority: Low
  serialKey: PA:SAVE:p<projectID>
  coalesceQueued: true

This prevents the UI from blocking on frequent project mutations.

Timestamp invariants

The file centralizes:

EnsureProjectTimestampsOnCreateLocked_
TouchProjectModifiedLocked_

Rule:

createdAtUtc is set once if missing
lastModifiedAtUtc updates on meaningful mutation
lastActivityTime updates alongside lastModifiedAtUtc
isDirty = true on mutation
Thread creation

AttemptNewThread creates a new thread with:

new UUID
title
archivePath = conversations/<tid>.jsonl
lastTurnEpoch = 0
parentThreadId = 0
tailLoaded = true
tailDirty = true

Then it loads active tail, saves if dirty, and warms telemetry.

Branching

BranchFromActiveThread is one of the strongest features in this block.

It:

identifies active source thread
creates new thread ID
computes branch ordinal among siblings
generates default title like:
<parentBase>-branch N
copies a bounded tail slice if parent tail is already loaded
preserves focused-file state from parent
sets:
parentThreadId
branchOrdinal
branchedAtUtc
branchDepth
archivePath
inheritPending
schedules async inherit if parent tail was not loaded
avoids copying large sidecar archives on the UI thread

The comments explicitly state that inherited summary nodes may continue to refer to the source archive, which is truthful branch semantics.

Close vs delete

CloseThread sets:

isClosed = true

and selects another open thread if needed.

DeleteThread erases the thread and clears telemetry. This is a clean semantic distinction.

Focused file

Focus is thread-scoped:

SetFocusedFile(projectID, threadID, relPath)
ClearFocusedFile(projectID, threadID)
GetFocusedFile(projectID, threadID)

Paths are normalized through FileCtxSvc::NormalizePath and sanitized.

This focused file feeds UI and likely turn-context/retrieval behavior in later blocks.

Inventory payload

BuildInventoryPayload converts included file-context items into inventory::InventoryItem records. This is a bridge from project file-context state into the inventory/context plane.

What this proves:
User-facing project operations are separated from raw manager/IO. Thread branches, closes, focus, saves, telemetry warm-loading, and dirty/timestamp behavior are intentionally modeled.

3. Main data/control flows
Startup flow
main.cpp
  → ProjectManager::loadFromDisk()
      → loadIndexFile()
      → for each open project:
          loadConversationHistory()
            → LoadThreadsMeta()
            → schedule tail hydrate if needed
            → schedule telemetry warm
          loadAgentConfig()
          loadProjectContext()
      → bind MemoryStore to active project .memory
      → sync active agent profile
      → defer file-context hydrate
      → schedule project warm

Key design point: startup does not eagerly parse all tails or all file-context content.

Save flow
ProjectActions mutation
  → TouchProjectModifiedLocked()
  → isDirty = true
  → ScheduleSaveProjectAsync()
      → ProjectManager::saveProject()
      → ProjectManager::saveToDisk()

ProjectManager save
  → SaveThreadsMeta
  → Save dirty loaded thread tails
  → SaveProjectContext
  → SaveAgentConfig
  → SaveFileContext only if hydrated
  → SaveIndexFile

The save path is coalesced and IO-lane scheduled for UI actions.

Thread tail load flow
Switch/open project or thread
  → ProjectManager::loadActiveThreadTail()
      if tailLoaded:
        return immediately
      else:
        scheduleThreadTailHydrateAsync()

IO lane
  → ProjectIO::LoadThreadTail()
      if normal:
        parse bounded tail
      if huge/corrupt:
        rebuild bounded tail from archive JSONL
  → mark tailLoaded
  → tailDirty = false
  → warm ActivityFeed / RunIndex
  → invalidate UI panes

This supports responsiveness for large histories.

Conversation persistence flow
ConversationThread::tail
  → bounded cache:
      conversations/<tid>.tail.json

Full history:
  → conversations/<tid>.jsonl

Summary block metadata:
  → conversations/<tid>.blocks.json

Thread metadata:
  → threads_meta.json

Important distinction:

tail.json = fast live cache
jsonl archive = full durable history
threads_meta.json = lightweight metadata
Branch flow
active thread
  → BranchFromActiveThread()
      → copy bounded parent tail if loaded
      → otherwise mark inheritPending
      → create branch metadata
      → copy focused file state
      → set active thread to branch
      → async inherit parent tail if needed
      → save meta/tail
      → warm telemetry

This supports branch workflow without UI stalls.

File-context state flow
Project.fileContextRootFolder
Project.fileContextItems
  → FileContextManager UI
  → BuildInventoryPayload()
  → FileContextService / retrieval / inventory blocks

ProjectManager:
  load file context only when explicitly requested
  save file context only if hydrated
  evict hydrated file-context state via LRU

This protects performance and prevents accidental persistence loss.

Active project state flow
setActiveProjectID(projectID)
  → load conversation history/context/config
  → sync active profile
  → bind MemoryStore to <project>/.memory
  → defer file-context hydrate
  → schedule warm
  → save index

This is the bridge from project selection to memory, UI, profile, and retrieval readiness.

4. How project state feeds other subsystems
Feeds GUI
Project.title
  → AppBar/sidebar/tab labels

Project.threads
  → sidebar thread tree
  → conversation thread strip
  → conversation_output

ConversationThread.tail
  → ConvView document model

ConversationThread.isClosed / parentThreadId / branchOrdinal
  → branch strip and sidebar tree

Project.contextData.lastRetrievalTrace
  → Dashboard context usage
  → File Context Manager trace view
  → Context Output / debug surfaces

Project.fileContextItems
  → File Context Manager
  → Memory Map
  → Context Output
Feeds retrieval / CLE
Project.fileContextRootFolder
Project.fileContextItems
  → FileContextService
  → inventory payload
  → vector memory / retrieval surface

ConversationThread.focusedFile
  → thread-scoped file focus
  → target/retrieval policy in later blocks

Project.contextData
  → user context / summary / trace cache

Project folder path
  → .memory
  → summaries
  → .cle_traces
  → .cle_workspace
Feeds runtime
active project ID
active thread ID
assignedProfileIndex
lastTurnEpoch
focusedFile
thread tail
contextData
file-context state

These are the core runtime inputs for turn execution, continuity, prompt assembly, and evidence surfaces.

Feeds telemetry
ProjectActions / ProjectManager
  → warm ActivityFeed and RunIndex for thread

delete/close/thread changes
  → clear or warm telemetry caches

project folder path
  → trace / .cle_traces / activity files
5. What this proves about the project
Proven: durable project model

Safe claim:

Coeus persists projects as durable workspace units with metadata, threads, context, profile assignment, file context, and editor bridge preferences.

Proven: multi-thread/branch conversation model

Safe claim:

Coeus supports multiple conversation threads per project, including branch identity, branch ordinal/depth, non-destructive close, destructive delete, and thread-scoped focused-file state.

Proven: large-history handling

Safe claim:

Coeus separates full JSONL conversation archives from bounded live tail caches, allowing fast startup/thread switching while preserving full history separately.

Proven: non-blocking UI design

Safe claim:

Project saves, thread hydration, telemetry warm-loading, and branch inheritance are scheduled on background lanes to avoid blocking the native GUI.

Proven: source-context state is part of the project

Safe claim:

File-context root, included files, attach flags, and retrieval trace state are persisted as project data and exposed to retrieval/UI subsystems.

Proven: project identity is reviewer-friendly

Safe claim:

Projects have both local PIDs and stable persisted UUIDs; the UI is intended to show user-facing project titles rather than internal IDs.

6. Architecture strengths
Strong separation between manager, IO, actions, and data

This is the cleanest Block 4 story:

project_data = structures
project_io = disk formats + atomic writes
project_manager = lifecycle + active/open registry + warming/hydration
project_actions = user-intent mutations
Thoughtful performance strategy

Several choices are clearly designed to protect UI responsiveness:

async saves
coalesced IO tasks
async tail hydration
deferred file-context hydration
LRU file-context unloading
bounded tail persistence
rebuild from archive if tail cache is huge/corrupt
no synchronous branch sidecar clone
Good thread semantics

The code distinguishes:

close thread = hide/non-destructive
delete thread = erase/destructive
branch = explicit parent/ordinal/depth
focus = thread-scoped
tailLoaded/tailDirty = runtime persistence state

That is mature enough to explain to a technical reviewer.

Persistence has recovery paths

ProjectIO::LoadThreadTail has defensive behavior for oversized, malformed, or exception-throwing tail files. It rebuilds from JSONL where possible.

File-context save safety

The “save only if hydrated” rule is important. It prevents a project save from overwriting existing file-context state just because the manager intentionally deferred loading that file context.

7. Complexity / risk notes
Compatibility shims are still present

There are legacy fields:

Project::messages
Project::focusedFile / focusedRelPath / focusEnabled
ConversationMessage::senderName
mirrorActiveTailToLegacy

The code comments correctly mark these as compatibility. In review docs, phrase them as transitional shims, not core architecture.

Tail/archive split needs clear explanation

A reviewer may ask whether only the last 240 messages are persisted. The answer is:

No — the bounded tail is a fast cache.
Full history is preserved in JSONL archive sidecars.

Be ready to say that clearly.

project_actions.h declares legacy focus helpers not shown implemented in this block

The header declares:

AttemptSetFocusedFile
AttemptSetFocusedFolder
AttemptClearFocus

but the supplied project_actions.cpp portion shows the newer explicit functions:

SetFocusedFile
ClearFocusedFile
GetFocusedFile

This may mean the legacy functions are implemented elsewhere, omitted from the paste, or currently stale declarations. Worth checking later, but not a blocker for understanding the architecture.

appendUserMessage / appendAssistantMessage in ProjectManager only mark metadata/dirty state

In the supplied code, these functions do not append message content directly to tail; they update activity/dirty metadata. That likely means actual message insertion is owned by ConversationManager in a later block. Do not describe ProjectManager as the main message writer.

Branch inherited summary paths intentionally reference source archives

This is called out in comments as truthful branch semantics. It is correct, but a reviewer may ask. Explanation:

A branch can inherit summary nodes referring to the source archive because they summarize inherited conversation history. New branch turns append to the branch’s own archive.
File context is not loaded eagerly

This is a performance choice, but it means some file-context state exists as persisted sidecar until an explicit operation hydrates it. This should be explained as intentional lazy hydration.

8. Reviewer questions and suggested answers
Q: What is the core unit of work in Coeus?

A:
A Project. It owns durable identity, title, folder path, assigned agent profile, editor bridge settings, file-context state, project context, retrieval trace metadata, and a multi-thread conversation model.

Q: Are conversations single-threaded?

A:
No. A project owns multiple ConversationThread objects. Each thread has its own ID, title, archive path, live tail, turn epoch, focused file, branch metadata, close state, and dirty/load flags.

Q: Does close delete a thread?

A:
No. Close is non-destructive and persists isClosed=true. Delete is a separate destructive operation that erases the thread.

Q: How are branches represented?

A:
A branch thread stores parentThreadId, branchOrdinal, createdAtUtc, branchedAtUtc, and branchDepth. It can inherit a bounded slice of parent tail if loaded, or schedule async inheritance if the parent tail must be hydrated first.

Q: How does Coeus avoid loading huge conversation histories on startup?

A:
It separates thread metadata from conversation content. threads_meta.json is lightweight. Each thread has a bounded .tail.json cache for the live tail, while the full conversation is preserved in a JSONL archive. Tail loading is async for UI paths and has recovery logic for huge/corrupt caches.

Q: Where does full conversation history live?

A:
Full history is appended to conversations/<threadId>.jsonl. The .tail.json file is only a bounded cache for fast UI startup and thread switching.

Q: How does project state feed retrieval?

A:
The project stores fileContextRootFolder, fileContextItems, user/project context, focused-file state, and recent retrieval trace state. Included file-context items can be converted into inventory payloads, and the focused file is thread-scoped for turn-context/retrieval policy.

Q: Is file context always loaded?

A:
No. File context is deliberately lazily hydrated. The manager tracks hydrated projects and saves file context only if it was actually hydrated, preventing accidental overwrite of persisted file-context sidecars.

9. Notes to carry into CODEBASE_OVERVIEW.md
The project subsystem defines Coeus’s durable workspace model. A `Project` owns stable identity, user-facing title, local folder path, assigned agent profile, editor bridge configuration, file-context state, project context, retrieval trace metadata, and a multi-thread conversation model. Conversations are not stored as a single monolithic chat log: each `ConversationThread` has metadata in `threads_meta.json`, a bounded live-tail cache in `conversations/<thread>.tail.json`, and a full JSONL archive in `conversations/<thread>.jsonl`. The manager loads metadata eagerly but hydrates thread tails asynchronously for UI responsiveness. Project actions implement user-intent operations such as project creation, thread creation, branching, switching, close/reopen, delete, and thread-scoped focused-file control. File-context state is lazily hydrated and only persisted when loaded, protecting existing file-context sidecars from accidental overwrites.
10. Notes to carry into ARCHITECTURE_NOTES.md
Block 4 project architecture

Data model:
  Project
    identity: uniqueID, project_uuid
    title/folder
    timestamps
    assignedProfileIndex
    editor bridge config
    threads[]
    fileContextRootFolder
    fileContextItems
    recent file-context latch
    contextData
    dirty/activity flags

  ConversationThread
    id/title
    tail
    archivePath
    lastTurnEpoch
    focusedFile/focusEnabled
    branch metadata
    close state
    tailLoaded/tailDirty

Persistence:
  index.json
    project registry + open/active projects

  threads_meta.json
    lightweight thread metadata, active index

  conversations/<tid>.tail.json
    bounded live-tail cache

  conversations/<tid>.jsonl
    full conversation archive

  conversations/<tid>.blocks.json
    summary block metadata

  context.md + context_meta.json
    user/project context + last retrieval trace + trace mode

  file_context.json
    root folder + included/attached file list

Manager:
  ProjectManager
    pointer-stable project registry
    active/open project lifecycle
    profile sync
    MemoryStore binding
    lazy file-context hydration
    async project warming
    async tail hydration
    telemetry warm-loading

Actions:
  ProjectActions
    project/thread CRUD
    branch creation
    close vs delete
    focused-file mutation
    async coalesced save scheduling
    inventory payload bridge
11. Portfolio-facing summary
Block 4 shows the private project substrate behind Coeus AI. Projects are durable local workspaces with stable identity, project folders, profile assignment, editor bridge settings, file-context state, project context, retrieval trace metadata, and multi-thread conversation state. Conversations are persisted through lightweight thread metadata, bounded live-tail caches, full JSONL archives, and summary block sidecars. The project manager keeps startup responsive through lazy file-context hydration, asynchronous thread-tail loading, background warming, and coalesced IO-lane saves. Project actions provide user-level operations for creating projects, creating/switching/branching/closing/deleting threads, and managing thread-scoped focused files.

A stronger version:

The Block 4 source proves that Coeus is a project-aware workspace rather than a session-only chatbot. The app persists project identity, context, source-file state, profile assignment, editor bridge settings, multi-thread conversation history, branches, focused-file state, retrieval trace metadata, and telemetry warm state. Its persistence design separates fast live-tail caches from full JSONL archives, keeping the native UI responsive while preserving durable conversation history.