Block 3A Summary — GUI Product Surfaces, Context Views, Evidence, and File-Context Controls

This first half of Block 3 is where the native GUI foundation from Block 2 becomes a visible product. These files prove that Coeus is not only a custom-rendered desktop app; it has actual workspace surfaces for projects, threads, context, evidence, traces, file context, summaries, memory/context mapping, command execution, editor bridge controls, profile/model selection, and retrieval/debug visibility. The uploaded batch includes the core visible surfaces around AppBar, command palette, context output, conversation output, dashboard/top strip, evidence UI, file context manager, and memory map.

The main reviewer-facing point is:

Block 3A shows the user-facing workspace layer: Coeus exposes project/thread navigation, chat and trace views, source evidence chips, context/inventory/meta views, file-context management, profile/model controls, editor bridge commands, prompt/context usage indicators, and a memory/context map. These are native C++/Skia UI surfaces backed by project state, conversation state, CLE file context, summaries, telemetry traces, and tool execution.

1. Batch-level architecture role

Block 2 proved the rendering shell. Block 3A proves the first set of application-level panes rendered inside that shell:

AppBar
  top title/menu strip
  project title + active surface label
  file/chat/tools/view commands

Command Palette
  Sublime-style command runner
  preferences/provider/editor bridge commands
  UI setting commands

Dashboard / Top Strip
  active profile/model/temperature
  context usage bar
  file-context token estimate
  navigation buttons
  profile dropdown

Conversation Output
  chat transcript view
  thread strip / branch tabs
  chat-vs-trace toggle
  line-buffer renderer
  minimap
  source evidence anchoring
  telemetry activity notes
  trace viewer integration

Context Output
  user context
  conversation summary
  semantic project meta
  file summaries
  project inventory
  inclusion toggles for prompt inventory

Evidence Box
  per-message source evidence chips
  source rows only
  open/pin/dismiss actions
  explicit authority boundary

File Context Manager
  operator surface for source files
  add folder/files
  scan/embed/rescan
  include/pin/focus controls
  retrieval trace view

Memory Map
  zone-based view of persona/user context/summary/latest prompt/file context
  folder/file tree
  focused file and pinned visibility

This block is especially strong for portfolio evidence because it shows the app has real product workflow, not just backend architecture hidden behind a chat box.

2. File-by-file role map
gui/components/app_bar.cpp

Role: Top application bar, title/menu strip, native-feeling app commands.

app_bar.cpp renders the top title strip and menu strip. It owns:

app title display: Coeus — <project title> — <surface>
window button hit-testing:
minimize
maximize/restore
close
menu labels:
File
Edit
Chat
Tools
View
Help
menu dropdown overlays
hover-switching between open menus
action codes returned to gui_main
menu commands for project/thread/tool/view actions

Important behavior:

File → New Project / Exit
Chat → New Chat / New Branch
Tools → Profiles / Providers / File Context / Memory Map
View → Toggle Sidebar / Density / Perf HUD

The source explicitly states that the AppBar is not runtime-theme customizable and should feel like a Windows native title/menu bar, with themeable UI starting below it. That is a useful design boundary.

What this proves:
The app has a desktop-style shell, not just embedded chat content. It exposes projects, threads, tools, file context, memory map, profile/provider settings, density, sidebar, and perf HUD through a conventional app menu.

Reviewer-facing wording:
“app_bar.cpp implements a native-feeling title/menu strip for Coeus, including window controls, menu overlays, project-aware title display, and app-level commands for projects, chat threads, tools, view settings, and diagnostic surfaces.”

gui/components/command_palette.cpp

Role: Sublime-like command palette and command launcher.

This file implements the Ctrl+Shift+P command palette. It includes:

modal command palette panel
text filter input
keyboard navigation
mouse execution
close-on-Escape / outside click behavior
UI settings commands
provider settings command
minimap/font commands
real editor bridge commands

The editor bridge commands are important:

Editor Bridge: Enable
Editor Bridge: Disable
Editor Bridge: Toggle
Editor Bridge: Check Status
Editor Bridge: Revert Last Apply

The palette resolves the active project through ProjectManager, supports an active-project override hook, writes bridge settings into project state, saves the project, and uses SublimeBridgeClient for status/revert operations.

What this proves:
The command palette is not cosmetic. It is a real workspace command surface that controls provider settings, UI preferences, minimap/font settings, and external editor bridge integration.

Reviewer-facing wording:
“command_palette.cpp provides a Sublime-style command palette for application actions and developer workflow controls, including provider preferences, UI settings, minimap/font controls, and real project-scoped Sublime editor bridge commands.”

gui/components/context_output.cpp

Role: Context tab UI for user context, summaries, semantic project meta, file summaries, and inventory.

This is one of the most important Block 3A files because it directly exposes Coeus’s project-awareness and context model.

The Context tab has five sub-tabs:

User Context
Conversation Summary
Project Meta
File Summaries
Inventory

The file’s own comments define an important CLE posture:

Inventory == “what exists”
Semantic meta == “what it means”
File Summaries == semantic-first, with inventory only as navigation scaffolding

That distinction is very valuable for the architecture notes because it matches the broader principle we have been tracking:

inventory/summaries can route and scaffold
source evidence proves claims
semantic meta is not the same as inventory

Important implementation responsibilities:

loads/sanitizes context and summary buffers from Project::contextData
reads/writes context inclusion toggles
mirrors inclusion settings to <summaries>/__inventory__/config.ini
reads semantic project meta via PathUtils::SemanticSummaryArtifactPath
reads inventory JSON/Markdown artifacts
canonicalizes file paths through FileCtxSvc::NormalizePath
supports PR7-style .memory/__meta__/v1 mirror paths
renders collapsible JSON-like sections
shows file summaries as an inventory-driven file list, expanding into semantic per-file mirrors
highlights the focused file in file summaries
distinguishes stable artifact timestamps from per-frame Now()

What this proves:
Coeus has a dedicated UI surface for context state, conversation summaries, project semantic meta, file summaries, and inventory. This is direct product evidence for “project-aware assistant,” “context lattice,” “inventory,” and “source-context workflow.”

Reviewer-facing wording:
“context_output.cpp renders the project context surface. It separates user context, conversation summary, semantic project meta, file summaries, and deterministic inventory, and it treats inventory as factual navigation scaffolding rather than semantic proof.”

Important nuance:
The Summarize path here is marked as a minimal/hypothetical local summarizer in this file. Do not oversell that button as the final LLM summary pipeline from this file alone. The stronger evidence is the context/inventory/meta surface and its artifact loading model.

gui/components/conversation_output.cpp

Role: Main conversation transcript, thread strip, trace mode, minimap, evidence anchoring, and telemetry activity overlay.

This file is a large product-surface orchestrator. It renders the chat transcript using the conv/* line-buffer modules:

conv_view
conv_painter
conv_minimap

Key responsibilities:

maintains per-thread view state, selection state, doc cache, and scrollbar state
snapshots conversation tail from project/thread state
supports thread-based conversations and legacy messages fallback
renders a thread strip with:
active thread tabs
new thread
branch thread
close/switch/reorder thread behavior
Chat/Trace toggle
supports Chat mode and Trace mode
opens trace details through TraceFileViewer
integrates GuiTelemetry snapshots
injects safe host activity notes into the transcript view
derives friendly action summaries from tool/chain trace JSON
anchors evidence panels under assistant messages
uses EvidenceUI::MeasureInlineButtonPanel to reserve layout space
supports summary-node raw expansion via background IO task
renders minimap and right scrollbar
uses profile role-label settings and agent display name

What this proves:
The conversation pane is much more than printing messages. It is a transcript renderer, thread/branch manager, trace viewer gateway, telemetry overlay, evidence anchoring surface, minimap surface, and summary-expansion UI.

Reviewer-facing wording:
“conversation_output.cpp is the main conversation surface. It renders a cached line-buffer transcript, supports thread tabs and branch workflow, toggles between chat and trace mode, integrates telemetry activity notes, anchors source-evidence panels under assistant messages, and delegates line rendering/minimap behavior to the conv/* modules.”

Important architecture boundary:
The file injects host/tool activity as view-only notes. That is good: it keeps telemetry visible to the user without necessarily making it part of the model-visible transcript.

gui/components/dashboard.cpp

Role: Top strip inside the main content area.

This is the strip below the tab bar, not the global AppBar.

It provides:

navigation buttons:
Chat
Context
File Context
Memory Map
active model chip
profile chip/dropdown
temperature display and slider
file-context token estimate
context usage bar
segmented context usage from lattice prompt markers
tooltip overlays
profile assignment behavior

The context usage bar is especially valuable. It parses contextData.lastRetrievalTrace and looks for lattice prompt markers such as:

lattice.tok_budget
lattice.tok_used
packed.turn_state_bytes
packed.conv_tail_bytes
packed.user_ctx_bytes
packed.conv_summary_bytes
packed.inventory_bytes
packed.summaries_bytes
packed.retrieval_bytes

This means the GUI can visualize prompt/context packing categories: turn state, conversation tail, user context, conversation summary, inventory, summaries, and retrieval.

What this proves:
The GUI exposes model/profile configuration and context-packing diagnostics directly in the product surface.

Reviewer-facing wording:
“dashboard.cpp renders the in-main top strip with navigation shortcuts, effective profile/model/temperature controls, file-context token estimates, and a segmented context-usage bar derived from lattice/retrieval trace markers.”

gui/components/evidence_box.cpp/h

Role: Per-assistant-message source evidence UI.

This file is extremely important for the source-grounding story because it explicitly defines its own authority boundary.

The comments state:

this is a presentation / interaction seam
it does not decide authoritative evidence
it does not decide what was packed into the prompt
it does not decide what the model was allowed to quote
chat evidence UI is sources only
planner/meta payloads can be cached but are not rendered in the chat evidence panel

The data model supports:

EvidenceItem
  id
  offset
  tok
  title
  relPath
  method
  origin
  injection
  score
  snippet
  why

PlannerTrace
TurnMeta

But the rendering contract is intentionally source-only:

RenderInlineButtonPanel()
RenderEvidenceForMessage()
HasEvidence()
MeasureInlineButtonPanel()

Evidence rows support:

open
pin / unpin
dismiss
method badges
injection badges
origin badges
score bars
offsets/tokens
snippets
source count chip

Opening a local evidence row also sets the per-thread focused file through ConversationManager::SetFocusedFile, which is a useful workflow bridge from evidence → focus.

What this proves:
The GUI exposes source evidence attached to assistant messages, but it does not confuse displayed evidence with prompt-authoritative evidence. This is a very strong architecture maturity point.

Reviewer-facing wording:
“evidence_box.cpp renders per-message source evidence rows while explicitly preserving the authority boundary: the UI displays evidence attached by telemetry/aggregation, but retrieval, prompt packing, validation, and exporter logic decide what was authoritative.”

gui/components/file_context_manager.cpp

Role: Operator surface for project file context.

This file is another strong source-grounding product surface.

The file comments define a key principle:

UI never mutates Project::fileContextRootFolder or Project::fileContextItems directly. All mutations are performed via tools submitted as /tool run ....

That is important because it means the UI uses the same tool/action path that should generate traces and preserve runtime discipline.

Controls include:

Add Files
Add Folder
Clear
Embed
Refresh
Trace toggle
Clear focused file
Per-file rescan/embed
Per-file focus
Per-file pin
Per-file include
Folder bulk include toggle

Tool calls include:

fc.set_root_v1
fc.scan_folder_async_v1
fc.add_files_v1
fc.embed_pending_async_v1
fc.cancel_ingest_v1
fc.rescan_then_embed_async_v1
fc.set_files_included_v1
fc.set_files_attached_v1

It also has a retrieval trace mode that reads project.contextData.lastRetrievalTrace and can show dense seeds, after-neighbor expansion, final top-k, or raw JSON.

What this proves:
The app has an explicit file-context management UI for adding/scanning/embedding/pinning/focusing project files. This directly supports the portfolio claim that Coeus loads and indexes local project files.

Reviewer-facing wording:
“file_context_manager.cpp is the operator UI for project file context. It exposes folder/file selection, scan/embed/rescan workflows, include/pin/focus controls, ingest progress, cancellation, and retrieval trace inspection while routing mutations through tool calls instead of directly modifying project file-context state.”

gui/components/memory_map.cpp

Role: Visual memory/context overview.

The visible header and search results show a zone-based memory map with:

Persona
User Context
Conversation Summary
Latest User Prompt
File Context

It also renders folder/file context as a collapsible tree, supports scroll/drag behavior, shows focused-file state, and marks pinned files used for retrieval bias. The comments also state that it snapshots project state under projMutex to avoid reading concurrently mutated vectors during async scans/embeds.

What this proves:
There is a dedicated visual surface for explaining what context the app has: persona/profile context, user context, conversation summary, latest prompt, and file context.

Reviewer-facing wording:
“memory_map.cpp gives reviewers a visual map of Coeus’s context zones and file-context tree, including project/thread focus state and pinned/included file visibility.”

3. Main data/control flows
App/menu command flow
AppBar / AppShortcuts / CommandPalette
  → UIActions / ProjectActions
  → ConversationManager
  → TabBar open/focus calls
  → UIFramework invalidation

This is the normal product-command path for things like new project, new chat, new branch, open profile/provider/file-context/memory-map surfaces, toggle sidebar, set density, and show perf HUD.

Conversation rendering flow
ProjectManager::findProject(projectID)
  → active thread / tail snapshot
  → ConvView::BuildDoc(...)
  → EvidenceUI::MeasureInlineButtonPanel(...)
  → ConvPainter::Render(...)
  → ConvMiniMap::Render(...)
  → EvidenceUI::RenderInlineButtonPanel(...)

The important point: conversation rendering is built around a document model (ConvView::Doc) and line-buffer rendering, not just immediate string painting.

Evidence display flow
Telemetry / aggregation layer
  → EvidenceUI::Attach(pid, threadId, msgIndex, EvidenceItem[])
  → ConversationOutput checks EvidenceUI::HasEvidence(...)
  → layout reserves space using MeasureInlineButtonPanel(...)
  → RenderInlineButtonPanel(...)
  → user can Open / Pin / Dismiss source rows

Authority boundary:

EvidenceUI displays attached rows.
Retrieval / prompt packing / validation / exporter decide evidence truth.

That distinction should be preserved in final architecture docs.

Context tab flow
Project contextData
  → user context / conversation summary buffers

PathUtils summaries + .memory meta roots
  → semantic project meta
  → inventory JSON/Markdown
  → per-file semantic mirrors

ContextOutput
  → canonical JSON-like sections
  → collapsible read-only viewer
  → prompt inclusion toggles

Important semantic distinction:

Inventory = factual enumeration
Semantic meta = meaning/responsibilities
File summaries = semantic mirrors navigated through inventory
File context manager flow
User action in File Context Manager
  → SubmitTool_(projectID, toolName, args)
  → ConversationManager::BeginAsyncUserMessage("/tool run ...")
  → tool/runtime path handles mutation
  → FileContextService progress polling for UI
  → project fileContextItems snapshot rendered

This is a good design story: file-context mutations are tool-owned, not arbitrary UI writes.

Dashboard/top-strip diagnostic flow
Project.contextData.lastRetrievalTrace
  → parse lattice/prompt markers
  → context usage bar
  → segmented categories:
       turn state
       conversation tail
       user context
       conversation summary
       inventory
       summaries
       retrieval

This visually connects runtime prompt packing to the GUI.

4. What this proves about Coeus
Proven: workspace UI, not just chat

Block 3A shows menus, command palette, thread tabs, branches, profile/model controls, context/inventory views, file-context management, evidence panels, trace mode, memory map, and dashboard diagnostics.

Safe claim:

Coeus has a native workspace interface with project, thread, context, evidence, tools, telemetry, file context, profile/provider, and trace/debug surfaces.

Proven: source-grounding is visible in the UI

The following UI surfaces expose source-grounding concepts:

File Context Manager
  add/scan/embed/include/pin/focus files

Context Output
  inventory, semantic meta, file summaries

Conversation Output
  evidence chips under assistant messages

Dashboard
  context usage / retrieval pack segments

Memory Map
  file-context tree and context zones

Safe claim:

Coeus exposes the project/source context pipeline through visible operator surfaces rather than hiding all retrieval state inside the model call.

Proven: evidence boundary is explicit

evidence_box.cpp is unusually clear that EvidenceUI is not the source of truth for prompt-authoritative evidence. It renders source rows attached by the UI aggregation layer, while retrieval, prompt packing, validation, and exporter logic decide evidence authority.

Safe claim:

The UI distinguishes displayed evidence from authoritative retrieval/prompt truth.

That is a very strong professional point.

Proven: file-context mutations are tool-routed

file_context_manager.cpp states and implements that file-context mutations are performed through tools, submitted as /tool run ..., rather than mutating project file-context state directly.

Safe claim:

File-context management is surfaced in the UI but routed through the tool/runtime path, preserving traceability and avoiding parallel mutation semantics.

Proven: trace/debug visibility exists in product UI

conversation_output.cpp supports Chat/Trace mode and opens trace details. dashboard.cpp parses lattice/prompt markers for context usage. file_context_manager.cpp has retrieval trace mode.

Safe claim:

Coeus has built-in trace/debug surfaces for inspecting runtime context, retrieval, and tool activity.

5. Architecture strengths
Strong product-surface coverage

The files cover a large portion of the reviewer-visible application:

app menus
command palette
conversation transcript
thread/branch workflow
trace mode
context tab
file context manager
memory map
evidence chips
top-strip diagnostics

This makes the portfolio claim concrete.

Good boundary discipline in key places

Two boundaries are especially strong:

EvidenceUI
  display seam only
  not evidence authority

FileContextManager
  operator UI only
  mutations go through tools

These are the kinds of boundaries a technical reviewer will appreciate.

Context concepts are visible and separated

context_output.cpp separates:

User Context
Conversation Summary
Project Meta
File Summaries
Inventory

and explicitly defines inventory as “what exists” versus semantic meta as “what it means.” That aligns well with the broader CLE architecture.

Runtime observability is user-facing

Telemetry/trace data is not only written to files; it is surfaced through:

trace mode
activity notes
evidence chips
context usage bar
retrieval trace table
perf HUD

This supports the “debuggable AI workspace” claim.

6. Complexity / risk notes

These are not refactor recommendations; they are notes for honest private review.

conversation_output.cpp is a major complexity hotspot

It coordinates transcript snapshots, thread tabs, branch tabs, minimap, scrollbars, summary expansion, telemetry activity injection, trace mode, evidence spacing, and line-buffer rendering. This is appropriate for a rich conversation surface, but it is high-coordination code.

Good reviewer phrasing:

“The conversation pane is one of the densest UI components because it integrates transcript rendering, thread workflow, evidence anchoring, trace mode, telemetry activity notes, minimap, selection, and summary expansion.”

context_output.cpp mixes display logic with artifact discovery

It has substantial logic for finding semantic project meta, inventory files, PR7 mirrors, per-file summaries, fallback paths, and UI wrapping. This is useful for product behavior, but it makes the Context tab both a renderer and an artifact reader.

Good reviewer phrasing:

“The Context tab is intentionally artifact-aware; it reads project context, inventory, and semantic summary artifacts directly to expose the current context state.”

Some UI commands are placeholders

AppBar has disabled placeholders like Open Project, Settings, About, and Shortcuts. Do not present every menu item as complete. The completed functionality is still substantial.

Evidence debug flag appears forced on

evidence_box.cpp has kForceEvidenceDebug = true. That may be intentional during current development, but for polished review/release it is worth checking whether evidence debug logging should remain forced.

The command palette includes real bridge commands but depends on active project state

This is fine, but reviewer wording should be project-scoped: bridge settings are persisted on the active project.

7. Reviewer questions and suggested answers
Q: Is Coeus just a chat window?

A:
No. The first half of Block 3 shows menus, command palette, tabs, thread/branch management, trace mode, context/inventory views, file-context controls, evidence panels, model/profile controls, memory map, and runtime diagnostics. The chat surface is only one workspace tab.

Q: How does the app show source grounding?

A:
Source grounding is visible through several UI surfaces: File Context Manager manages scanned/embedded/included/pinned files; Context Output shows inventory, semantic project meta, and file summaries; Conversation Output anchors evidence chips under assistant messages; Dashboard shows context-usage breakdown; Memory Map visualizes context zones and file context.

Q: Does EvidenceUI determine what evidence is authoritative?

A:
No. evidence_box.cpp explicitly states that it is a presentation/interaction seam. It renders source rows attached by the UI aggregation layer. Retrieval, prompt packing, validation, and exporter logic decide what was authoritative or model-visible.

Q: How are file-context mutations performed from the UI?

A:
The File Context Manager routes mutations through tools submitted as /tool run ..., including setting roots, scanning folders, adding files, embedding pending files, rescan+embed, include toggles, and pin toggles. The UI does not directly mutate the core file-context vectors.

Q: Does the UI expose runtime traces?

A:
Yes. Conversation Output has Chat/Trace mode and opens trace details through TraceFileViewer. Dashboard parses lattice/prompt markers from the last retrieval trace. File Context Manager includes a retrieval trace view with dense, after-neighbor, final top-k, and raw JSON modes.

8. Notes to carry into CODEBASE_OVERVIEW.md

Use this:

The first half of the GUI component layer turns the native SDL2/Skia shell into a full project-aware workspace. `app_bar.cpp` and `command_palette.cpp` provide desktop-style commands for projects, threads, tools, provider/profile settings, UI settings, and editor bridge operations. `conversation_output.cpp` renders the main chat surface as a line-buffer transcript with thread/branch workflow, trace mode, minimap, telemetry activity notes, summary expansion, and source-evidence anchoring. `context_output.cpp` exposes user context, conversation summary, semantic project meta, file summaries, and project inventory while preserving the distinction between inventory as factual enumeration and semantic meta as meaning. `file_context_manager.cpp` provides the operator surface for adding, scanning, embedding, pinning, including, focusing, and tracing project file context. `evidence_box.cpp` renders source evidence attached to assistant messages while explicitly preserving the boundary that displayed evidence is not the same as prompt-authoritative evidence.
9. Notes to carry into ARCHITECTURE_NOTES.md

Carry forward this map:

Block 3A GUI product-surface map

AppBar
  desktop menu/title strip
  project title + active tab surface
  new project/chat/branch
  open profile/provider/file-context/memory-map tabs
  view settings / perf HUD

CommandPalette
  Ctrl+Shift+P modal command runner
  UI settings / provider settings
  editor bridge enable/disable/toggle/status/revert
  minimap/font settings

Dashboard Top Strip
  quick nav: Chat / Context / File Context / Memory Map
  profile/model/temperature controls
  file-context token estimate
  context usage bar from lattice/retrieval trace markers

ConversationOutput
  line-buffer transcript renderer
  thread and branch strip
  Chat/Trace mode
  telemetry activity note injection
  evidence anchoring
  summary raw expansion
  minimap and scrollbar

ContextOutput
  User Context
  Conversation Summary
  Project Meta
  File Summaries
  Inventory
  inventory inclusion toggles
  semantic meta rebuild
  PR7/meta mirror reads
  focused file highlighting

EvidenceUI
  source-only per-message evidence display
  open/pin/dismiss interactions
  explicit non-authority boundary

FileContextManager
  Add Files / Add Folder / Clear / Embed / Refresh
  scan/embed/rescan via tool calls
  include/pin/focus controls
  retrieval trace view

MemoryMap
  context zones
  file-context tree
  focused/pinned file visibility
10. Short portfolio-facing summary

Use this version:

Block 3A shows the first major set of Coeus AI’s user-facing workspace surfaces. The app includes a native AppBar, Sublime-style command palette, context dashboard, rich conversation transcript, thread/branch workflow, trace mode, source evidence chips, context/inventory views, file-context management, memory map, profile/model controls, context-usage diagnostics, and editor bridge commands. These surfaces connect the GUI directly to project state, conversation state, CLE file context, semantic summaries, inventory artifacts, telemetry traces, and tool execution.

Stronger version:

The Block 3A source proves that Coeus AI is a full project-aware desktop workspace rather than a standalone chat pane. Users can inspect and manage source context, view semantic project summaries and deterministic inventory, control file inclusion/pinning/focus, inspect retrieval traces, open source evidence attached to assistant answers, switch between chat and trace views, manage conversation branches, and see profile/model/context-usage state directly in the native UI.




Block 3B Summary — Conversation Renderer Internals, Panels, Sidebar Helpers, Tool Service, and Telemetry Aggregation

This second half of Block 3 is the internal GUI infrastructure behind the product surfaces from Block 3A. Block 3A showed the visible workspace panes; Block 3B shows the machinery that makes those panes usable: the line-buffer conversation renderer, syntax highlighting, minimap, prompt/settings/tool panels, non-transcript tool service bridge, sidebar project/thread modeling, and the GUI telemetry aggregator that attaches CLE evidence and turn metadata into the chat UI.

The main reviewer-facing point:

Block 3B proves Coeus has a custom conversation document model and renderer, not a generic text box. It also proves the GUI has dedicated infrastructure for prompt management, UI settings, tool/monitor panels, sidebar ordering/thread trees, non-transcript tool execution, and telemetry aggregation from ActivityFeed, RunIndex, RunTrace, CLE manifests, CLE bundles, retrieval artifacts, and EvidenceUI.

1. Batch-level architecture role

Block 3B can be grouped into five layers:

1. Conversation document/rendering engine
   conv_view
   conv_painter
   conv_syntax
   conv_minimap
   conv_fence_parser

2. Workspace/panel surfaces
   panel_prompts
   panel_ui_settings
   panel_tools
   panel_monitor

3. GUI service bridge
   non_transcript_tool_service

4. Sidebar data/model helpers
   side_bar_helper

5. GUI telemetry aggregation
   telemetry_ui

This batch is less about “what the app looks like” and more about how the GUI behaves as a serious development workspace.

2. File-by-file role map
gui/conv/conv_view.cpp/h

Role: Conversation document model builder.

This is the core of the custom transcript renderer. It converts a vector of Message objects into a flat renderable Doc:

Message snapshot
  → DocLine[]
  → prefixY[]
  → CodeBlock[]
  → totalH

It owns:

role lines:
User
Assistant
Summary
Host/System as Role::None
text lines
summary headers and summary bodies
fenced code block detection
assistant-only file header association
code block metadata:
first line
last line
language
file label
assistant key
soft wrapping for non-code lines
no-wrap behavior for code
implicit verbatim mode for huge assistant outputs
document height/prefix sums
scroll clamping/sticky-bottom behavior
hit testing
selection state
Ctrl+A selection
selection extraction
code block extraction for copy buttons

Important design point:

conv_view builds the semantic/renderable line document.
conv_painter paints it.
conversation_output orchestrates it.

What this proves:
The transcript is not rendered as raw paragraphs. It is compiled into a structured document model with line kinds, roles, code blocks, summary nodes, selection support, and copy extraction.

Reviewer-facing wording:
“conv_view.cpp is the conversation document compiler. It transforms thread messages into a structured line-buffer document with role lines, text lines, summary nodes, code blocks, spacers, wrapping metadata, prefix-height tables, selection support, and stable anchors for summary/evidence behavior.”

gui/conv/conv_painter.cpp/h

Role: Paints the ConvView::Doc.

This file renders the line-buffer document produced by conv_view.

It owns:

visible range painting with overscan
text/code font setup
role-based colors from UISettings
code background
summary background/header styling
text selection rectangles
hit-testing handoff to ConvView
Ctrl+A and Ctrl+C support
syntax highlighting via ConvSyntax
code block headers
file/language labels above code blocks
“Copy” button per code block
“Copied!” toast/flash

Important behavior:

conv_view creates CodeBlock metadata
conv_painter draws code headers and copy controls
ConvSyntax provides visible-line highlighting spans

What this proves:
Coeus has a custom transcript renderer with developer-grade affordances: selectable text, copyable code blocks, syntax highlighting, code headers, and summary styling.

Reviewer-facing wording:
“conv_painter.cpp renders the compiled conversation document, including selection, clipboard copy, code block headers, syntax-highlighted visible code lines, summary styling, and per-code-block copy buttons.”

gui/conv/conv_syntax.cpp/h

Role: Lightweight syntax highlighter for visible code lines.

This file implements a small single-line lexer and span cache. It supports:

C/C++-style keywords
Python keywords
comments
strings
numbers
preprocessor-ish tokens
UTF-8-safe skipping for non-ASCII
per-scope cache keyed by:
scope key
build signature
document line index
line hash
language
palette colors

It is intentionally lightweight. It does not aim to be a full parser.

What this proves:
The chat transcript supports code readability without pulling in a heavy editor component.

Reviewer-facing wording:
“conv_syntax.cpp provides cached lightweight syntax spans for visible code lines, giving the custom transcript renderer practical C/C++/Python-style highlighting without a heavyweight editor dependency.”

gui/conv/conv_minimap.cpp/h

Role: Sublime-like conversation minimap.

This file renders a scaled glyph view of the conversation document and mutates ViewState::scrollY through minimap interactions.

It supports:

scaled rendering of visible document lines
role/code coloring
internal minimap scroll
viewport rectangle
drag-to-scroll
track paging
mouse wheel scrolling
smooth viewport animation
fit-case handling when minimap content fits but main view scrolls
per-document state keyed by doc.scopeKey
reset on document rebuild via buildSig

What this proves:
The conversation view has an editor-like navigation surface, not just a scrollbar.

Reviewer-facing wording:
“conv_minimap.cpp renders a Sublime-style minimap for the conversation document, including a scaled glyph preview, viewport thumb, wheel/drag/track interactions, and scroll synchronization with the main transcript.”

gui/conv/conv_fence_parser.cpp/h

Role: Deterministic fence/file-header parser.

This module parses messages into UTF-16 text runs and fenced code blocks with language/file metadata. It:

sanitizes UTF-8
converts to UTF-16
detects fenced code blocks
tolerates leading whitespace before fences
parses language tokens
detects file headers:
// File:
# File:
#File:
supports file headers outside fences
supports file headers inside the first non-empty code line
supports file headers glued onto fence tails
emits FenceBlock ranges in flattened UTF-16 index space

Important nuance:
conversation_output.cpp currently routes through the conv_view / conv_painter line-buffer path, and conv_view has its own fence parsing logic. So conv_fence_parser appears to be either a supporting/legacy/separate parser or intended for another path. The overlap is worth tracking.

Risk note:
There are two fence/file-header parsers in this batch: conv_fence_parser and conv_view. That is not automatically wrong, but if both remain active, they need to stay behaviorally aligned so code block/file-label behavior does not drift.

Reviewer-facing wording:
“conv_fence_parser.cpp is a deterministic fenced-code/file-header parser that produces UTF-16 text runs and code-block ranges. It complements the line-buffer renderer but should be kept aligned with conv_view’s active fence parsing behavior.”

3. Panel layer
gui/panels/panel_prompts.cpp

Role: Prompt/work-mode management panel.

This is one of the stronger panel files. It is not merely placeholder UI.

It supports:

Work-mode prompt tab
Locked prompt tab
dropdown selection
full-width prompt editor
locked prompt viewer
import Markdown/text from file
create new custom prompt
save custom prompt
set active work-mode prompt on the active profile
create assistant profile from a new prompt
read built-in prompts from PromptTemplates
read custom prompts from Prompts::PromptStore
read locked prompts from LockedPrompts
supports variations in prompt store struct/API shape through templates/adapters

Important behaviors:

Built-in prompts are read-only.
Custom prompts can be edited/saved.
Locked prompts are view-only.
Create assistant creates a blank custom prompt and assistant profile.

What this proves:
Coeus has an in-app prompt/profile management surface. This supports the claim that the user can manage assistant behavior without editing source files.

Reviewer-facing wording:
“panel_prompts.cpp provides a full in-app prompt management surface, including built-in work-mode prompts, editable custom prompts, locked prompt viewing, Markdown import, active work-mode assignment, and assistant/profile creation.”

gui/panels/panel_ui_settings.cpp

Role: Full UI customization/settings panel.

This file provides a broad settings surface for:

renderer backend:
auto
CPU
OpenGL
scheduler/debug settings
resize refresh rates
resize render scale
frame colors
sidebar colors
tab colors
menu colors
editor colors
selection colors
accent colors
conversation role colors:
user text
assistant text
summary text
code text
typography:
conversation text size
code size
user input text size
minimap:
enabled
scale
width
min/max width

It writes theme overrides through Theme::SetOverride, clears them through Theme::ClearOverride, and saves via UISettings.

What this proves:
The GUI is configurable from inside the app. It has a real surface for renderer/scheduler tuning and visual customization.

Reviewer-facing wording:
“panel_ui_settings.cpp implements the in-app UI settings editor, exposing renderer/scheduler tuning, theme color overrides, conversation role colors, typography, and minimap configuration.”

gui/panels/panel_tools.cpp

Role: Tools/toolchains panel scaffold.

This panel is currently a compile-safe UI scaffold. It renders:

Tools / Toolchains mode buttons
catalog selector
JSON args editor
Run button
result preview
footer note explaining future backend wiring

But the file explicitly says backend calls to ToolRegistry, ToolRunner, ToolChainRegistry, and ToolChainExecutor are not wired yet.

What this proves:
There is a planned/visible surface for tool execution, but from this file alone it should be described as scaffolded rather than complete.

Reviewer-facing wording:
“panel_tools.cpp is the first-cut Tools/Toolchains panel scaffold. It provides the intended selector, JSON args editor, run button, and result preview layout, with backend registry/executor wiring marked for a later phase.”

gui/panels/panel_monitor.cpp

Role: Monitor panel scaffold for model-invisible streams.

This panel is also currently a placeholder/scaffold. It renders:

active thread summary
show activity toggle
show agent steps toggle
ActivityFeed box
RunTrace box
placeholder rows
footer note saying future wiring will use:
ActivityFeed::SnapshotLatest
RunIndex::Snapshot
RunTrace::Service::GetActive

Important conceptual value:

Monitor is meant for MODEL-INVISIBLE streams:
  ActivityFeed
  Agentic RunTrace

What this proves:
There is a planned dedicated monitor surface for backend/tool/agentic telemetry, but it is not fully wired in this file.

Reviewer-facing wording:
“panel_monitor.cpp defines the dedicated monitor panel for model-invisible activity and agentic run traces. In this batch it is a compile-safe UI placeholder, with backend telemetry wiring explicitly planned for a later phase.”

4. GUI service bridge
gui/services/non_transcript_tool_service.cpp/h

Role: GUI-safe bridge to non-transcript tools and toolchains.

This service wraps Tooling::NonTranscriptToolApi for the GUI. It exposes:

BeginAsyncToolRun(...)
BeginAsyncToolchainRun(...)

There are two overload sets:

Modern API:
  project_id
  tool/chain name
  args
  callback
  thread_id
  tier
  priority

Legacy ABI shim:
  project_id
  tool/chain name
  args
  callback

The implementation converts core tool/toolchain results into a GUI-friendly result:

ok
name
run_id
details_ref
activity_line
error_code
error_message
outputs_json
artifact_refs

Crucially, callbacks from the core API are marshaled back to the UI thread through UIFramework::PostToMainThread.

What this proves:
The GUI has a safe async service boundary for running non-transcript tools/toolchains without dumping them into chat history, while preserving details refs, artifacts, outputs, and activity lines.

Reviewer-facing wording:
“non_transcript_tool_service.cpp is the GUI bridge for non-transcript tool and toolchain runs. It wraps the core non-transcript tool API, converts core results into UI-safe result structs, preserves details/artifact refs, supports explicit thread/tier/priority routing, and marshals callbacks back to the UI thread.”

5. Sidebar helper
gui/sidebar/side_bar_helper.cpp/h

Role: Sidebar project/thread model builder and persistence helper.

This file owns sidebar data/state helpers, not the visual renderer itself.

It supports:

persisted sidebar state in config/sidebar_state.json
project manual order
sort mode:
manual
recent
pinned projects
pin toggling
drag reorder commits
project order sync to existing projects
project row model building
thread tree flattening
thread depth calculation
closed/open thread metadata
project UUID/created/modified/profile metadata
search matching
local time formatting
UUID shortening
UTF-8-safe ellipsizing
throttled telemetry ticks for threads via GuiTelemetry::TickThread

What this proves:
The sidebar is backed by a persistent model: project ordering, pinning, sorting, thread hierarchy, and telemetry polling are first-class.

Potential bug/risk note:
CountVisibleThreadRows accepts pid but currently comments that pid is unused and uses a fallback key with pid 0. The comment says caller should prefer pid-aware keys. That is a small risk area for thread-collapse state correctness.

Reviewer-facing wording:
“side_bar_helper.cpp builds the project/thread models for the sidebar, including persisted project order, pinned projects, manual/recent sort modes, flattened thread trees with branch depth, project metadata, search filtering, and throttled telemetry ticks.”

6. Telemetry aggregation
gui/telemetry/telemetry_ui.cpp/h

Role: GUI-side telemetry aggregator and EvidenceUI attachment seam.

This is probably the most architecturally important file in Block 3B.

The file clearly defines its role:

telemetry_ui.cpp is the GUI-side telemetry AGGREGATOR for a thread.

It reads stable backend surfaces:

ActivityFeed
RunIndex
RunTrace service
CLE turn manifests
CLE turn bundles
retrieval trace artifacts
EvidenceUI attachment caches

It exposes UI-safe structs:

ActivityLine
RunRef
PlannerCall
ActionEntry
Termination
RunTrace
ThreadSummary

It supports:

binding backend telemetry services
ticking a thread
warm-loading activity/run index state
incremental activity polling
run index snapshot polling
active run trace polling
cached thread summaries
run stop handles
friendly activity label derivation from tool/chain trace details
generic activity text rehydration once details_ref becomes available
CLE manifest loading:
.cle_traces/threads/pidX_tidY.json
CLE bundle loading
retrieval trace resolution from:
inline retrieval trace
top-level retrieval_debug_json
bundle["artifacts"]["rr"]
legacy aliases
fallback source/evidence extraction from bundles
turn meta extraction and attachment
evidence source extraction and attachment to EvidenceUI

The authority boundary is explicit:

telemetry_ui aggregates and presents telemetry truth for the UI.
It does not define prompt truth or evidence policy.

That is very important and aligns with the EvidenceUI boundary from Block 3A.

CLE evidence attachment flow

The rough flow:

TickThread(pid, tid)
  → ensure ActivityFeed / RunIndex loaded if services exist
  → poll activity/run index/run trace
  → check CLE manifest stamp
  → if changed, clear stale project evidence
  → read CLE turn manifest
  → read turn bundles
  → resolve retrieval trace
  → extract EvidenceItem rows
  → attach TurnMeta to EvidenceUI
  → attach source rows to EvidenceUI
Mapping CLE turns to messages

The file avoids mapping by epoch because epoch can reset/reuse across restarts. Instead it maps by:

A. user preview/query → matching user message → next assistant message
B. fallback chronological position

That is a mature detail.

What this proves:
The GUI evidence chips are not manually faked. There is a thread-scoped telemetry aggregation layer that reads persisted CLE traces/bundles/retrieval artifacts and attaches source evidence and turn meta back into the live chat surface.

Reviewer-facing wording:
“telemetry_ui.cpp is the GUI telemetry aggregation seam. It polls backend ActivityFeed, RunIndex, and RunTrace services, loads CLE manifests and bundles from .cle_traces, resolves retrieval debug artifacts, derives UI-safe activity/run/turn metadata, and attaches source evidence plus turn meta into EvidenceUI while preserving the boundary that UI aggregation does not define prompt or evidence authority.”

7. Main architecture flows proven by Block 3B
Conversation rendering flow
ConversationOutput
  → snapshots active thread messages
  → ConvView::BuildDoc
      - role lines
      - text lines
      - summaries
      - code blocks
      - wrapping
      - prefixY
  → ConvPainter::Render
      - visible lines
      - selection
      - syntax highlighting
      - copy buttons
  → ConvMiniMap::Render
      - scaled glyph map
      - scroll interactions

This is a strong “custom editor-like transcript” story.

Code block rendering flow
Assistant message text
  → ConvView detects fenced code
  → associates file header / language
  → creates CodeBlock metadata
  → ConvPainter draws header lane
  → ConvPainter draws Copy button
  → ConvView::ExtractCodeBlockUtf8 handles copy text
Syntax highlighting flow
ConvPainter visible code line
  → ConvSyntax::GetLineSpans(scopeKey, buildSig, lineIdx, line, lang, isCode)
  → cached lexer spans
  → painter draws colored byte ranges
Telemetry/evidence aggregation flow
GuiTelemetry::TickThread(pid, tid)
  → ActivityFeed / RunIndex / RunTrace polling
  → CLE manifest stamp check
  → CLE bundle loading
  → retrieval artifact resolution
  → evidence item extraction
  → EvidenceUI::AttachTurnMeta
  → EvidenceUI::Attach
  → ConversationOutput reserves and renders evidence panel
Non-transcript tool flow
GUI caller
  → Gui::NonTranscriptToolService::BeginAsyncToolRun / BeginAsyncToolchainRun
  → Tooling::NonTranscriptToolApi
  → worker callback
  → UIFramework::PostToMainThread
  → GUI callback receives NonTranscriptToolResult
Sidebar model flow
ProjectManager::getAllProjects
  → sync persisted project order
  → apply pin/sort/search
  → snapshot project/thread metadata
  → flatten thread tree with depth
  → return ProjectRow[]
8. What this proves about Coeus
Proven: custom transcript engine

Safe claim:

Coeus uses a custom line-buffer conversation renderer with structured document building, soft wrapping, code block metadata, selection/copy, syntax highlighting, summary nodes, and minimap support.

This is stronger than saying “it has a chat UI.”

Proven: model-visible and model-invisible streams are separated

telemetry_ui.h says the GUI exposes UI-safe telemetry structs and does not insert trace JSON into transcript history. panel_monitor.cpp is also explicitly for model-invisible streams.

Safe claim:

Coeus separates transcript-visible conversation from model-invisible telemetry/activity/run-trace streams.

Proven: evidence chips are backed by persisted CLE traces

Safe claim:

EvidenceUI source rows can be attached from persisted CLE manifests, turn bundles, and retrieval trace artifacts, not only transient in-memory retrieval results.

That is a strong durability/reproducibility point.

Proven: the GUI can survive backend evolution

Several files include compatibility seams:

panel_prompts.cpp uses template adapters for varied PromptStore API shapes.
non_transcript_tool_service keeps legacy ABI overloads.
telemetry_ui.cpp tolerates multiple retrieval artifact shapes.
conv_syntax caches per build signature.
side_bar_helper tolerates missing config and syncs order to existing projects.

Safe claim:

The GUI layer includes compatibility seams for evolving backend APIs and persisted artifact shapes.

9. Complexity / risk notes
telemetry_ui.cpp is a major complexity hotspot

It does a lot:

service polling
cache management
activity dedup/sort
run index repair
friendly label derivation
CLE manifest loading
bundle loading
retrieval artifact resolution
fallback source extraction
evidence attachment
turn meta attachment

This is justified, but it is a high-coordination seam.

Good internal note:

telemetry_ui.cpp should remain a UI aggregation seam only.
It must not become an alternate source of prompt/evidence truth.
panel_tools.cpp and panel_monitor.cpp are scaffolds

They are useful product-direction evidence, but do not oversell them as complete backend-integrated panels yet.

Safe wording:

Tools Panel: scaffolded layout, backend wiring pending.
Monitor Panel: scaffolded model-invisible stream viewer, backend wiring pending.
Fence parsing duplication should be watched

conv_fence_parser and conv_view both understand fences and file headers. If both remain used, define which one is canonical or add tests to keep behavior aligned.

Sidebar collapse helper has a noted keying caveat

CountVisibleThreadRows comments that pid is unused and raw thread-id fallback is used. That may be fine depending on caller behavior, but for robust multi-project thread collapse, pid-aware keys should be enforced consistently.

10. Notes to carry into CODEBASE_OVERVIEW.md

Use this:

The second half of the GUI layer contains the renderer and aggregation infrastructure behind the visible workspace. The `gui/conv/*` modules implement a custom line-buffer conversation renderer: `conv_view` compiles thread messages into a structured document with role lines, summaries, code blocks, wrapping, prefix-height tables, selection, and copy extraction; `conv_painter` renders visible lines, selections, code headers, copy buttons, and syntax-highlighted code spans; `conv_minimap` provides a Sublime-style conversation minimap; and `conv_syntax` provides lightweight cached code highlighting. The `panels/*` files add prompt management, UI settings, tools, and monitor surfaces. `non_transcript_tool_service` bridges GUI requests to non-transcript tool/toolchain execution while marshaling callbacks back to the UI thread. `side_bar_helper` builds persisted project/thread sidebar models. `telemetry_ui` is the GUI telemetry aggregation seam, polling ActivityFeed/RunIndex/RunTrace and attaching CLE-derived turn metadata and source evidence from persisted manifests, bundles, and retrieval artifacts into EvidenceUI.
11. Notes to carry into ARCHITECTURE_NOTES.md

Use this map:

Block 3B GUI infrastructure map

Conversation renderer:
  conv_view
    message snapshot -> DocLine[]
    role lines / text / summary / code / spacers
    soft wrap / no-wrap code
    code block metadata
    selection / hit testing / extraction

  conv_painter
    paints ConvView::Doc
    selection highlights
    code backgrounds
    summary styling
    syntax spans
    code copy buttons

  conv_syntax
    lightweight visible-line lexer
    cached spans by scope/buildSig/line/lang/palette

  conv_minimap
    scaled transcript preview
    viewport thumb
    drag/wheel/track scrolling

  conv_fence_parser
    deterministic UTF-16 fence/file-header parser
    potential overlap with conv_view fence parsing

Panels:
  panel_prompts
    built-in/custom/locked prompt management
    Markdown import
    active work-mode setting
    assistant/profile creation

  panel_ui_settings
    renderer/scheduler settings
    theme tokens
    conversation role colors
    typography
    minimap settings

  panel_tools
    tools/toolchains scaffold
    JSON args editor + result preview
    backend wiring pending

  panel_monitor
    model-invisible activity/runtrace scaffold
    backend wiring pending

Services:
  non_transcript_tool_service
    GUI-safe async bridge to NonTranscriptToolApi
    explicit thread/tier/priority routing
    UI-thread callback marshal

Sidebar:
  side_bar_helper
    persisted project order
    pinned projects
    manual/recent sort
    thread tree flattening
    throttled telemetry tick

Telemetry:
  telemetry_ui
    ActivityFeed / RunIndex / RunTrace aggregation
    CLE manifest + bundle loading
    retrieval artifact resolution
    EvidenceUI TurnMeta + source attachment
    not evidence/prompt authority
12. Portfolio-facing summary

Use this shorter version:

Block 3B contains the infrastructure that makes the Coeus desktop GUI feel like a real developer workspace. The conversation pane is backed by a custom line-buffer document model, painter, syntax highlighter, and minimap rather than a generic text widget. The GUI also includes prompt management, UI settings, tool/monitor panel scaffolds, non-transcript tool execution bridging, persistent sidebar project/thread modeling, and a telemetry aggregation layer that reads CLE manifests, turn bundles, retrieval artifacts, ActivityFeed, RunIndex, and RunTrace data to attach source evidence and turn metadata into the chat UI.

Stronger version:

The Block 3B source shows that Coeus AI has a custom editor-grade conversation renderer and a telemetry-backed evidence pipeline. Messages are compiled into a structured line document with summaries, code blocks, wrapping, selection, copy extraction, syntax highlighting, and minimap navigation. Separately, GUI telemetry aggregation reads persisted CLE traces and backend run/activity services to attach source evidence and turn metadata into the live chat, while preserving the boundary that UI display does not define prompt or evidence authority.