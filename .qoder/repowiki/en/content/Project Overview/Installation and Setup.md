# Installation and Setup

<cite>
**Referenced Files in This Document**
- [README.md](file://README.md)
- [install.sh](file://install.sh)
- [Cargo.toml](file://Cargo.toml)
- [src/main.rs](file://src/main.rs)
- [src/config/mod.rs](file://src/config/mod.rs)
- [src/agent/mod.rs](file://src/agent/mod.rs)
- [src/tmux/mod.rs](file://src/tmux/mod.rs)
- [src/tmux/operations.rs](file://src/tmux/operations.rs)
- [plugins/agtx/plugin.toml](file://plugins/agtx/plugin.toml)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [System Requirements](#system-requirements)
3. [Installation Methods](#installation-methods)
4. [Environment Preparation](#environment-preparation)
5. [Initial Configuration](#initial-configuration)
6. [First-Time Setup Wizard](#first-time-setup-wizard)
7. [Verification Steps](#verification-steps)
8. [Platform-Specific Considerations](#platform-specific-considerations)
9. [Troubleshooting Guide](#troubleshooting-guide)
10. [Conclusion](#conclusion)

## Introduction
This guide helps you install and set up AGTX quickly. It covers all supported installation methods, system requirements, environment preparation, configuration, and verification steps. AGTX orchestrates AI coding agents in a terminal kanban board, using tmux for persistent agent sessions and optional GitHub CLI integration for pull request operations.

## System Requirements
- tmux: Required for agent sessions and persistent workspaces
- git: Required for worktree management and repository operations
- gh (optional): GitHub CLI for PR operations
- claude (optional): Claude Code CLI for MCP integration

These requirements are validated during installation and when running AGTX.

**Section sources**
- [README.md:100-104](file://README.md#L100-L104)
- [install.sh:145-171](file://install.sh#L145-L171)

## Installation Methods

### Curl Installer Script
The simplest way to install AGTX is using the official installer script. It detects your OS and architecture, downloads the appropriate release binary, installs it to your local bin directory, and checks dependencies.

- Run the installer:
  ```bash
  curl -fsSL https://raw.githubusercontent.com/fynnfluegge/agtx/main/install.sh | bash
  ```

- The installer:
  - Detects OS (Linux/Darwin) and architecture (x86_64/aarch64)
  - Downloads the latest release tarball from GitHub
  - Installs the binary to ~/.local/bin/
  - Checks PATH and suggests adding ~/.local/bin/ if missing
  - Verifies tmux, git, gh, and claude availability

- After installation, run AGTX in any git repository:
  ```bash
  cd your-project && agtx
  ```

**Section sources**
- [README.md:73-98](file://README.md#L73-L98)
- [install.sh:1-176](file://install.sh#L1-L176)

### Manual Compilation from Source
If you prefer building from source or need to customize the build:

- Prerequisites:
  - Rust toolchain (cargo/rustc)
  - curl and tar (for downloading dependencies)
  - tmux, git, and optionally gh/claude for full functionality

- Steps:
  - Build the release binary:
    ```bash
    cargo build --release
    ```
  - Install to ~/.local/bin/:
    ```bash
    cp target/release/agtx ~/.local/bin/
    ```

- Verify installation:
  ```bash
  agtx --help
  ```

**Section sources**
- [README.md:94-98](file://README.md#L94-L98)
- [Cargo.toml:1-50](file://Cargo.toml#L1-L50)

### Package Manager Options
There is no dedicated package manager distribution documented in the repository. The recommended approaches are the curl installer or building from source.

**Section sources**
- [README.md:73-98](file://README.md#L73-L98)

## Environment Preparation
- Ensure ~/.local/bin is in your PATH. The installer will warn if it is not and provide shell-specific export commands for bash and zsh.
- Install tmux, git, and optionally gh and claude as needed for your workflow.
- On macOS, you may need to allow terminal applications to control your computer in System Settings under Privacy & Security for full tmux integration.

**Section sources**
- [install.sh:120-132](file://install.sh#L120-L132)
- [install.sh:145-171](file://install.sh#L145-L171)

## Initial Configuration
AGTX stores configuration in two locations:

- Global configuration:
  - Location: ~/.config/agtx/config.toml
  - Purpose: Default agent selection, per-phase agent overrides, worktree settings, theme, and UI preferences

- Project-level configuration:
  - Location: .agtx/config.toml in your project root
  - Purpose: Overrides for base branch, worktree directory, copy_files, init_script, cleanup_script, and workflow plugin

### Global Config Example Fields
- default_agent: Default coding agent for new tasks
- agents: Per-phase agent overrides (research, planning, running, review)
- worktree: base_branch, worktree_dir, enable/disable, auto_cleanup
- theme: Color customization for UI elements
- fullscreen_on_enter: Auto-fullscreen when opening a task popup

### Project Config Example Fields
- base_branch: Override base branch for worktrees
- worktree_dir: Directory for worktrees (relative to project root)
- copy_files: Comma-separated list of files to copy into worktrees
- init_script: Shell command to run inside worktree after creation
- cleanup_script: Shell command to run inside worktree before removal
- workflow_plugin: Name of the plugin to use by default

**Section sources**
- [README.md:261-303](file://README.md#L261-L303)
- [src/config/mod.rs:5-274](file://src/config/mod.rs#L5-L274)

## First-Time Setup Wizard
On first run, AGTX determines the appropriate action based on existing configuration and data:

- If a config file exists or was migrated: no action
- If data exists but no config: saves defaults silently
- If no config and no data: prompts you to select a default coding agent

The wizard presents available agents detected on your system and lets you choose the default. Supported agents include Claude, Codex, Copilot, Gemini, OpenCode, and Cursor.

- Agent detection uses which::which to check for executables
- The prompt supports keyboard navigation (arrow keys/j/k) and Enter to confirm

**Section sources**
- [src/main.rs:63-89](file://src/main.rs#L63-L89)
- [src/config/mod.rs:305-335](file://src/config/mod.rs#L305-L335)
- [src/agent/mod.rs:124-130](file://src/agent/mod.rs#L124-L130)

## Verification Steps
After installation, verify your setup:

- Confirm AGTX binary is available:
  ```bash
  which agtx
  ```
- Test help output:
  ```bash
  agtx --help
  ```
- Run in dashboard mode:
  ```bash
  agtx -g
  ```
- In a git repository:
  ```bash
  cd your-project && agtx
  ```
- Verify tmux server:
  ```bash
  tmux -L agtx list-sessions
  ```

**Section sources**
- [README.md:73-98](file://README.md#L73-L98)
- [src/tmux/mod.rs:50-84](file://src/tmux/mod.rs#L50-L84)

## Platform-Specific Considerations
- macOS:
  - Ensure tmux is installed and accessible
  - If you encounter permission prompts, grant accessibility permissions for Terminal.app in System Settings under Privacy & Security
- Linux:
  - Confirm tmux is installed and available in PATH
  - Some distributions may require installing tmux via your package manager
- Windows:
  - AGTX targets Unix-like environments; use WSL or a compatible environment for tmux and agent CLIs

**Section sources**
- [install.sh:35-53](file://install.sh#L35-L53)
- [README.md:100-104](file://README.md#L100-L104)

## Troubleshooting Guide

### Installation Issues
- Unsupported OS or architecture:
  - The installer detects Linux/Darwin and x86_64/aarch64. Other combinations will fail early with an unsupported message.
- Missing curl:
  - The installer requires curl; install it via your system’s package manager and rerun the script.
- Download failures:
  - Check network connectivity and the release URL. The installer prints the exact URL it attempts to fetch.
- PATH not updated:
  - The installer warns if ~/.local/bin is not in PATH. Add the export line shown by the installer to your shell profile.

**Section sources**
- [install.sh:35-53](file://install.sh#L35-L53)
- [install.sh:76-87](file://install.sh#L76-L87)
- [install.sh:100-103](file://install.sh#L100-L103)
- [install.sh:120-132](file://install.sh#L120-L132)

### Runtime Issues
- tmux not found:
  - Install tmux and ensure it is in PATH. AGTX requires tmux to manage agent sessions.
- git not found:
  - Install git; AGTX uses git for worktree management and repository operations.
- gh not found (optional):
  - Install GitHub CLI if you plan to use PR operations. Without it, PR-related features are unavailable.
- claude not found (optional):
  - Install Claude Code CLI if you want MCP integration for the sweep/brainstorm skills.

**Section sources**
- [install.sh:145-171](file://install.sh#L145-L171)
- [README.md:100-104](file://README.md#L100-L104)

### Configuration Issues
- Config location changes:
  - AGTX migrates from legacy locations to ~/.config/agtx/config.toml. If you previously used an older version, your settings may be moved automatically.
- Default agent selection:
  - If no agent is detected, AGTX falls back to a default. Install one of the supported agents or manually edit ~/.config/agtx/config.toml.

**Section sources**
- [src/main.rs:98-119](file://src/main.rs#L98-L119)
- [src/agent/mod.rs:124-130](file://src/agent/mod.rs#L124-L130)

### tmux Integration Problems
- Session server:
  - AGTX uses a dedicated tmux server named “agtx”. If sessions are missing, verify the server exists and is accessible.
- Session naming:
  - AGTX sanitizes project names for tmux session names. If a session fails to create, check for invalid characters or extremely long names.

**Section sources**
- [src/tmux/mod.rs:50-84](file://src/tmux/mod.rs#L50-L84)
- [src/tmux/mod.rs:144-165](file://src/tmux/mod.rs#L144-L165)

## Conclusion
You now have multiple paths to install AGTX, understand system requirements, and configure both global and project-level settings. Use the curl installer for speed, compile from source for customization, and verify your setup with the provided steps. If issues arise, consult the troubleshooting section tailored to installation and runtime concerns.