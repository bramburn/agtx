# Task Management and Editing

<cite>
**Referenced Files in This Document**
- [main.rs](file://src/main.rs)
- [app.rs](file://src/tui/app.rs)
- [input.rs](file://src/tui/input.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [board.rs](file://src/tui/board.rs)
- [app_tests.rs](file://tests/app_tests.rs)
- [db_tests.rs](file://tests/db_tests.rs)
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
This document explains the task management and editing features in AGTX with a focus on the task creation wizard and inline editing capabilities. It covers:
- Step-by-step task creation: title input, plugin selection, and description composition
- Inline editor features for inserting file references, agent skills, and task dependencies
- Task modification workflows: editing descriptions, updating agent assignments, changing priorities
- The shell popup system for viewing task details, running commands, and monitoring agent activity
- Practical examples for complex tasks with multiple dependencies, external file references, and cross-task coordination
- Validation and error handling for invalid inputs and conflicting dependencies

## Project Structure
AGTX’s task lifecycle spans the TUI application, database persistence, and tmux-backed agent sessions. The key areas are:
- TUI application state and modes for task creation and editing
- Database schema and model definitions for tasks and transitions
- Shell popup rendering and scrolling for live agent output
- MCP server integration for task queries and actions

```mermaid
graph TB
subgraph "TUI"
A["app.rs<br/>AppState, InputMode, Wizard"]
B["input.rs<br/>InputMode enum"]
C["board.rs<br/>BoardState"]
D["shell_popup.rs<br/>ShellPopup rendering"]
end
subgraph "Persistence"
E["models.rs<br/>Task, TaskStatus"]
F["schema.rs<br/>Database, SQL ops"]
end
subgraph "Integration"
G["main.rs<br/>App bootstrap"]
H["server.rs<br/>MCP task queries"]
end
G --> A
A --> B
A --> C
A --> D
A --> E
A --> F
H --> F
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [input.rs](file://src/tui/input.rs)
- [board.rs](file://src/tui/board.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [main.rs](file://src/main.rs)
- [server.rs](file://src/mcp/server.rs)

**Section sources**
- [main.rs](file://src/main.rs)
- [app.rs](file://src/tui/app.rs)
- [input.rs](file://src/tui/input.rs)
- [board.rs](file://src/tui/board.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [server.rs](file://src/mcp/server.rs)

## Core Components
- Task model and status: defines task fields, lifecycle, and helper methods
- Database operations: create, update, query tasks, and dependency satisfaction checks
- TUI wizard: title input, plugin selection, and description composition
- Inline editing: file references, agent skills, and task dependencies
- Shell popup: detached tmux window for monitoring agent activity
- MCP server: task retrieval and allowed actions computed from status and dependencies

**Section sources**
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [app.rs](file://src/tui/app.rs)
- [input.rs](file://src/tui/input.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [server.rs](file://src/mcp/server.rs)

## Architecture Overview
The task creation and editing pipeline integrates TUI input modes, wizard state, database persistence, and tmux agent sessions. The MCP server augments task visibility and action availability.

```mermaid
sequenceDiagram
participant User as "User"
participant App as "TUI App (app.rs)"
participant DB as "Database (schema.rs)"
participant TMUX as "tmux (operations)"
participant MCP as "MCP Server (server.rs)"
User->>App : "Press New Task"
App->>App : "InputMode : : InputTitle"
User->>App : "Enter title"
App->>App : "Advance to plugin selection"
App->>App : "InputMode : : SelectPlugin"
User->>App : "Select plugin"
App->>App : "Advance to description"
App->>App : "InputMode : : InputDescription"
User->>App : "Insert references and save"
App->>DB : "save_task() -> create/update"
DB-->>App : "OK"
App->>TMUX : "Start agent session (if needed)"
TMUX-->>App : "Session ready"
App-->>User : "Task saved and visible"
MCP->>DB : "List tasks / Get task details"
DB-->>MCP : "Task data"
MCP-->>User : "Allowed actions based on status"
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [schema.rs](file://src/db/schema.rs)
- [server.rs](file://src/mcp/server.rs)

## Detailed Component Analysis

### Task Creation Wizard
The wizard guides users through three steps:
1. Title input
2. Plugin selection (optional, skipped when no agents are detected)
3. Description composition with inline references

Key behaviors:
- Title step advances to plugin selection unless agents are absent
- Plugin selection filters compatible plugins for the configured agent
- Description step supports file references, agent skills, and task dependencies
- Saving persists title, description, agent, plugin, and referenced task IDs

```mermaid
flowchart TD
Start(["Start Wizard"]) --> Title["Input Mode: InputTitle<br/>Enter task title"]
Title --> AgentsPresent{"Agents detected?"}
AgentsPresent --> |No| Desc["Init description input"]
AgentsPresent --> |Yes| PluginSel["Input Mode: SelectPlugin<br/>Choose workflow plugin"]
PluginSel --> Desc
Desc --> EditDesc["Input Mode: InputDescription<br/>Compose description"]
EditDesc --> Save["save_task()<br/>Create or update task"]
Save --> End(["Task Saved"])
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [input.rs](file://src/tui/input.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [input.rs](file://src/tui/input.rs)
- [app_tests.rs](file://tests/app_tests.rs)

### Inline Editor Capabilities
While composing descriptions, users can insert:
- File references: triggers a file search dropdown and inserts a reference marker
- Agent skills: triggers a skill search dropdown and inserts a skill reference
- Task dependencies: triggers a task reference search and inserts a task reference

These triggers are handled during description input and maintain an input buffer with a cursor position. Insertions update the buffer and optionally highlight references.

```mermaid
sequenceDiagram
participant User as "User"
participant App as "TUI App"
participant FS as "File Search"
participant SS as "Skill Search"
participant TS as "Task Ref Search"
User->>App : "Type #"
App->>FS : "Open file search dropdown"
FS-->>App : "Selected file reference"
App->>App : "Insert reference into input buffer"
User->>App : "Type /"
App->>SS : "Open skill search dropdown"
SS-->>App : "Selected skill reference"
App->>App : "Insert reference into input buffer"
User->>App : "Type !"
App->>TS : "Open task reference search"
TS-->>App : "Selected task reference"
App->>App : "Insert reference into input buffer"
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [app_tests.rs](file://tests/app_tests.rs)

### Task Modification Workflows
Editing an existing task updates:
- Title and description
- Agent assignment (from configuration)
- Plugin selection
- Referenced task IDs (dependencies)

Saving an existing task performs an update operation and refreshes the board.

```mermaid
flowchart TD
EditStart(["Edit Existing Task"]) --> Load["Load task from DB"]
Load --> Compose["Compose changes in wizard buffers"]
Compose --> Update["save_task() -> update_task()"]
Update --> Refresh["refresh_tasks()"]
Refresh --> Done(["Task Updated"])
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [schema.rs](file://src/db/schema.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [schema.rs](file://src/db/schema.rs)
- [app_tests.rs](file://tests/app_tests.rs)

### Shell Popup System
The shell popup displays a detached tmux window for a selected task:
- Renders styled terminal output lines
- Supports scrolling up/down and jumping to bottom
- Shows a footer with navigation hints
- Optionally shows an escalation banner

```mermaid
classDiagram
class ShellPopup {
+string task_title
+string window_name
+i32 scroll_offset
+Vec~u8~ cached_content
+Option<(u16,u16)> last_pane_size
+Option<string> escalation_note
+Option<string> task_id
+scroll_up(lines)
+scroll_down(lines)
+scroll_to_bottom()
+is_at_bottom() bool
}
class ShellPopupView {
+string title
+Vec~Line~ lines
+usize start_line
+usize total_lines
+bool is_at_bottom
}
ShellPopup --> ShellPopupView : "compute_visible_lines()"
```

**Diagram sources**
- [shell_popup.rs](file://src/tui/shell_popup.rs)

**Section sources**
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [app.rs](file://src/tui/app.rs)

### Task Validation and Error Handling
- Dependency satisfaction: a task is considered satisfied if all referenced tasks are in Review or Done; missing references are treated as satisfied
- Status transitions: allowed actions are computed based on current status and dependency satisfaction
- Warning messages: the UI surfaces warnings for prerequisites (e.g., research phase required before planning)

```mermaid
flowchart TD
CheckDeps["Check referenced_tasks"] --> HasRefs{"Any refs?"}
HasRefs --> |No| OK["deps_satisfied = true"]
HasRefs --> |Yes| Iterate["Iterate refs"]
Iterate --> Exists{"Task exists?"}
Exists --> |No| TreatOK["Treat as satisfied"]
Exists --> |Yes| Status{"Status in Review/Done?"}
Status --> |Yes| Next["Next ref"]
Status --> |No| NotOK["deps_satisfied = false"]
Next --> Iterate
```

**Diagram sources**
- [schema.rs](file://src/db/schema.rs)
- [server.rs](file://src/mcp/server.rs)

**Section sources**
- [schema.rs](file://src/db/schema.rs)
- [db_tests.rs](file://tests/db_tests.rs)
- [server.rs](file://src/mcp/server.rs)
- [app.rs](file://src/tui/app.rs)

## Dependency Analysis
- TUI depends on database models and schema for task persistence
- TUI coordinates with tmux operations for agent sessions
- MCP server reads from the database to provide task summaries and allowed actions
- Board state tracks selected tasks and column positions for navigation

```mermaid
graph LR
App["app.rs"] --> Models["models.rs"]
App --> Schema["schema.rs"]
App --> Board["board.rs"]
App --> Shell["shell_popup.rs"]
MCP["server.rs"] --> Schema
```

**Diagram sources**
- [app.rs](file://src/tui/app.rs)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [board.rs](file://src/tui/board.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [server.rs](file://src/mcp/server.rs)

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [board.rs](file://src/tui/board.rs)
- [shell_popup.rs](file://src/tui/shell_popup.rs)
- [server.rs](file://src/mcp/server.rs)

## Performance Considerations
- Database operations are executed synchronously in the main thread; batch operations are supported for bulk inserts
- Dependency satisfaction checks iterate over referenced task IDs; caching of satisfaction results can reduce repeated DB queries
- Shell popup computes visible lines and trims trailing empty lines to optimize rendering
- Input mode transitions minimize unnecessary UI redraws

## Troubleshooting Guide
Common scenarios and resolutions:
- Research phase required before planning: the UI warns and prevents progression until prerequisites are met
- Empty description handling: saving with an empty description stores a null value
- Plugin filtering: incompatible plugins are excluded based on agent support
- Task deletion: removal cleans up tmux sessions and Git resources before deleting from the database

**Section sources**
- [app.rs](file://src/tui/app.rs)
- [schema.rs](file://src/db/schema.rs)
- [app_tests.rs](file://tests/app_tests.rs)

## Conclusion
AGTX provides a structured task creation wizard and robust inline editing features for composing rich task descriptions with file references, agent skills, and task dependencies. The shell popup enables continuous monitoring of agent activity, while the database and MCP server integrate task state, dependencies, and allowed actions. The validation system ensures tasks can only progress when dependencies are satisfied, and the UI surfaces helpful warnings to guide users through required steps.