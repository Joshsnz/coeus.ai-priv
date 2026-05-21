# GUI, Rendering, and Product Surface

## Why this matters

The GUI blocks prove that Coeus is a substantial authored native desktop application. It is not Electron, not a webview, and not a thin chat box. It uses a custom C++ SDL2/Skia interface, render backend abstraction, pane compositor, frame scheduler, custom Windows chrome, input/editor surfaces, conversation rendering, context/evidence displays, panels, telemetry UI, and runtime-service binding.

## GUI lifecycle

The desktop app follows a three-function GUI lifecycle called from `main.cpp`:

```text
InitGuiMain(width, height, fullscreen)
RunGuiMain()
ShutdownGuiMain()
```

A simplified flow:

```text
InitGuiMain
→ SDL window creation
→ render backend selection
→ AppBar/UIFramework/UIFx initialization
→ telemetry service binding
→ main-thread task wake hook
→ tab/sidebar/evidence callbacks

RunGuiMain
→ SDL event loop
→ shortcuts
→ pane invalidation
→ frame scheduler
→ pane recording
→ compositing
→ backend present

ShutdownGuiMain
→ clear wake hook
→ uninstall Win32 chrome
→ shutdown UI framework/appbar/backend/window/SDL
```

## Render architecture

The render layer is a key source of native-app credibility.

Relevant files:

```text
gui/render/frame_scheduler.cpp
gui/render/frame_scheduler.h
gui/render/render_backend.cpp
gui/render/render_backend.h
gui/render/render_backend_gl.cpp
gui/render/render_backend_gl.h
gui/render/render_backend_iface.h
```

The render design includes:

- render backend interface
- CPU/SDL texture upload path
- optional OpenGL/Ganesh path
- frame scheduler
- targeted dirty-pane redraws
- cached pane composition

This means the GUI is not a naive redraw loop. The source review indicates a performance-conscious rendering model: pane invalidation, SkPicture-style cached drawing, dirty regions, scheduler/pacer, and backend presentation.

## Core UI layer

Relevant files:

```text
gui/ui_framework.cpp
gui/ui_framework.h
gui/ui_text_edit.cpp
gui/ui_text_edit.h
gui/ui_richtext.cpp
gui/ui_richtext.h
gui/ui_theme.h
gui/ui_theme_runtime.cpp
gui/ui_theme_runtime.h
gui/ui_settings.cpp
gui/ui_settings.h
```

Main responsibilities:

- immediate-mode UI primitives
- input/event state
- text editing
- rich text and code-block rendering
- theme tokens and runtime theme settings
- persisted UI settings

This matters because Coeus needs to display code, evidence, traces, messages, panels, and long technical answers. A codebase-aware assistant needs stronger UI primitives than a plain text area.

## Pane compositor

Relevant files:

```text
gui/ui_compositor.cpp
gui/ui_compositor.h
```

The compositor supports cached pane rendering and composition. This is part of the performance story and helps explain how the app can have multiple live UI surfaces—conversation, context, sidebars, telemetry, panels—without treating the whole window as one constantly redrawn surface.

## App policy and actions

Relevant files:

```text
gui/app_actions.cpp
gui/app_actions.h
gui/app_shortcuts.cpp
gui/app_shortcuts.h
gui/panel_context.h
gui/panel_registry.cpp
gui/panel_registry.h
```

These files keep app-level actions and global shortcuts separate from low-level UI widget logic. App actions become a boundary between UI events and runtime operations. Shortcuts are handled independently of widget editing logic.

## GUI components

Relevant files:

```text
gui/components/app_bar.cpp
gui/components/command_palette.cpp
gui/components/context_output.cpp
gui/components/conversation_output.cpp
gui/components/dashboard.cpp
gui/components/evidence_box.cpp
gui/components/file_context_manager.cpp
gui/components/memory_map.cpp
gui/components/profile_editor.cpp
gui/components/provider_editor.cpp
gui/components/settings_panel.cpp
gui/components/side_bar.cpp
gui/components/tab_bar.cpp
gui/components/trace_file_viewer.cpp
gui/components/user_input.cpp
```

These files prove the app is a workspace, not a single chat screen. Important product surfaces include:

- app bar and sidebar
- tab management
- conversation output
- context output
- evidence box
- user input dock
- dashboard
- provider editor
- profile editor
- memory map
- trace viewer
- settings
- command palette

## Conversation rendering

Relevant files:

```text
gui/conv/conv_view.cpp
gui/conv/conv_painter.cpp
gui/conv/conv_syntax.cpp
gui/conv/conv_fence_parser.cpp
gui/conv/conv_minimap.cpp
```

The conversation subsystem is more than plain message display. It handles fenced blocks, syntax-ish rendering, minimap/painting, and conversation view mechanics. This is important because source-grounded answers often contain code, file paths, trace excerpts, evidence tables, and structured technical output.

## Telemetry UI

Relevant files:

```text
gui/telemetry/telemetry_ui.cpp
gui/telemetry/telemetry_ui.h
```

The telemetry UI connects the runtime’s trace/debug/eval substrate to the product surface. This supports the claim that Coeus has structured observability, not just hidden logs.

## Platform integration

Relevant files:

```text
gui/platform/win32_window_chrome.cpp
gui/platform/win32_window_chrome.h
```

These files support native Windows chrome integration, borderless/custom window behavior, and platform-specific polish.

## Reviewer talking points

If asked why the GUI matters:

> Coeus is a native desktop workspace. The GUI is authored in C++ with SDL2/Skia and has its own rendering, composition, input, panels, code rendering, telemetry, and context surfaces. That is a different engineering problem from embedding a chat widget in a webpage.

If asked why not web/Electron:

> The project was built as a systems/desktop AI workspace. Native C++ gave direct control over UI rendering, local runtime state, filesystem/project integration, telemetry, and app packaging. It also demonstrates native systems work rather than only frontend/web integration.

## Risk and refactor notes

The GUI is naturally complex because it owns many product surfaces. Risk areas to watch:

- large orchestration files such as `gui_main.cpp`
- pane invalidation and frame scheduling correctness
- UI/editor state interactions
- potential coupling between components and runtime state
- numeric tab or panel identifiers if not centralized
- difference between debug-only telemetry and release behavior

These are normal risks for a custom native UI layer and should be framed as review/refactor topics, not evidence of failure.
