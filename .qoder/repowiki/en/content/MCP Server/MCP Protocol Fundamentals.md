# MCP Protocol Fundamentals

<cite>
**Referenced Files in This Document**
- [server.rs](file://src/mcp/server.rs)
- [mod.rs](file://src/mcp/mod.rs)
- [main.rs](file://src/main.rs)
- [.mcp.json](file://.mcp.json)
- [README.md](file://README.md)
- [CLAUDE.md](file://CLAUDE.md)
- [Cargo.lock](file://Cargo.lock)
- [models.rs](file://src/db/models.rs)
- [schema.rs](file://src/db/schema.rs)
- [config.rs](file://src/config/mod.rs)
- [mcp_tests.rs](file://tests/mcp_tests.rs)
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
10. [Appendices](#appendices)

## Introduction
This document explains the Model Context Protocol (MCP) fundamentals as implemented by AGTX. It focuses on AGTX's MCP server that exposes a Kanban-style task management system as JSON-RPC tools over stdio. The server supports two operational modes:
- Global MCP server: serves all projects indexed in the global database and requires a project_id parameter for CRUD operations.
- Project-scoped MCP server: binds to a single project path and ignores project_id in tool calls.

The MCP server integrates tightly with AGTX's task lifecycle, Git operations, tmux sessions, and SQLite-backed persistence. It enables external automation and agent integration by exposing tools for listing tasks, moving tasks between phases, conflict checking, notification retrieval, and pane inspection.

## Project Structure
AGTX organizes MCP-related code under src/mcp with a minimal public surface:
- src/mcp/mod.rs exports the MCP server entry points.
- src/mcp/server.rs implements the MCP server, tool router, and transport setup.
- The CLI entry point in src/main.rs routes "mcp-serve" to the MCP server and validates project paths.
- A configuration file .mcp.json registers the MCP server as a stdio service.
- The MCP server relies on AGTX's database layer (src/db), configuration layer (src/config), and task models (src/db/models.rs).

```mermaid
graph TB
CLI["CLI Entry Point<br/>src/main.rs"] --> MCPMod["MCP Module<br/>src/mcp/mod.rs"]
MCPMod --> Server["MCP Server Implementation<br/>src/mcp/server.rs"]
Server --> Transport["JSON-RPC over stdio<br/>rmcp transport::io::stdio"]
Server --> DBLayer["Database Layer<br/>src/db/schema.rs"]
Server --> Models["Task/Project Models<br/>src/db/models.rs"]
Server --> Config["Configuration Layer<br/>src/config/mod.rs"]
Config --> Plugins["Workflow Plugins<br/>config/mod.rs"]
CLI --> MCPReg["MCP Registration<br/>.mcp.json"]
```

**Diagram sources**
- [main.rs:30-46](file://src/main.rs#L30-L46)
- [mod.rs:1-5](file://src/mcp/mod.rs#L1-L5)
- [server.rs:1245-1263](file://src/mcp/server.rs#L1245-L1263)
- [schema.rs:12-67](file://src/db/schema.rs#L12-L67)
- [models.rs:58-79](file://src/db/models.rs#L58-L79)
- [config.rs:410-595](file://src/config/mod.rs#L410-L595)
- [.mcp.json:1-7](file://.mcp.json#L1-L7)

**Section sources**
- [main.rs:30-46](file://src/main.rs#L30-L46)
- [mod.rs:1-5](file://src/mcp/mod.rs#L1-L5)
- [.mcp.json:1-7](file://.mcp.json#L1-L7)

## Core Components
- ServerMode enum: Determines whether the MCP server operates in global mode (requires project_id) or project-scoped mode (fixed path).
- AgtxMcpServer: Implements the MCP server, tool router, and project resolution helpers.
- Tool router: Declares MCP tools for listing projects/tasks, task transitions, conflict checks, notifications, pane reads, and task CRUD.
- Transport: Uses rmcp's stdio transport for JSON-RPC over stdin/stdout.
- Database integration: Opens project/global databases, resolves project paths, and persists transitions and notifications.
- Configuration integration: Resolves default agent and plugin from merged global/project configuration.

Key responsibilities:
- Capability negotiation: Reports ServerInfo with tool capabilities.
- Transport setup: Initializes stdio transport and waits for requests.
- Project resolution: Validates and resolves project paths depending on mode.
- Serialization: Returns JSON responses for all tools.

**Section sources**
- [server.rs:16-24](file://src/mcp/server.rs#L16-L24)
- [server.rs:395-471](file://src/mcp/server.rs#L395-L471)
- [server.rs:1218-1243](file://src/mcp/server.rs#L1218-L1243)
- [server.rs:1245-1263](file://src/mcp/server.rs#L1245-L1263)

## Architecture Overview
The MCP server architecture follows a layered design:
- Transport layer: JSON-RPC over stdio via rmcp.
- Server layer: AgtxMcpServer implements ServerHandler and tool_router.
- Domain layer: Tools operate on tasks, transitions, and notifications.
- Persistence layer: SQLite-backed databases for projects and tasks.
- Configuration layer: Global and project configurations for defaults.

```mermaid
graph TB
subgraph "Transport Layer"
Stdio["JSON-RPC over stdio<br/>rmcp::transport::io::stdio"]
end
subgraph "Server Layer"
Handler["ServerHandler<br/>AgtxMcpServer"]
Router["Tool Router<br/>tool_router macro"]
end
subgraph "Domain Layer"
Tasks["Task Operations<br/>list_tasks, get_task, move_task"]
CRUD["CRUD Operations<br/>create_task, update_task, delete_task"]
Conflicts["Conflict Checks<br/>check_conflicts"]
Notifications["Notifications<br/>get_notifications"]
PaneOps["Pane Ops<br/>read_pane_content, send_to_task"]
Projects["Project Ops<br/>list_projects"]
end
subgraph "Persistence Layer"
ProjDB["Project DB<br/>schema.rs"]
GlobalDB["Global DB<br/>schema.rs"]
end
subgraph "Configuration Layer"
GlobalCfg["GlobalConfig<br/>config.rs"]
ProjCfg["ProjectConfig<br/>config.rs"]
end
Stdio --> Handler
Handler --> Router
Router --> Tasks
Router --> CRUD
Router --> Conflicts
Router --> Notifications
Router --> PaneOps
Router --> Projects
Tasks --> ProjDB
CRUD --> ProjDB
Conflicts --> ProjDB
Notifications --> ProjDB
Projects --> GlobalDB
Handler --> GlobalCfg
Handler --> ProjCfg
```

**Diagram sources**
- [server.rs:4-10](file://src/mcp/server.rs#L4-L10)
- [server.rs:521-1216](file://src/mcp/server.rs#L521-L1216)
- [schema.rs:12-67](file://src/db/schema.rs#L12-L67)
- [config.rs:230-303](file://src/config/mod.rs#L230-L303)

## Detailed Component Analysis

### ServerMode and Project Resolution
ServerMode determines how project_id is handled:
- Global: Requires project_id; resolves path via global DB lookup.
- Project: Ignores project_id; uses the fixed path provided at startup.

```mermaid
classDiagram
class ServerMode {
<<enum>>
+Project(PathBuf)
+Global
}
class AgtxMcpServer {
-mode : ServerMode
-tool_router : ToolRouter<Self>
+new(mode)
+resolve_project_path(project_id) Result<PathBuf,String>
+open_project_db_for(project_id) Result<Database,String>
+open_project_db() Result<Database,String>
+open_global_db() Result<Database,String>
+project_name_for(project_id) String
+config_defaults_for(project_id) (String,Option<String>)
+allowed_actions(task, deps_satisfied) Vec<String>
}
AgtxMcpServer --> ServerMode : "uses"
```

**Diagram sources**
- [server.rs:16-24](file://src/mcp/server.rs#L16-L24)
- [server.rs:395-471](file://src/mcp/server.rs#L395-L471)

**Section sources**
- [server.rs:16-24](file://src/mcp/server.rs#L16-L24)
- [server.rs:409-444](file://src/mcp/server.rs#L409-L444)

### MCP Server Initialization and Transport Setup
The serve function selects ServerMode based on CLI arguments, validates database accessibility, constructs AgtxMcpServer, and starts the stdio transport.

```mermaid
sequenceDiagram
participant CLI as "CLI"
participant Main as "main.rs"
participant MCP as "AgtxMcpServer"
participant Transport as "rmcp stdio"
participant DB as "Database"
CLI->>Main : "agtx mcp-serve [path]"
Main->>Main : "parse args and validate path"
Main->>DB : "open_project(path) or open_global()"
Main->>MCP : "AgtxMcpServer : : new(ServerMode)"
MCP->>Transport : "serve(stdio())"
Transport-->>MCP : "service.handle()"
MCP-->>CLI : "waiting() await"
```

**Diagram sources**
- [main.rs:30-46](file://src/main.rs#L30-L46)
- [server.rs:1245-1263](file://src/mcp/server.rs#L1245-L1263)

**Section sources**
- [main.rs:30-46](file://src/main.rs#L30-L46)
- [server.rs:1245-1263](file://src/mcp/server.rs#L1245-L1263)

### Capability Negotiation and Server Info
The ServerHandler implementation reports ServerInfo with instructions tailored to the mode and enables tools capability.

```mermaid
flowchart TD
Start(["ServerHandler::get_info"]) --> ModeCheck{"ServerMode"}
ModeCheck --> |Global| GlobalInstr["Set instructions for global mode<br/>require project_id"]
ModeCheck --> |Project| ProjInstr["Set instructions for project-scoped mode"]
GlobalInstr --> BuildInfo["Build ServerInfo with tools capability"]
ProjInstr --> BuildInfo
BuildInfo --> End(["Return ServerInfo"])
```

**Diagram sources**
- [server.rs:1218-1243](file://src/mcp/server.rs#L1218-L1243)

**Section sources**
- [server.rs:1218-1243](file://src/mcp/server.rs#L1218-L1243)

### Tool Routing Patterns
Tools are declared with #[tool] and mapped via #[tool_router]. The router delegates to methods on AgtxMcpServer. Parameters are validated using schemars-generated JSON schemas.

```mermaid
classDiagram
class AgtxMcpServer {
+list_projects(params) String
+list_tasks(params) String
+get_task(params) String
+move_task(params) String
+get_transition_status(params) String
+check_conflicts(params) String
+get_notifications(params) String
+read_pane_content(params) String
+send_to_task(params) String
+create_task(params) String
+create_tasks_batch(params) String
+update_task(params) String
+delete_task(params) String
}
class ToolRouter {
+route(tool_name, params) -> String
}
AgtxMcpServer --> ToolRouter : "declares tools"
```

**Diagram sources**
- [server.rs:521-1216](file://src/mcp/server.rs#L521-L1216)

**Section sources**
- [server.rs:521-1216](file://src/mcp/server.rs#L521-L1216)

### Task Transition Workflow
External orchestrators can queue transitions via move_task. The MCP server validates actions, checks dependency gates, and writes a TransitionRequest. The TUI processes the queue and updates state.

```mermaid
sequenceDiagram
participant Orchestrator as "External Agent"
participant MCP as "AgtxMcpServer"
participant DB as "Project DB"
participant TUI as "AGTX TUI"
Orchestrator->>MCP : "move_task(task_id, action, reason?)"
MCP->>DB : "get_task(task_id)"
DB-->>MCP : "Task"
MCP->>MCP : "validate action and dependencies"
MCP->>DB : "create_transition_request(TransitionRequest)"
DB-->>MCP : "OK"
MCP-->>Orchestrator : "MoveTaskResult(request_id, message)"
TUI->>DB : "poll pending transition_requests"
TUI->>DB : "mark_transition_processed(request_id, error?)"
```

**Diagram sources**
- [server.rs:655-721](file://src/mcp/server.rs#L655-L721)
- [models.rs:159-184](file://src/db/models.rs#L159-L184)

**Section sources**
- [server.rs:655-721](file://src/mcp/server.rs#L655-L721)
- [models.rs:159-184](file://src/db/models.rs#L159-L184)

### Conflict Checking Algorithm
The check_conflicts tool inspects task branches against the main branch for merge conflicts using Git operations.

```mermaid
flowchart TD
Start(["check_conflicts(params)"]) --> ResolveProj["Resolve project path"]
ResolveProj --> DetectMain["Detect main branch"]
DetectMain --> FetchTasks{"task_id provided?"}
FetchTasks --> |Yes| GetOne["Get single task"]
FetchTasks --> |No| GetReview["Get all Review tasks"]
GetOne --> Iterate["Iterate tasks"]
GetReview --> Iterate
Iterate --> CheckGit["check_merge_conflicts(branch, main)"]
CheckGit --> BuildResp["Build ConflictCheckResult"]
BuildResp --> Serialize["Serialize JSON"]
Serialize --> End(["Return JSON"])
```

**Diagram sources**
- [server.rs:757-832](file://src/mcp/server.rs#L757-L832)

**Section sources**
- [server.rs:757-832](file://src/mcp/server.rs#L757-L832)

### Notification Retrieval Pattern
The get_notifications tool consumes and returns pending notifications, removing them from the queue.

```mermaid
sequenceDiagram
participant Caller as "Caller"
participant MCP as "AgtxMcpServer"
participant DB as "Project DB"
Caller->>MCP : "get_notifications(project_id?)"
MCP->>DB : "consume_notifications()"
DB-->>MCP : "[Notification...]"
MCP-->>Caller : "GetNotificationsResponse(JSON)"
```

**Diagram sources**
- [server.rs:834-858](file://src/mcp/server.rs#L834-L858)

**Section sources**
- [server.rs:834-858](file://src/mcp/server.rs#L834-L858)

### tmux Integration for Pane Inspection and Messaging
The read_pane_content and send_to_task tools interact with tmux sessions associated with tasks.

```mermaid
flowchart TD
Start(["read_pane_content/send_to_task"]) --> LookupTask["Lookup task by task_id"]
LookupTask --> ValidatePhase{"Active phase?"}
ValidatePhase --> |No| Error["Return error"]
ValidatePhase --> |Yes| Session["Use session_name"]
Session --> TmuxCmd["Execute tmux capture-pane/send-keys"]
TmuxCmd --> BuildResp["Build response"]
BuildResp --> Serialize["Serialize JSON"]
Serialize --> End(["Return JSON"])
```

**Diagram sources**
- [server.rs:860-974](file://src/mcp/server.rs#L860-L974)

**Section sources**
- [server.rs:860-974](file://src/mcp/server.rs#L860-L974)

### MCP Protocol Compliance and Serialization
- Transport: Uses rmcp::transport::io::stdio for JSON-RPC over stdio.
- Schema generation: Parameters and responses derive schemars::JsonSchema for tool metadata.
- Serialization: Tools return pretty-printed JSON strings; errors are formatted strings.
- Capability negotiation: ServerInfo declares tool capability.

**Section sources**
- [server.rs:4-10](file://src/mcp/server.rs#L4-L10)
- [server.rs:28-244](file://src/mcp/server.rs#L28-L244)
- [server.rs:1218-1243](file://src/mcp/server.rs#L1218-L1243)

### Relationship to AGTX Task Management
MCP complements AGTX's TUI and task lifecycle:
- The TUI monitors transition_requests and applies side effects (worktree creation, agent spawning).
- Notifications bridge TUI state to the orchestrator when idle.
- MCP enables external agents to orchestrate tasks without manual intervention.

```mermaid
graph TB
Orchestrator["External Orchestrator"] --> MCP["MCP Server"]
MCP --> DB["Project/Global DB"]
DB --> TUI["AGTX TUI"]
TUI --> Orchestrator
MCP --> Git["Git Operations"]
MCP --> Tmux["tmux Sessions"]
```

**Diagram sources**
- [CLAUDE.md:205-213](file://CLAUDE.md#L205-L213)
- [README.md:623-636](file://README.md#L623-L636)

**Section sources**
- [CLAUDE.md:205-213](file://CLAUDE.md#L205-L213)
- [README.md:623-636](file://README.md#L623-L636)

## Dependency Analysis
- External dependencies: rmcp (0.16.0) provides the MCP framework, stdio transport, and tool macros.
- Internal dependencies: MCP server depends on database schema, task models, and configuration layers.
- Coupling: Tools depend on Database for persistence; project resolution depends on global DB in global mode.

```mermaid
graph TB
RMCP["rmcp 0.16.0<br/>Cargo.lock"] --> Server["AgtxMcpServer"]
Server --> DB["Database (schema.rs)"]
Server --> Models["Task/Project Models (models.rs)"]
Server --> Config["Global/Project Config (config.rs)"]
```

**Diagram sources**
- [Cargo.lock:1385-1405](file://Cargo.lock#L1385-L1405)
- [server.rs:13-14](file://src/mcp/server.rs#L13-L14)

**Section sources**
- [Cargo.lock:1385-1405](file://Cargo.lock#L1385-L1405)
- [server.rs:13-14](file://src/mcp/server.rs#L13-L14)

## Performance Considerations
- Database access: Each tool call opens the appropriate database (project or global). Consider connection pooling if scaling to many concurrent clients.
- JSON serialization: Pretty-printed JSON improves readability but increases payload size; consider compact serialization for high-throughput scenarios.
- Git operations: Conflict checks spawn Git commands; cache results or limit frequency for large Review sets.
- tmux operations: Pane reads and send-keys are synchronous; batch operations may benefit from rate limiting.

## Troubleshooting Guide
Common issues and resolutions:
- Invalid project_id in global mode: Ensure project_id is provided and resolves to an existing project.
- Task not found: Verify task_id exists in the target project.
- Dependency gates: Forward transitions from Backlog require referenced_tasks to be in Review/Done.
- Transition status: Use get_transition_status to check pending/completed/error states.
- Notifications: Use get_notifications to drain and consume pending events.

**Section sources**
- [server.rs:413-429](file://src/mcp/server.rs#L413-L429)
- [server.rs:593-653](file://src/mcp/server.rs#L593-L653)
- [server.rs:686-698](file://src/mcp/server.rs#L686-L698)
- [server.rs:726-755](file://src/mcp/server.rs#L726-L755)
- [server.rs:837-858](file://src/mcp/server.rs#L837-L858)

## Conclusion
AGTX’s MCP implementation provides a robust, protocol-compliant interface for external automation and agent integration. By supporting both global and project-scoped modes, it accommodates diverse usage patterns—from orchestrator binding to ad-hoc scripting. The server’s tight integration with AGTX’s task lifecycle, Git, and tmux enables seamless orchestration of AI-assisted development workflows.

## Appendices

### MCP Server Modes Summary
- Global mode: Requires project_id; all CRUD tools need project_id.
- Project-scoped mode: Fixed path; project_id is ignored.

**Section sources**
- [CLAUDE.md:215-226](file://CLAUDE.md#L215-L226)
- [server.rs:1245-1263](file://src/mcp/server.rs#L1245-L1263)

### MCP Registration
The MCP service is registered as a stdio service named "agtx".

**Section sources**
- [.mcp.json:1-7](file://.mcp.json#L1-L7)

### Test Coverage Highlights
- TransitionRequest lifecycle: creation, pending retrieval, processing, cleanup.
- Task CRUD operations: creation, batch creation, updates, deletions.
- Project management: upsert and retrieval in global DB.
- Notifications: peek/consume semantics and ordering.

**Section sources**
- [mcp_tests.rs:5-150](file://tests/mcp_tests.rs#L5-L150)
- [mcp_tests.rs:152-356](file://tests/mcp_tests.rs#L152-L356)
- [mcp_tests.rs:357-472](file://tests/mcp_tests.rs#L357-L472)