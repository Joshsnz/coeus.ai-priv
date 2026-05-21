Block 2 Summary — GUI Foundation, Rendering, and Platform Layer

Block 2 proves the native GUI layer is a substantial authored subsystem. This is not a thin wrapper around an existing UI toolkit. The source shows a custom SDL2 + Skia immediate-mode interface, a pane compositor, render-backend abstraction, CPU and optional OpenGL/Ganesh rendering paths, frame scheduling, custom Windows chrome, app-level shortcuts, panel registration, theme/settings persistence, text editing, rich code-block rendering, and GUI-to-runtime service binding.

The strongest portfolio-level point is:

Coeus AI implements its own native desktop UI shell in C++ using SDL2 for window/events, Skia for drawing, a custom pane compositor for cached rendering, selectable CPU/OpenGL render backends, a frame scheduler for targeted redraws, and Windows-specific custom window chrome.

This block is one of the strongest pieces of evidence that Coeus is a real native C++ desktop application rather than a web app or chatbot wrapper.

1. Batch summary

Block 2 establishes the foundation of the visible application:

main.cpp
  → InitGuiMain()
      → SDL window creation
      → render backend selection
      → AppBar/UIFramework/UIFx initialization
      → telemetry service binding
      → main-thread task wake hook
      → tab/sidebar/evidence callback setup
  → RunGuiMain()
      → SDL event loop
      → app shortcuts
      → pane invalidation
      → frame scheduler
      → pane recording
      → compositing
      → backend present
  → ShutdownGuiMain()
      → clear wake hook
      → uninstall Win32 chrome
      → shutdown UI framework/appbar/backend/window/SDL

The architecture is split into several layers:

App-level GUI policy
  app_actions
  app_shortcuts
  panel_registry / panel_context

Core immediate-mode UI layer
  ui_framework
  ui_text_edit
  ui_richtext
  ui_theme
  ui_theme_runtime
  ui_settings

Pane compositor
  ui_compositor

Render abstraction
  render_backend_iface
  render_backend        CPU/SDL texture upload path
  render_backend_gl     optional OpenGL/Ganesh path
  frame_scheduler       targeted redraw/present policy

Platform integration
  win32_window_chrome

For the portfolio overview, Block 2 supports three major claims:

Native desktop application
SDL2 window/event loop.
Skia drawing.
Custom borderless window and Win32 chrome.
CPU/OpenGL renderer abstraction.
Custom product UI
App bar, sidebar, tab bar, command palette, panels, overlays, conversation dock, project/thread shortcuts.
Immediate-mode widgets, text input, rich code blocks, virtualized lists.
Performance-conscious rendering
Cached SkPicture panes.
Dirty-pane invalidation.
Frame scheduler.
Resize/move handling.
Cached presents.
HiDPI-aware backend sizing.
2. File-by-file role map
gui/gui_main.cpp

Role: Central GUI lifecycle and event/render loop.

This file implements the three GUI functions declared in main.cpp:

bool InitGuiMain(int width, int height, bool fullscreen);
int  RunGuiMain();
void ShutdownGuiMain();

It owns the native GUI boot path:

initializes SDL video/timer systems
creates the SDL window titled "Coeus AI"
enables drag/drop events
selects CPU/OpenGL render backend
initializes custom Win32 chrome
creates system cursors
initializes AppBar and UIFramework
binds telemetry UI to:
ActivityFeed
RunIndex
RunTrace::Service
installs a main-thread wake hook using Tasking::MainThread::SetWakeFn
initializes the pane compositor layout
loads tab UI state
registers evidence callbacks

RunGuiMain() is the core GUI loop. It handles:

SDL event processing
resize and native window state tracking
sidebar resizing
drag-resize behavior
command palette toggle
global shortcuts
main-thread task draining
active tab selection
pane dirty-state tracking
frame-scheduler signal assembly
selective pane recording
compositing cached panes
backend present

The visible tab rendering is dispatched by activeTabType:

0 → conversation output
1 → context output
2 → profile editor
3 → provider editor
4 → file context manager
5 → memory map
6 → settings panel

What this proves:
The app has a real native GUI loop with custom event handling, pane scheduling, multiple product surfaces, telemetry binding, tab routing, and render-backend control.

Reviewer-facing wording:
“gui_main.cpp is the native GUI orchestrator. It creates the SDL window, initializes the Skia render backend, installs platform window chrome, binds telemetry services, runs the event loop, coordinates pane invalidation, and renders the active app surface through a scheduled pane compositor.”

gui/app_actions.cpp/h

Role: Unified UI command surface for app-level actions.

This layer keeps app components thin. Instead of Sidebar/AppBar/Shortcuts/Command Palette each creating projects differently, they call UIActions.

Currently this block proves:

default project titles are generated in a Windows-safe date format
new project creation delegates to ProjectActions::AttemptNewProject(title)
the active project ID is read before/after creation
the new project opens the conversation tab through TabBar::OpenConversation(newPid)
optional inline rename is delegated through a registered callback

Important architectural boundary:
app_actions is UI-aware but delegates actual project creation to core ProjectActions. That is a good separation:

UIActions
  owns UI follow-up behavior

ProjectActions
  owns core project mutation

What this proves:
There is deliberate separation between UI entrypoints and core project operations.

gui/app_shortcuts.cpp/h

Role: Central app-level shortcut policy.

This handles global shortcuts independently of low-level UI widget editing.

Supported shortcuts include:

Ctrl+Tab / Ctrl+Shift+Tab     cycle tabs
Ctrl+PageUp / Ctrl+PageDown   cycle tabs
Ctrl+W / Ctrl+F4              close active tab
Ctrl+N                        new project
Ctrl+T                        new chat/thread
Ctrl+B                        branch from active thread
Ctrl+Shift+P                  handled through UIFramework pulse into command palette

It calls:

UIActions::NewProject
ConversationManager::CreateThread
ConversationManager::BranchFromActiveThread
TabBar::OpenConversation
SideBar::BeginThreadInlineRename
UIFramework::Invalidate*

What this proves:
The app has desktop-style workflow semantics: tabs, projects, threads, branches, command palette, and keyboard-driven navigation.

Reviewer-facing wording:
“App-level shortcuts are centralized outside widget plumbing, which keeps SDL input handling separate from product semantics like new project, new thread, branch thread, close tab, and tab cycling.”

gui/panel_context.h

Role: Data model for panel tabs.

PanelContext describes a live panel instance rendered inside a tab. It includes:

panel_id
title
init payload
stable instance_key
optional owner_id
close/rename requests
last viewport size

It defines a generic render function:

using RenderFn = void(*)(PanelContext& ctx, float x, float y, float w, float h);

What this proves:
The UI supports generic panel tabs, not just hardcoded conversation screens. This matters for tools, monitor, prompts, and settings panels.

gui/panel_registry.cpp/h

Role: Registry of panel kinds.

Built-in registered panel types:

ui_settings
prompts
tools
monitor

Each panel definition includes:

stable ID
title
description
allowMultiple

The comments show current intended use:

prompts → work-mode prompts, custom prompts, locked previews
tools → Tier-1 runner/catalogs/results/details refs
monitor → ActivityFeed + RunTrace overlays

What this proves:
The GUI shell has an extensible panel system for internal tooling, monitoring, prompt management, and settings. This supports the claim that Coeus is a workspace-style application, not just one chat view.

gui/ui_compositor.cpp/h

Role: Pane compositor built on Skia SkPicture.

This is one of the most important GUI foundation files.

It defines panes:

AppBar
Sidebar
Tabbar
Main
Overlays

Each pane has:

bounds
cached SkPicture
dirty flag
timing instrumentation

The compositor records panes independently with:

BeginPane(PaneId id)
EndPane(PaneId id)

Then composites all cached pane pictures in order:

AppBar → Sidebar → Tabbar → Main → Overlays

Important design points:

panes are recorded in absolute window coordinates
resize updates bounds and marks panes dirty
cached pictures are retained across resize to avoid blank windows
compositing clips each cached pane to current bounds
per-pane record timings and composite timing are exposed through FramePerf

What this proves:
The app has a custom rendering strategy, not basic repaint-everything UI. The compositor supports targeted redraws and performance-aware scheduling.

Reviewer-facing wording:
“ui_compositor records each major UI pane into a cached Skia picture and composites those pictures in a stable z-order. Dirty-pane tracking lets the frame scheduler redraw only the hot panes instead of repainting the entire UI every frame.”

gui/ui_framework.cpp/h

Role: Immediate-mode UI framework and input state system.

This file is large because it owns a broad set of UI primitives and event-state handling.

Major responsibilities:

Frame scheduling signals
RequestFrame
ConsumeFrameRequest
frame reason bits
interaction pulses
wheel pulses
text input pulses
animation pulses

These pulses are consumed by gui_main and FrameScheduler so typing, scrolling, popups, and animation are not starved.

SDL event translation

Handles:

mouse motion
mouse button down/up
mouse wheel
window enter/leave
key down/up
text input
text editing / IME composition
Widget input state

Tracks:

focus widget
key pulses
mouse transitions
pointer coordinates
wheel delta
ID stack
active widget
modal capture
input clipping
Overlay handling
overlay queue
outside-click closers
popup/modal repaint invalidation
Main-thread queue

Separate from Tasking::MainThread, it also provides:

PostToMainThread
PumpMainThreadTasks
Utility widgets
integer stepper
smooth scrolling
virtualized list
Performance HUD

Tracks/draws a UI performance HUD, including:

frame time
overlay/dropdown counts
save counts per minute for profiles/providers/projects/conversation history

What this proves:
Coeus contains an authored immediate-mode GUI framework. It handles event translation, focus, text input, modals, overlays, invalidation, virtualized lists, scrolling, and performance instrumentation.

Reviewer-facing wording:
“ui_framework is the app’s custom immediate-mode UI layer over SDL2 and Skia. It translates SDL input into widget-level pulses, manages focus and modal capture, queues overlays, handles text input and frame requests, and exposes invalidation hooks into the pane compositor.”

gui/ui_richtext.cpp/h

Role: Rich code-block rendering and interaction.

This file renders no-wrap code blocks with:

language/file header
copy button
copied toast
horizontal scrollbar
tab expansion
selection drag
Ctrl+C selection copy
full-block copy
per-code-block state keyed by scope/id

This is directly relevant to a codebase-aware AI workspace because assistant responses and context surfaces need readable, copyable code blocks.

What this proves:
The UI includes custom code-display behavior, not just plain text rendering.

Reviewer-facing wording:
“ui_richtext implements interactive code blocks with selection, clipboard copy, horizontal scrolling, file/language headers, and per-block retained UI state.”

gui/ui_text_edit.cpp/h

Role: Advanced text field implementation.

This supports:

caret movement
UTF-8-aware previous/next codepoint navigation
selection
multiline mode
soft wrap
read-only mode
copy/cut/paste
undo/redo
held-key repeat for Backspace/Delete
horizontal and vertical scroll offsets
per-widget font size

What this proves:
The app does not rely entirely on OS-native text widgets. It implements its own text editing primitives suitable for Skia rendering.

Important nuance:
Caret positions are byte indices into UTF-8 strings, but movement/deletion uses UTF-8 codepoint helpers. That is a practical compromise.

gui/ui_settings.cpp/h

Role: Persistent UI configuration.

This owns ui_settings.json and supports:

conversation font sizes
user-input font size
UI text size/density
minimap controls
conversation colors
syntax highlighting colors
minimap viewport colors
theme override map
performance HUD toggle
renderer backend selection:
auto
cpu
opengl
scheduler settings:
target Hz
resize refresh Hz
chrome refresh Hz
resize render scale
scheduler debug
GPU polite mode

It validates/clamps loaded values and saves through debounced UI framework tasks.

What this proves:
The GUI has persistent runtime-tunable rendering/theme/scheduler behavior. This is stronger than hardcoded demo UI.

gui/ui_theme.h

Role: Compile-time theme constants.

Defines the default ARGB colors and font helpers used across UI components:

panel colors
text colors
buttons
tabs
menus
code blocks
summary blocks
badges
selection highlight
toasts

What this proves:
There is a consistent internal visual language for the custom GUI.

gui/ui_theme_runtime.cpp/h

Role: Runtime theme token layer.

This maps stable theme keys to color tokens, with human labels and effective color resolution.

Canonical token groups include:

Frame
Sidebar
Tabs
Menus
Editor
Selection
Accents

It also maintains legacy compatibility tokens and maps old tokens to newer surface tokens where appropriate.

Effective color resolution is:

exact override
  → legacy surface fallback override
  → compile-time default

What this proves:
The app supports runtime theme overrides using stable string keys stored in UI settings, while preserving backward compatibility.

gui/platform/win32_window_chrome.cpp/h

Role: Windows custom window chrome integration.

This is a substantial native-platform layer.

It:

hooks the Win32 window procedure for the SDL window
suppresses default non-client painting
manages borderless sizeable window styles
handles WM_NCHITTEST
handles maximized and half-snapped states
adjusts WM_GETMINMAXINFO
detects stuck SDL renderer swapchain sizes
ensures window remains visible on display changes
supports smooth dedock/native move after presenting a correct frame
dynamically enables high-resolution Windows timers through winmm.dll

What this proves:
The app includes platform-specific desktop integration, not just cross-platform SDL defaults.

Reviewer-facing wording:
“On Windows, Coeus installs a custom Win32 window procedure around the SDL window to support borderless chrome, native resize/move behavior, snap/maximize handling, and swapchain recovery.”

gui/render/frame_scheduler.cpp/h

Role: Targeted frame planning policy.

This file decides whether a frame should be:

Idle
CompositeOnly
HotPanes
Full

It consumes FrameSignals from gui_main, including:

dirty pane mask
missing picture mask
resize drag state
sidebar drag state
modal/menu/popup state
animation/wheel activity
pointer in main/appbar
native move state
GPU backend flag
present performance snapshot

It outputs FramePlan:

frame intent
record mask
force present flag
throttle Hz
continuation-frame request
debug reason bits

Important behavior:

Missing cached pictures force pane recording.
Resize/move favors cached compositing and limited hot-pane refresh.
GPU backends allow more frequent main-pane refresh during resize.
Optional GPU polite mode can reduce refresh under present stalls.
Wheel/modal/animation states request continuation frames.

What this proves:
The render loop is explicitly performance-aware. It does not naively redraw everything or blindly present every loop.

Reviewer-facing wording:
“FrameScheduler converts UI state, dirty panes, resize state, modal state, and performance signals into a concrete frame plan: idle, composite cached panes, redraw selected hot panes, or redraw the full UI.”

gui/render/render_backend_iface.h

Role: Render backend abstraction.

Defines the common interface for render backends:

class IRenderBackend

Key methods:

Init
Shutdown
EnsureBackbufferMatchesRendererOutput
RecreateRendererAndBackbuffer
Canvas
Present
PresentCached
ForceRendererStateSaneEveryFrame
GetSize
SDLRenderer
GetDiagnostics

Also defines backend diagnostics:

backend name
SDL renderer info
GL vendor/renderer/version
notes

What this proves:
Rendering is abstracted behind an interface, enabling CPU and OpenGL backend selection.

gui/render/render_backend.cpp/h

Role: CPU/SDL texture upload render backend.

This backend:

creates an SDL renderer
creates a streaming SDL texture
allocates pixel buffer
wraps pixels in a Skia surface
draws using Skia into CPU memory
uploads to SDL texture with SDL_UpdateTexture
presents with SDL_RenderCopy and SDL_RenderPresent
supports cached present without re-uploading
handles HiDPI output size
updates UI DPI scale
can recreate renderer/backbuffer

CpuRenderBackend adapts this implementation to IRenderBackend.

What this proves:
The app has a reliable CPU fallback rendering path. Even if GPU/Ganesh is not available, Skia rendering works through SDL texture upload.

gui/render/render_backend_gl.cpp/h

Role: Optional OpenGL/Ganesh render backend.

This backend is build-gated by:

RENDER_ENABLE_OPENGL_BACKEND

If enabled and compatible Skia Ganesh GL headers are available, it:

creates an SDL OpenGL context
creates a Skia Ganesh GrDirectContext
wraps the OpenGL framebuffer into an Skia surface
flushes Skia/GL work
swaps via SDL_GL_SwapWindow
exposes GL diagnostics

If not enabled, the factory returns empty and GUI falls back to CPU.

What this proves:
The app has an optional GPU rendering path while preserving a stable CPU fallback.

Important nuance:
From this source, we should say “optional OpenGL/Ganesh backend exists behind a build flag,” not “the public demo always uses OpenGL.” Actual active backend depends on build flags and runtime settings.

3. Main GUI/render control flow

The GUI boot flow is:

InitGuiMain(width, height, fullscreen)
  → SDL_Init
  → SDL_CreateWindow("Coeus AI")
  → Win32Chrome::Install(...)
  → SelectAndInitBackend()
      → CreateOpenGLBackend() if enabled/requested
      → fallback CreateCpuBackend()
  → AppBar::Init()
  → UIFramework::InitUIFramework()
  → GuiTelemetry::BindServices(...)
  → Tasking::MainThread::SetWakeFn(UIFramework::RequestFrame)
  → UIFx::Init(...)
  → RegisterEvidenceCallbacks()
  → TabBar::LoadUIState()
  → RequestFrame()

The per-frame render flow is:

RunGuiMain loop
  → wait/poll SDL events
  → UIFramework::BeginFrame()
  → ProcessEvent(...)
      → UIFramework::HandleSDLEvent(...)
      → AppShortcuts::HandleKeyDown(...)
      → AppBar / drag / resize / drop / command palette handling
  → apply resize/layout changes
  → drain Tasking::MainThread queue
  → pump UIFramework main-thread tasks
  → collect active tab state
  → build FrameSignals
  → FrameScheduler::MakePlan(signals, perf)
  → if Idle: skip present
  → ensure backend backbuffer/drawable
  → record selected dirty/hot panes into SkPictures
  → CompositeAllTo(windowCanvas)
  → backend Present()
  → UIFramework::EndFrame()

Pane recording flow:

FramePlan.record_mask
  → AppBar pane       AppBar::RenderBar
  → Sidebar pane      SideBar::RenderSidebar
  → Tabbar pane       TabBar::RenderTabBar
  → Main pane         active tab content
  → Overlays pane     user input dock, menus, popups, command palette

This is a strong native application architecture story.

4. How GUI state connects to backend/runtime

This block shows several integration points:

Project/runtime integration
UIActions calls ProjectActions::AttemptNewProject.
app_shortcuts reads ProjectManager::getActiveProjectID.
app_shortcuts calls ConversationManager::CreateThread and BranchFromActiveThread.
gui_main uses ProjectManager for active project fallback in settings panel.
Telemetry integration

gui_main binds GUI telemetry to:

Tooling::ActivityFeed::instance()
Tooling::RunIndex::instance()
Telemetry::RunTrace::Service::instance()

This proves the UI foundation is aware of telemetry/debug surfaces before we inspect the actual telemetry panels.

Task integration

gui_main installs:

Tasking::MainThread::SetWakeFn([]{
    UIFramework::RequestFrame();
});

This connects background task completion to the UI event loop.

Evidence integration

RegisterEvidenceCallbacks_() registers an evidence open handler with EvidenceUI.

The implementation is currently minimal in this batch, but it proves a callback boundary exists between evidence UI and project/file context.

5. What this proves about the project

Block 2 supports these portfolio claims:

Native C++ desktop application

Source evidence:

SDL2 window creation and event loop.
Skia drawing through SkCanvas.
custom render backend interface.
CPU SDL texture backend.
optional OpenGL/Ganesh backend.
Win32 custom window procedure/chrome.

Safe claim:

Coeus AI is a native C++ desktop application using SDL2 and Skia, with custom rendering infrastructure and platform-specific Windows chrome handling.

Custom workspace UI

Source evidence:

app bar
sidebar
tab bar
command palette
generic panel tabs
settings/tools/prompts/monitor panel registry
code block renderer
custom text editor
overlays/popups/modals
performance HUD

Safe claim:

The UI is a custom workspace shell with tabs, panels, shortcuts, command palette, conversation surfaces, project/thread actions, and diagnostic panels.

Performance-aware rendering

Source evidence:

SkPicture pane caching.
dirty-pane invalidation.
scheduler frame intents.
targeted hot-pane recording.
composite-only frames.
cached present.
resize-specific refresh policy.
GPU/CPU backend awareness.

Safe claim:

The GUI uses a targeted rendering model: panes are cached and only dirty/hot panes are re-recorded, with scheduler policy adapting to resize, modal, input, and backend conditions.

Runtime/debug integration

Source evidence:

telemetry service binding.
ActivityFeed/RunIndex/RunTrace references.
panel registry includes monitor/tools/prompts.
perf HUD and frame diagnostics.
backend diagnostics.

Safe claim:

The GUI foundation includes hooks for telemetry, trace inspection, tools, prompt management, and runtime monitoring.

6. Architecture strengths
Strong separation between shell, framework, compositor, and backend

The code separates:

gui_main
  orchestration/event loop

ui_framework
  input/widget state/immediate-mode UI helpers

ui_compositor
  pane caching/composition

frame_scheduler
  frame policy

render_backend_iface
  backend abstraction

render_backend / render_backend_gl
  concrete presentation paths

This is a professional structure. The frame scheduler does not draw. The compositor does not decide policy. The backend does not own product UI.

Good native-app details

The code handles several issues that show real desktop-app work:

HiDPI output size
Windows snap/maximize behavior
custom borderless resizing
SDL renderer swapchain mismatch recovery
text input starvation avoidance
overlay invalidation under pane caching
drag/drop event support
cached presents during resize
wake events for background tasks

These are not typical toy/demo details.

UI state is persistent and configurable

ui_settings includes renderer/backend choice, font sizes, theme overrides, minimap sizing, scheduler settings, and debug flags. This makes the GUI tunable.

Custom code-centric rendering

ui_richtext and ui_text_edit support code blocks, copy, selection, UTF-8 movement, multiline fields, soft wrap, and undo/redo. This is relevant to codebase-aware assistant use.

7. Complexity / risk areas to mention carefully

These should not be presented as refactor demands, but they are fair reviewer-aware notes.

gui_main.cpp is a large orchestrator

It owns many responsibilities:

SDL event processing
window resize/move
active tab dispatch
frame-scheduler setup
pane recording
backend present
platform-specific details
telemetry binding

This is understandable for a native GUI main loop, but it is a complexity hotspot.

Professional wording:

“gui_main.cpp is intentionally the central GUI orchestrator and therefore carries high coordination complexity. The surrounding files reduce that risk by separating widget state, compositing, frame policy, render backend, and platform chrome into separate modules.”

Heavy use of forward declarations

The GUI layer frequently forward-declares functions from TabBar, SideBar, AppBar, etc. This reduces include coupling, but it also means the module boundaries are convention-based rather than strongly expressed through headers everywhere.

Professional wording:

“The GUI shell uses forward declarations to avoid broad include coupling. This keeps compile dependencies lighter but places more responsibility on naming and convention.”

Some compatibility aliases remain

PaneId::Dashboard is a back-compat alias for PaneId::AppBar. This is not a problem but indicates the UI evolved from earlier naming.

Professional wording:

“The compositor retains a few compatibility aliases from earlier UI naming, such as Dashboard/AppBar, but the current canonical pane is AppBar.”

OpenGL backend is optional

Do not oversell it as always active. The code proves the optional backend exists behind RENDER_ENABLE_OPENGL_BACKEND, with CPU fallback.

UI framework is custom, not a complete general-purpose toolkit

This is a purpose-built immediate-mode framework for this application. That is good for portfolio positioning, but avoid presenting it as a standalone production UI library.

8. Reviewer questions and answers
Q: Is the GUI actually native?

A:
Yes. gui_main.cpp creates an SDL2 window, handles SDL events, draws through Skia canvases, manages native cursors and drag/drop, and installs Windows-specific custom chrome. Rendering is handled through native CPU/OpenGL backends rather than browser UI.

Q: What rendering backend does the app use?

A:
The app uses a render backend abstraction. The CPU backend renders with Skia into a CPU pixel buffer, uploads it to an SDL texture, and presents through SDL. An optional OpenGL/Ganesh backend exists behind a build flag and can wrap the GL framebuffer as a Skia surface. Runtime settings can select auto, cpu, or opengl, with fallback behavior.

Q: How does the app avoid redrawing the whole UI constantly?

A:
The UI is divided into panes: AppBar, Sidebar, Tabbar, Main, and Overlays. Each pane is recorded into a cached Skia picture. The frame scheduler consumes dirty masks, interaction state, resize state, modal state, and backend/performance signals to decide whether to idle, composite cached panes, redraw hot panes, or redraw everything.

Q: Does the GUI have real app workflow, or just a chat box?

A:
The GUI has app-level actions and shortcuts for projects, threads, branches, tabs, command palette, panels, provider/profile/settings surfaces, telemetry, tools, and monitor panels. The conversation surface is one tab type within a broader workspace shell.

Q: How does background work update the UI?

A:
The GUI installs a wake function into Tasking::MainThread, so background tasks can post work and request a frame. The GUI loop drains main-thread tasks with a budget to avoid frame hitches.

Q: Are telemetry/debug panels part of the GUI?

A:
Yes. gui_main.cpp binds GUI telemetry services to ActivityFeed, RunIndex, and RunTrace::Service, and panel_registry registers a monitor panel for ActivityFeed and RunTrace views. The actual panel implementation appears in later blocks.

9. Notes to carry into CODEBASE_OVERVIEW.md

Use this:

The GUI layer is a custom native SDL2 + Skia desktop shell. `gui_main.cpp` owns the SDL window lifecycle, event loop, render backend selection, platform chrome installation, telemetry binding, pane scheduling, and tab-surface dispatch. The UI is divided into cached Skia picture panes — AppBar, Sidebar, Tabbar, Main, and Overlays — which are selectively re-recorded and composited according to a frame scheduler. Rendering is abstracted behind `IRenderBackend`, with a CPU/SDL texture backend and an optional OpenGL/Ganesh backend.

The GUI is not a simple chat window. It includes app-level shortcuts, a command palette, tabs, project/thread actions, generic panels, runtime settings, theme overrides, custom text editing, rich code-block rendering, telemetry binding, and monitor/tools/prompts panel registration.
10. Notes to carry into ARCHITECTURE_NOTES.md

Carry this architecture map forward:

GUI foundation architecture

main.cpp
  → MyApp::InitGuiMain / RunGuiMain / ShutdownGuiMain

gui_main.cpp
  owns SDL lifecycle, window events, tab dispatch, render loop, telemetry binding

UIFramework
  immediate-mode input/widget layer
  focus, modal capture, overlays, text input, frame requests, virtual lists

UIFx / ui_compositor
  pane cache and composition layer
  panes: AppBar, Sidebar, Tabbar, Main, Overlays

FrameScheduler
  converts UI signals + perf snapshot into frame plans
  intents: Idle, CompositeOnly, HotPanes, Full

IRenderBackend
  backend abstraction

CpuRenderBackend
  Skia CPU surface → SDL texture upload → SDL present

OpenGLRenderBackend
  optional Skia Ganesh/OpenGL backend behind build flag

Win32Chrome
  custom Windows borderless chrome, hit testing, snap/maximize handling

UISettings + Theme
  persistent visual/render/scheduler config
  runtime theme override tokens

AppActions/AppShortcuts
  app-level project/thread/tab command policy

PanelRegistry/PanelContext
  generic panel-tab model for settings, prompts, tools, monitor
11. Short portfolio-facing summary

Use this version:

Block 2 establishes Coeus AI’s native GUI foundation. The app uses SDL2 for windowing and events, Skia for custom drawing, a pane compositor for cached AppBar/Sidebar/Tabbar/Main/Overlay rendering, and a frame scheduler that redraws only dirty or hot panes. Rendering is abstracted behind CPU and optional OpenGL/Ganesh backends, with runtime settings for backend selection and scheduler behavior. The GUI shell includes app-level shortcuts, project/thread actions, tabs, command palette support, generic panels, custom text editing, rich code-block rendering, theme/runtime settings, telemetry binding, and Windows-specific custom window chrome.

Stronger version:

The GUI source proves that Coeus AI is a real native C++ desktop workspace, not a browser-based chatbot wrapper. Its SDL2/Skia interface includes a custom event loop, pane-cached compositor, targeted frame scheduler, CPU/OpenGL render backend abstraction, Win32 borderless-window integration, persistent UI settings, and workspace surfaces for tabs, panels, tools, prompts, telemetry, provider/profile editing, and conversation interaction.