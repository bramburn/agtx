# Dashboard Mode

<cite>
**Referenced Files in This Document**
- [main.rs](file://src/main.rs)
- [lib.rs](file://src/lib.rs)
- [app.rs](file://src/tui/app.rs)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [mod.rs](file://src/db/mod.rs)
- [mod.rs](file://src/config/mod.rs)
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
Dashboard Mode provides a unified, multi-project management interface that consolidates tasks across all tracked projects into a single view. It enables seamless navigation between workspaces, project discovery, and consolidated task filtering. The dashboard aggregates project metadata from a global index database and presents a streamlined project list with quick-switch capabilities. Users can filter tasks globally across projects and seamlessly transition into individual project contexts for detailed work.

## Project Structure
Dashboard Mode is implemented within the TUI application framework and leverages a dual-database architecture:
- Global index database: Tracks project metadata and relationships
- Project-specific databases: Store task data per project
- Configuration system: Manages theme, agents, and workflow plugins
- MCP integration: Provides global task listing and project discovery for external clients

```mermaid
graph TB
subgraph "CLI Layer"
MAIN["main.rs<br/>AppMode::Dashboard"]
end
subgraph "TUI Layer"
APP["app.rs<br/>App::new() + draw_dashboard()"]
STATE["AppState<br/>projects, selected_project,<br/>show_project_list"]
end
subgraph "Database Layer"
GLOBAL_DB["schema.rs<br/>open_global()"]
PROJECT_DB["schema.rs<br/>open_project()"]
MODELS["models.rs<br/>Project, Task"]
end
subgraph "Config Layer"
CONFIG["config/mod.rs<br/>GlobalConfig, MergedConfig"]
end
MAIN --> APP
APP --> STATE
APP --> GLOBAL_DB
APP --> PROJECT_DB
GLOBAL_DB --> MODELS
PROJECT_DB --> MODELS
APP --> CONFIG
```

**Diagram sources**
- [main.rs:47](file://src/main.rs#L47)
- [app.rs:749](file://src/tui/app.rs#L749)
- [schema.rs:51](file://src/db/schema.rs#L51)
- [schema.rs:14](file://src/db/schema.rs#L14)
- [models.rs](file://src/db/models.rs#L136)

**Section sources**
- [main.rs:47](file://src/main.rs#L47)
- [app.rs:749](file://src/tui/app.rs#L749)
- [schema.rs:51](file://src/db/schema.rs#L51)
- [schema.rs:14](file://src/db/schema.rs#L14)

## Core Components
Dashboard Mode consists of several interconnected components that work together to provide a cohesive multi-project experience:

### AppMode and CLI Integration
The application determines mode based on command-line arguments and current directory context. Dashboard mode is explicitly selected with the `-g` flag or when no git repository is detected in the current directory.

### AppState and Project Management
The AppState maintains project discovery state, including:
- `projects`: Vector of ProjectInfo structs containing project name and path
- `selected_project`: Index of currently selected project
- `show_project_list`: Boolean flag controlling project list visibility
- `sidebar_visible`: Controls sidebar presence in dashboard mode

### Database Architecture
Two distinct database instances operate concurrently:
- Global database: Stores project metadata and running agent sessions
- Project database: Contains task data for individual projects

**Section sources**
- [lib.rs:12](file://src/lib.rs#L12)
- [app.rs:448](file://src/tui/app.rs#L448)
- [app.rs:481](file://src/tui/app.rs#L481)
- [schema.rs:51](file://src/db/schema.rs#L51)
- [schema.rs:14](file://src/db/schema.rs#L14)

## Architecture Overview
Dashboard Mode implements a hybrid rendering approach with separate UI flows for dashboard and project contexts:

```mermaid
sequenceDiagram
participant User as "User"
participant App as "App (dashboard)"
participant GlobalDB as "Global Database"
participant ProjectDB as "Project Database"
participant Tmux as "tmux Sessions"
User->>App : Launch with -g flag
App->>GlobalDB : open_global()
GlobalDB-->>App : Global connection
App->>GlobalDB : get_all_projects()
GlobalDB-->>App : Project list
App->>App : refresh_projects()
App->>App : draw_dashboard()
User->>App : Press 'p'
App->>App : show_project_list = true
App->>App : draw_project_list()
User->>App : Navigate projects (j/k)
App->>App : switch_to_project_keep_sidebar()
App->>ProjectDB : open_project(path)
ProjectDB-->>App : Project DB connection
App->>ProjectDB : get_all_tasks()
ProjectDB-->>App : Tasks for selected project
User->>App : Enter project
App->>App : switch_to_project()
App->>Tmux : ensure_project_tmux_session()
App->>App : mode = Project
```

**Diagram sources**
- [main.rs:47](file://src/main.rs#L47)
- [app.rs:6218](file://src/tui/app.rs#L6218)
- [app.rs:6699](file://src/tui/app.rs#L6699)
- [app.rs:2635](file://src/tui/app.rs#L2635)

The architecture supports seamless transitions between dashboard and project modes while maintaining state consistency across multiple project databases.

## Detailed Component Analysis

### Dashboard Rendering Engine
The dashboard employs a three-section layout with ASCII art branding, project selection interface, and footer controls:

```mermaid
flowchart TD
Start([Dashboard Draw]) --> CheckList{"show_project_list?"}
CheckList --> |Yes| RenderList["Render Project List<br/>- Scrollable list<br/>- Navigation keys (j/k)<br/>- Selection highlighting"]
CheckList --> |No| RenderOptions["Render Options Panel<br/>- [p] Open project<br/>- [n] New project<br/>- [q] Quit"]
RenderList --> Footer["Render Footer<br/>- [p] projects<br/>- [n] new project<br/>- [q] quit"]
RenderOptions --> Footer
Footer --> End([Complete])
```

**Diagram sources**
- [app.rs:2635](file://src/tui/app.rs#L2635)
- [app.rs:2685](file://src/tui/app.rs#L2685)

### Project Discovery and Management
Project discovery operates through the global database, which tracks all previously opened projects with timestamps:

```mermaid
classDiagram
class Project {
+String id
+String name
+String path
+String github_url
+String default_agent
+DateTime last_opened
+new(name, path)
}
class Database {
+Connection conn
+open_global() Database
+open_project(project_path) Database
+upsert_project(project) void
+get_all_projects() Vec~Project~
+get_project_by_id(id) Option~Project~
}
class ProjectInfo {
+String name
+String path
}
Database --> Project : "manages"
AppState --> ProjectInfo : "displays"
ProjectInfo --> Database : "derived from"
```

**Diagram sources**
- [models.rs:136](file://src/db/models.rs#L136)
- [schema.rs:404](file://src/db/schema.rs#L404)
- [app.rs:672](file://src/tui/app.rs#L672)

### Project Switching Workflow
The project switching mechanism ensures smooth transitions between work contexts:

```mermaid
sequenceDiagram
participant User as "User"
participant Dashboard as "Dashboard"
participant ProjectDB as "Project DB"
participant GlobalDB as "Global DB"
participant Tmux as "tmux"
User->>Dashboard : Select project
Dashboard->>Dashboard : switch_to_project_keep_sidebar()
Dashboard->>ProjectDB : open_project(path)
ProjectDB-->>Dashboard : Connection OK
Dashboard->>GlobalDB : upsert_project(project)
GlobalDB-->>Dashboard : Updated last_opened
Dashboard->>Tmux : ensure_project_tmux_session()
Tmux-->>Dashboard : Session ready
Dashboard->>Dashboard : Update AppState
Dashboard->>Dashboard : mode = Project
```

**Diagram sources**
- [app.rs:6708](file://src/tui/app.rs#L6708)
- [app.rs:6723](file://src/tui/app.rs#L6723)
- [app.rs:6732](file://src/tui/app.rs#L6732)

### Task Filtering Across Multiple Databases
Dashboard Mode provides two primary filtering approaches:

#### Global Task Aggregation
The MCP server enables cross-project task queries through the global database:

```mermaid
flowchart TD
MCP["MCP Server"] --> ListProjects["list_projects()"]
MCP --> ListTasks["list_tasks(status?)"]
ListProjects --> GlobalDB["Global Database"]
ListTasks --> ProjectDB["Project Database"]
GlobalDB --> Projects["Project list"]
ProjectDB --> Tasks["Filtered tasks"]
Projects --> MCP
Tasks --> MCP
```

**Diagram sources**
- [server.rs:525](file://src/mcp/server.rs#L525)
- [server.rs:548](file://src/mcp/server.rs#L548)

#### Local Task Filtering Within Projects
Within project mode, the application filters tasks by status and displays them in Kanban columns with dependency awareness.

**Section sources**
- [app.rs:2635](file://src/tui/app.rs#L2635)
- [app.rs:6699](file://src/tui/app.rs#L6699)
- [server.rs:525](file://src/mcp/server.rs#L525)
- [server.rs:548](file://src/mcp/server.rs#L548)

### Dashboard-Specific UI Elements
The dashboard interface includes specialized elements for multi-project management:

#### Project List Interface
- Scrollable project list with navigation keys (j/k)
- Visual indicators for current project
- Keyboard shortcuts for quick actions
- Responsive layout adapting to terminal size

#### Options Panel
- Project creation from current directory
- Existing project opening
- Quit application

#### Footer Controls
- Consistent keyboard navigation
- Quick access to project management
- Status indicators for application state

**Section sources**
- [app.rs:2685](file://src/tui/app.rs#L2685)
- [app.rs:2709](file://src/tui/app.rs#L2709)
- [app.rs:2722](file://src/tui/app.rs#L2722)

## Dependency Analysis
Dashboard Mode exhibits well-defined dependencies between components:

```mermaid
graph TB
subgraph "External Dependencies"
SQLITE["rusqlite"]
TMUX["tmux"]
RUST["Rust Standard Library"]
end
subgraph "Internal Modules"
MAIN["main.rs"]
LIB["lib.rs"]
APP["tui/app.rs"]
DB["db/*"]
CFG["config/*"]
MCP["mcp/server.rs"]
end
MAIN --> APP
APP --> DB
APP --> CFG
APP --> MCP
DB --> SQLITE
APP --> TMUX
LIB --> APP
```

**Diagram sources**
- [main.rs:1](file://src/main.rs#L1)
- [app.rs:20](file://src/tui/app.rs#L20)
- [schema.rs:2](file://src/db/schema.rs#L2)

The dependency structure maintains clear separation of concerns:
- Main module handles CLI parsing and mode selection
- TUI module manages rendering and user interaction
- Database module provides persistence abstraction
- Config module handles application settings
- MCP module enables external integrations

**Section sources**
- [main.rs:1](file://src/main.rs#L1)
- [app.rs:20](file://src/tui/app.rs#L20)
- [schema.rs:2](file://src/db/schema.rs#L2)

## Performance Considerations
Dashboard Mode implements several optimizations for managing multiple concurrent projects:

### Database Connection Management
- Separate database connections for global and project contexts
- Lazy loading of project databases on demand
- Efficient indexing on project path and task status

### Memory Optimization
- Project list caching to avoid repeated database queries
- Incremental task loading for active projects
- State consolidation to minimize memory footprint

### Rendering Efficiency
- Conditional rendering based on state changes
- Minimal redraw operations
- Optimized layout calculations

### Concurrency Patterns
- Non-blocking background operations for task refresh
- Asynchronous database operations
- Event-driven UI updates

**Section sources**
- [schema.rs:51](file://src/db/schema.rs#L51)
- [schema.rs:14](file://src/db/schema.rs#L14)
- [app.rs:1190](file://src/tui/app.rs#L1190)

## Troubleshooting Guide
Common issues and solutions when using Dashboard Mode:

### Project Not Appearing in Dashboard
**Symptoms**: Project missing from project list
**Causes**: 
- Project path no longer exists
- Database corruption
- Permission issues

**Solutions**:
- Verify project directory accessibility
- Check global database integrity
- Re-add project through normal project mode

### Slow Project Switching
**Symptoms**: Delayed transitions between projects
**Causes**:
- Large project databases
- Network-mounted project directories
- tmux session initialization delays

**Solutions**:
- Optimize project database size
- Move projects to local storage
- Verify tmux server responsiveness

### Task Filtering Issues
**Symptoms**: Incorrect task filtering results
**Causes**:
- Database query errors
- Cache inconsistencies
- Plugin configuration conflicts

**Solutions**:
- Clear application cache
- Rebuild database indices
- Validate plugin configurations

**Section sources**
- [app.rs:6711](file://src/tui/app.rs#L6711)
- [schema.rs:404](file://src/db/schema.rs#L404)

## Conclusion
Dashboard Mode provides a comprehensive solution for multi-project management by combining efficient database architecture with intuitive user interface design. The dual-database approach ensures scalability while maintaining responsive user interactions. Key strengths include seamless project switching, consolidated task management, and robust state consistency across multiple project contexts. The implementation demonstrates careful consideration of performance, usability, and extensibility, making it suitable for developers managing numerous concurrent projects.