# tmux Session Management

<cite>
**Referenced Files in This Document**
- [mod.rs](file://src/tmux/mod.rs)
- [operations.rs](file://src/tmux/operations.rs)
- [app.rs](file://src/tui/app.rs)
- [worktree.rs](file://src/git/worktree.rs)
- [config/mod.rs](file://src/config/mod.rs)
- [Cargo.toml](file://Cargo.toml)
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
This document explains how AGTX manages tmux sessions for agent coordination. It covers the dedicated tmux server architecture, session and window organization, integration with Git worktrees, and practical guidance for configuration and troubleshooting. The goal is to make tmux session management approachable for beginners while providing deep technical insights for advanced users.

## Project Structure
AGTX organizes tmux integration into focused modules:
- Dedicated tmux server for agent sessions
- Low-level tmux operations abstraction
- TUI orchestration that creates, recovers, and controls tmux windows
- Git worktree integration for task isolation
- Configuration for tmux-related UI behavior

```mermaid
graph TB
subgraph "tmux Layer"
A["tmux/mod.rs<br/>Public API: spawn/list/attach/kill/sanitize"]
B["tmux/operations.rs<br/>Trait + RealTmuxOps: window/session ops"]
end
subgraph "TUI Orchestration"
C["tui/app.rs<br/>ensure_window_or_recover()<br/>switch_agent_in_tmux()<br/>wait_for_agent_ready()"]
end
subgraph "Git Integration"
D["git/worktree.rs<br/>Worktree lifecycle<br/>per-task isolation"]
end
subgraph "Config"
E["config/mod.rs<br/>GlobalConfig<br/>fullscreen_on_enter"]
end
A --> B
C --> B
C --> A
C --> D
E --> C
```

**Diagram sources**
- [mod.rs:1-189](file://src/tmux/mod.rs#L1-L189)
- [operations.rs:1-249](file://src/tmux/operations.rs#L1-L249)
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [worktree.rs:1-345](file://src/git/worktree.rs#L1-L345)
- [config/mod.rs:1-595](file://src/config/mod.rs#L1-L595)

**Section sources**
- [mod.rs:1-189](file://src/tmux/mod.rs#L1-L189)
- [operations.rs:1-249](file://src/tmux/operations.rs#L1-L249)
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [worktree.rs:1-345](file://src/git/worktree.rs#L1-L345)
- [config/mod.rs:1-595](file://src/config/mod.rs#L1-L595)

## Core Components
- Dedicated tmux server: AGTX uses a named server to isolate agent sessions from the user's default tmux environment.
- Public tmux API: Functions for spawning sessions, listing sessions, attaching, killing, and sanitizing names.
- Window operations abstraction: A trait defines window/session operations, enabling testing and consistent behavior.
- TUI recovery and switching: Robust logic to recover missing windows, gracefully switch agents, and wait for readiness.
- Worktree integration: Tasks run in isolated worktrees, with tmux windows targeting the worktree directory for consistent agent contexts.

**Section sources**
- [mod.rs:11-189](file://src/tmux/mod.rs#L11-L189)
- [operations.rs:8-249](file://src/tmux/operations.rs#L8-L249)
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [worktree.rs:1-345](file://src/git/worktree.rs#L1-L345)

## Architecture Overview
AGTX coordinates agents through tmux windows within a dedicated server. The TUI orchestrator ensures windows exist, recovers them if lost, and switches agents safely. Worktrees provide task isolation, and configuration influences UI behavior like fullscreen attachment.

```mermaid
sequenceDiagram
participant User as "User"
participant TUI as "TUI App"
participant Ops as "TmuxOperations"
participant Real as "RealTmuxOps"
participant Agent as "Agent Ops"
participant WT as "Worktree"
User->>TUI : "Open task"
TUI->>WT : "Ensure worktree exists"
TUI->>Ops : "ensure_window_or_recover(target, agent, wt)"
Ops->>Real : "window_exists(target)?"
alt Window missing
Ops->>Real : "has_session(session)?"
alt Session missing
Ops->>Real : "create_session(session, wt_path)"
end
Ops->>Agent : "build_resume_command()"
Ops->>Real : "create_window(session, window, wt_path, resume_cmd, keep_shell=true)"
else Window exists
Note over Ops : "No action"
end
TUI->>Ops : "switch_agent_in_tmux(target, current, new)"
Ops->>Real : "send_keys(target, exit_cmd or Ctrl+C)"
Ops->>Real : "pane_current_command(target)"
Ops->>Real : "send_keys(target, new_cmd)"
TUI->>Ops : "wait_for_agent_ready(target)"
Ops->>Real : "capture_pane(target)"
TUI-->>User : "Agent ready in tmux window"
```

**Diagram sources**
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [operations.rs:62-248](file://src/tmux/operations.rs#L62-L248)
- [worktree.rs:1-345](file://src/git/worktree.rs#L1-L345)

## Detailed Component Analysis

### Dedicated tmux Server and Session Naming
- Server identifier: AGTX uses a fixed server name for agent sessions, keeping them separate from user sessions.
- Session naming: Tasks get deterministic names derived from task IDs, project names, and sanitized slugs. The system sanitizes special characters and enforces length limits.
- Public API: Functions wrap tmux CLI calls to spawn sessions, list sessions, attach, kill, capture panes, and send keys.

```mermaid
flowchart TD
Start(["Build session name"]) --> SanitizeProject["Sanitize project name"]
SanitizeProject --> SlugTitle["Slugify task title"]
SlugTitle --> Truncate["Truncate slug to limit"]
Truncate --> Compose["Compose 'task-{id}--{project}--{slug}'"]
Compose --> Validate["Validate length and safety"]
Validate --> End(["Return session name"])
```

**Diagram sources**
- [mod.rs:144-166](file://src/tmux/mod.rs#L144-L166)
- [app.rs:8579-8602](file://src/tui/app.rs#L8579-L8602)

**Section sources**
- [mod.rs:11-189](file://src/tmux/mod.rs#L11-L189)

### Window Operations Abstraction
- Trait design: A trait defines window and session operations, enabling mocking and consistent behavior across environments.
- Real implementation: Executes tmux commands for creating windows, killing windows, existence checks, sending keys, pasting text, capturing panes, resizing, and detecting pane commands.
- Key behaviors:
  - Windows are created with optional commands and working directories.
  - Pane content capture supports history for rich TUI rendering.
  - Cursor info and pane command detection support intelligent agent readiness checks.

```mermaid
classDiagram
class TmuxOperations {
+create_window(session, window_name, working_dir, command, keep_shell_on_exit) Result
+kill_window(target) Result
+window_exists(target) Result~bool~
+send_keys(target, keys) Result
+send_keys_literal(target, keys) Result
+paste_text(target, text) Result
+capture_pane(target) Result~String~
+capture_pane_with_history(target, history_lines) Vec~u8~
+get_cursor_info(target) Option~(usize, usize)~
+resize_window(target, width, height) Result
+pane_current_command(target) Option~String~
+has_session(session) bool
+create_session(session, working_dir) Result
}
class RealTmuxOps {
}
TmuxOperations <|.. RealTmuxOps
```

**Diagram sources**
- [operations.rs:8-249](file://src/tmux/operations.rs#L8-L249)

**Section sources**
- [operations.rs:8-249](file://src/tmux/operations.rs#L8-L249)

### Recovery and Agent Switching
- Window recovery: If a tmux window disappears (e.g., tmux restart or manual kill), AGTX recreates it using the agent's resume command, ensuring continuity.
- Agent switching: Gracefully terminates the current agent, detects shell prompt, and starts the new agent. It handles agent-specific exit commands and uses pane command detection to avoid false positives.
- Readiness detection: Waits for agents to finish loading by monitoring pane content and process commands, with fallbacks for universal compatibility.

```mermaid
flowchart TD
A["Window missing?"] --> |Yes| B["Create session if missing"]
B --> C["Build resume command"]
C --> D["Create window with resume command"]
A --> |No| E["Proceed"]
F["Switch agent"] --> G["Send exit command or Ctrl+C"]
G --> H["Poll pane_current_command"]
H --> |Still active| I["Retry exit or Ctrl+C"]
I --> J["Last resort: Ctrl+D"]
J --> K["Wait for shell"]
K --> L["Start new agent command"]
L --> M["Detect new process or content indicators"]
```

**Diagram sources**
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [operations.rs:213-247](file://src/tmux/operations.rs#L213-L247)

**Section sources**
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [operations.rs:62-248](file://src/tmux/operations.rs#L62-L248)

### Integration with Git Worktrees
- Isolation: Each task runs in a dedicated worktree branch and directory, ensuring clean, reproducible environments for agent work.
- Setup: Worktrees are created from a base branch, initialized with agent configuration and optional project files, and cleaned up according to configuration.
- Targeting: Tmux windows are created with the worktree directory as the working directory, aligning agent contexts with task isolation.

```mermaid
sequenceDiagram
participant TUI as "TUI App"
participant WT as "Worktree"
participant TMUX as "TmuxOperations"
TUI->>WT : "create_worktree_from_base(project, task_slug, base_branch, worktree_dir)"
WT-->>TUI : "worktree_path"
TUI->>TMUX : "create_window(session, window, worktree_path, ...)"
TMUX-->>TUI : "Window ready"
```

**Diagram sources**
- [worktree.rs:14-65](file://src/git/worktree.rs#L14-L65)
- [operations.rs:64-110](file://src/tmux/operations.rs#L64-L110)

**Section sources**
- [worktree.rs:1-345](file://src/git/worktree.rs#L1-L345)
- [operations.rs:62-110](file://src/tmux/operations.rs#L62-L110)

### Configuration Options for tmux Integration
- Fullscreen on enter: A configuration flag controls whether the TUI automatically fullscreen-attaches to tmux sessions when opening task popups.
- Worktree settings: Global and project configurations govern whether worktrees are used, automatic cleanup, base branch selection, and worktree directory location.
- Theme and UI: While not tmux-specific, the theme affects TUI presentation around tmux panes and windows.

**Section sources**
- [config/mod.rs:24-27](file://src/config/mod.rs#L24-L27)
- [config/mod.rs:160-197](file://src/config/mod.rs#L160-L197)

## Dependency Analysis
- tmux public API depends on the tmux CLI and uses a fixed server identifier.
- The TUI orchestrator depends on both the tmux operations trait and worktree utilities.
- Configuration influences UI behavior but does not alter tmux internals.
- Cargo features enable test mocks for tmux operations.

```mermaid
graph LR
Cargo["Cargo.toml features"] --> Mocks["test-mocks"]
TUI["tui/app.rs"] --> Ops["tmux/operations.rs"]
TUI --> Mod["tmux/mod.rs"]
TUI --> WT["git/worktree.rs"]
Config["config/mod.rs"] --> TUI
Mocks --> Ops
```

**Diagram sources**
- [Cargo.toml:39-41](file://Cargo.toml#L39-L41)
- [operations.rs:5-6](file://src/tmux/operations.rs#L5-L6)
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [worktree.rs:1-345](file://src/git/worktree.rs#L1-L345)
- [config/mod.rs:1-595](file://src/config/mod.rs#L1-L595)

**Section sources**
- [Cargo.toml:39-41](file://Cargo.toml#L39-L41)
- [operations.rs:5-6](file://src/tmux/operations.rs#L5-L6)
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)

## Performance Considerations
- Command overhead: Each tmux operation executes external commands. Batch operations and minimize repeated polling when possible.
- Pane capture: Capturing large histories can be expensive; use targeted line counts for typical UI updates.
- Agent readiness: Waiting loops poll pane content and commands; tune thresholds based on agent startup characteristics.
- Worktree initialization: Copying files and running scripts adds latency; leverage caching and selective copying for frequent tasks.

## Troubleshooting Guide
Common issues and solutions:
- Session conflicts
  - Symptom: Multiple tmux servers or conflicting session names.
  - Solution: Use the dedicated server name consistently; sanitize project names and slugs to avoid invalid characters.
  - Reference: [mod.rs:11-166](file://src/tmux/mod.rs#L11-L166)

- Lost windows after tmux restart
  - Symptom: Agents disappear or cannot be attached.
  - Solution: Use the recovery routine to recreate windows with the agent's resume command; ensure worktree paths exist.
  - Reference: [app.rs:8579-8602](file://src/tui/app.rs#L8579-L8602)

- Agent not responding to exit commands
  - Symptom: Busy agents require forceful termination.
  - Solution: The switching logic retries exit commands, sends Ctrl+C, and falls back to Ctrl+D; confirm agent-specific commands are used.
  - Reference: [app.rs:8616-8735](file://src/tui/app.rs#L8616-L8735)

- Pane content detection false positives
  - Symptom: Readiness checks trigger prematurely.
  - Solution: Use pane command detection alongside content checks; adjust polling intervals and stability thresholds.
  - Reference: [operations.rs:213-229](file://src/tmux/operations.rs#L213-L229)

- Worktree not found or inaccessible
  - Symptom: Tmux windows cannot be created or attached.
  - Solution: Verify worktree creation succeeded and paths are valid; reinitialize if needed.
  - Reference: [worktree.rs:25-65](file://src/git/worktree.rs#L25-L65)

- UI fullscreen behavior
  - Symptom: Unexpected fullscreen attachment behavior.
  - Solution: Adjust the configuration flag controlling fullscreen on enter.
  - Reference: [config/mod.rs:24-27](file://src/config/mod.rs#L24-L27)

**Section sources**
- [mod.rs:11-189](file://src/tmux/mod.rs#L11-L189)
- [app.rs:8579-8778](file://src/tui/app.rs#L8579-L8778)
- [operations.rs:213-248](file://src/tmux/operations.rs#L213-L248)
- [worktree.rs:25-65](file://src/git/worktree.rs#L25-L65)
- [config/mod.rs:24-27](file://src/config/mod.rs#L24-L27)

## Conclusion
AGTX’s tmux integration centers on a dedicated server, robust window recovery, and seamless agent switching. By combining deterministic session naming, worktree isolation, and careful readiness detection, it delivers a reliable environment for multi-agent coordination. Configuration options allow tuning UI behavior, while the abstraction layer ensures maintainability and testability.