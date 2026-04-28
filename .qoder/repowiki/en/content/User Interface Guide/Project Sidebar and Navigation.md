# Project Sidebar and Navigation

<cite>
**Referenced Files in This Document**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [input.rs](file://src/tui/input.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [main.rs](file://src/main.rs)
- [server.rs](file://src/mcp/server.rs)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)

## Introduction
This document explains the AGTX project sidebar and navigation system with a focus on multi-project management and workspace switching. It covers the sidebar layout, project selection mechanisms, keyboard navigation (j/k), project activation (Enter), sidebar visibility controls (e), and the dashboard versus project-specific modes. It also documents the relationship between projects and tmux sessions, project recovery for lost sessions, and cleanup procedures.

## Project Structure
The sidebar and navigation functionality is implemented in the TUI module, with tight integration to tmux session management and database-backed project/task state.

```mermaid
graph TB
subgraph "TUI Layer"
APP["App (app.rs)"]
BOARD["BoardState (board.rs)"]
INPUT["InputMode (input.rs)"]
SHELL["ShellPopup (shell_popup.rs)"]
end
subgraph "System Integration"
TMUX_MOD["tmux/mod.rs"]
TMUX_OPS["tmux/operations.rs"]
MAIN["main.rs"]
MCP["mcp/server.rs"]
end
APP --> BOARD
APP --> INPUT
APP --> SHELL
APP --> TMUX_MOD
APP --> TMUX_OPS
MAIN --> APP
MCP --> APP
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [input.rs](file://src/tui/input.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [main.rs](file://src/main.rs)
- [server.rs](file://src/mcp/server.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [input.rs](file://src/tui/input.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [main.rs](file://src/main.rs)
- [server.rs](file://src/mcp/server.rs)

## Core Components
- Sidebar and project list: renders a list of tracked projects, supports keyboard navigation (j/k), Enter to open, and e to hide/show.
- Board navigation: Kanban board with column/row selection and keyboard controls (h/j/k/l).
- tmux integration: attaching to sessions, creating sessions, checking session existence, and killing sessions.
- Dashboard mode: lists projects and allows quick project opening.
- Project-specific mode: focused board view with project context.
- Project recovery and cleanup: background cleanup of resources, archival of artifacts, and resource removal.

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)

## Architecture Overview
The sidebar and navigation system centers on the App state machine, which manages:
- Sidebar visibility and focus
- Project list population and selection
- Board state and navigation
- tmux session lifecycle
- Keyboard and mouse interactions

```mermaid
sequenceDiagram
participant User as "User"
participant App as "App (app.rs)"
participant Board as "BoardState (board.rs)"
participant Tmux as "tmux (mod.rs/operations.rs)"
User->>App : Press j/k to navigate sidebar
App->>App : Update selected_project
App->>App : switch_to_project_keep_sidebar(project)
App->>Board : Update board state (if applicable)
User->>App : Press Enter to open project
App->>Tmux : attach_session(session_name) or create_session
Tmux-->>App : Session attached/created
App-->>User : Focus moves to board with project context
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)

## Detailed Component Analysis

### Sidebar Layout and Project List
- The sidebar displays a list of tracked projects with names and selection markers.
- Keyboard navigation:
  - j/k: move selection up/down
  - Enter: open selected project (focus moves to board)
  - l or Right or Esc: move focus back to board
  - e: toggle sidebar visibility
- The sidebar is drawn conditionally based on visibility and focus state.

```mermaid
flowchart TD
Start(["Sidebar Visible"]) --> Nav{"Key Press?"}
Nav --> |j/k| MoveSel["Update selected_project"]
MoveSel --> AutoSwitch["switch_to_project_keep_sidebar()"]
AutoSwitch --> Nav
Nav --> |Enter| OpenProj["Focus board<br/>Open project"]
Nav --> |l/Right/Esc| FocusBoard["sidebar_focused = false"]
Nav --> |e| ToggleVis["Toggle sidebar_visible"]
OpenProj --> End(["Board Active"])
FocusBoard --> End
ToggleVis --> End
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)

### Project Selection and Activation
- Project selection updates the selected_project index and immediately switches to the project while keeping the sidebar visible.
- Activation (Enter) focuses the board and opens the project context.

```mermaid
sequenceDiagram
participant User as "User"
participant App as "App"
participant DB as "Database"
participant Tmux as "tmux"
User->>App : Press j/k
App->>App : selected_project +=/-= 1
App->>App : switch_to_project_keep_sidebar(project)
App->>DB : Load project tasks
App->>Tmux : Ensure session exists
User->>App : Press Enter
App->>App : sidebar_focused = false
App-->>User : Board focused with project context
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)

### Dashboard Mode vs Project-Specific Mode
- Dashboard mode shows a project list with navigation and options to open or create projects.
- Project-specific mode shows the Kanban board with project context and tmux session integration.

```mermaid
stateDiagram-v2
[*] --> Dashboard
Dashboard --> ProjectMode : "Open project"
ProjectMode --> Dashboard : "Close project"
Dashboard --> Dashboard : "Navigate projects"
ProjectMode --> ProjectMode : "Board navigation"
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)

### Board Navigation and tmux Integration
- Board navigation uses h/j/k/l for column/row movement.
- Enter on a task opens either a fullscreen tmux session or a shell popup depending on configuration.
- tmux integration includes attaching, creating, checking existence, and killing sessions.

```mermaid
classDiagram
class BoardState {
+tasks : Vec<Task>
+selected_column : usize
+selected_row : usize
+move_left()
+move_right()
+move_up()
+move_down()
+tasks_in_column(column) Vec<&Task>
+selected_task() Option<&Task>
}
class TmuxOps {
+attach_session(session_name)
+kill_session(session_name)
+has_session(session_name) bool
+create_session(session, working_dir)
+pane_current_command(target) Option<String>
}
BoardState --> TmuxOps : "uses for session management"
```

**Diagram sources**
- [board.rs](file://src/tui/board.rs)
- [operations.rs](file://src/tmux/operations.rs)

**Section sources**
- [board.rs](file://src/tui/board.rs)
- [operations.rs](file://src/tmux/operations.rs)

### Project Information Display
- The sidebar shows project names and selection markers.
- The dashboard shows project names with navigation hints.
- Project information includes repository names and paths stored in the global database.

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [server.rs](file://src/mcp/server.rs)

### Project Recovery and Cleanup Procedures
- Cleanup resources: archives artifacts, kills tmux windows, removes worktrees, and deletes branches when tasks are deleted or moved to Done.
- Recovery: ensures sessions exist before opening tasks; creates sessions if missing; sanitizes project names for tmux session names.

```mermaid
flowchart TD
Start(["Task Cleanup Triggered"]) --> CheckRes["Check session_name/worktree_path"]
CheckRes --> |Present| Archive["Archive .agtx artifacts"]
Archive --> KillWin["Kill tmux window"]
KillWin --> RemoveWT["Remove worktree"]
RemoveWT --> DeleteBranch["Delete branch (optional)"]
DeleteBranch --> End(["Cleanup Complete"])
CheckRes --> |Missing| End
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [operations.rs](file://src/tmux/operations.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [operations.rs](file://src/tmux/operations.rs)

## Dependency Analysis
The sidebar and navigation system depends on:
- TUI rendering and state management (App, BoardState, InputMode)
- tmux operations for session lifecycle
- Database for project and task persistence
- MCP server for project listing in external contexts

```mermaid
graph TB
App["App (app.rs)"] --> Board["BoardState (board.rs)"]
App --> Input["InputMode (input.rs)"]
App --> TmuxMod["tmux/mod.rs"]
App --> TmuxOps["tmux/operations.rs"]
App --> DB["Database (via db module)"]
MCP["mcp/server.rs"] --> DB
Main["main.rs"] --> App
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [input.rs](file://src/tui/input.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [server.rs](file://src/mcp/server.rs)
- [main.rs](file://src/main.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [board.rs](file://src/tui/board.rs)
- [input.rs](file://src/tui/input.rs)
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [server.rs](file://src/mcp/server.rs)
- [main.rs](file://src/main.rs)

## Performance Considerations
- Sidebar updates and project switching are immediate to maintain responsive navigation.
- tmux operations are invoked only when needed (attach/create/check/kill).
- Background cleanup threads prevent blocking the UI.

## Troubleshooting Guide
Common issues and resolutions:
- Sidebar does not appear: Press e to toggle sidebar visibility; ensure the sidebar is visible before attempting navigation.
- Cannot switch projects: Verify j/k navigation is active; ensure the sidebar is focused.
- Lost tmux session: Use cleanup procedures to remove stale resources; recreate sessions as needed.
- Project not listed: Confirm the project is tracked in the global database; use the MCP server to list projects if integrating externally.

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [server.rs](file://src/mcp/server.rs)

## Conclusion
The AGTX sidebar and navigation system provides efficient multi-project management with keyboard-driven workflows, tmux session integration, and robust project recovery and cleanup procedures. Users can quickly switch between projects, focus on specific boards, and manage resources with confidence.