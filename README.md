# agtx

A terminal-native kanban board for managing coding agent sessions with isolated git worktrees.

![agtx demo](docs/demo.gif)

## Features

- **Kanban workflow**: Backlog → Planning → Running → Review → Done
- **Git worktree isolation**: Each task gets its own worktree, keeping work separated
- **Claude Code integration**: Automatic session management with resume capability
- **PR workflow**: Generate descriptions with AI, create PRs directly from the TUI
- **Multi-project dashboard**: Manage tasks across all your projects
- **Customizable themes**: Configure colors via config file

## Installation

### Quick Install (recommended)

```bash
curl -fsSL https://raw.githubusercontent.com/USER/agtx/main/install.sh | bash
```

### From Source

```bash
git clone https://github.com/USER/agtx.git
cd agtx
cargo build --release
cp target/release/agtx ~/.local/bin/
```

### Requirements

- **tmux** - Agent sessions run in a dedicated tmux server
- **git** - For worktree management
- **gh** - GitHub CLI for PR operations
- **claude** - Claude Code CLI (optional, for Claude integration)

## Quick Start

```bash
# Run in any git repository
cd your-project
agtx

# Or run in dashboard mode (manage all projects)
agtx -g
```

## Usage

### Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `h/l` or `←/→` | Move between columns |
| `j/k` or `↑/↓` | Move between tasks |
| `o` | Create new task |
| `Enter` | Open task (view Claude session) |
| `m` | Move task forward in workflow |
| `r` | Resume task (Review → Running) |
| `d` | Show git diff |
| `x` | Delete task |
| `/` | Search tasks |
| `e` | Toggle project sidebar |
| `q` | Quit |

### Task Workflow

1. **Create a task** (`o`): Enter title and description
2. **Move to Planning** (`m`): Creates worktree, starts Claude in planning mode
3. **Move to Running** (`m`): Claude implements the plan
4. **Move to Review** (`m`): Opens PR with AI-generated description
5. **Move to Done** (`m`): Cleans up after PR is merged

### Claude Session Features

- Sessions automatically resume when moving Review → Running
- Full conversation context is preserved across the task lifecycle
- View live Claude output in the task popup

## Configuration

Config file location: `~/.config/agtx/config.toml`

```toml
# Default agent for new tasks
default_agent = "claude"

[worktree]
enabled = true
auto_cleanup = true
base_branch = "main"

[theme]
color_selected = "#FFFF99"
color_normal = "#00FFFF"
color_dimmed = "#666666"
color_text = "#FFFFFF"
color_accent = "#00FFFF"
color_description = "#E8909C"
```

## How It Works

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      agtx TUI                           │
├─────────────────────────────────────────────────────────┤
│  Backlog  │  Planning  │  Running  │  Review  │  Done   │
│  ┌─────┐  │  ┌─────┐   │  ┌─────┐  │  ┌─────┐ │         │
│  │Task1│  │  │Task2│   │  │Task3│  │  │Task4│ │         │
│  └─────┘  │  └─────┘   │  └─────┘  │  └─────┘ │         │
└─────────────────────────────────────────────────────────┘
                    │           │
                    ▼           ▼
            ┌───────────────────────────┐
            │   tmux server "agtx"      │
            │  ┌───────┐  ┌───────┐     │
            │  │Claude │  │Claude │     │
            │  │Task2  │  │Task3  │     │
            │  └───────┘  └───────┘     │
            └───────────────────────────┘
                    │           │
                    ▼           ▼
            ┌───────────────────────────┐
            │   Git Worktrees           │
            │  .agtx/worktrees/task2/   │
            │  .agtx/worktrees/task3/   │
            └───────────────────────────┘
```

### Data Storage

- **Database**: `~/Library/Application Support/agtx/` (macOS) or `~/.config/agtx/` (Linux)
- **Worktrees**: `.agtx/worktrees/` in each project
- **Tmux sessions**: Dedicated server `agtx` (view with `tmux -L agtx ls`)

> **Important**: Add `.agtx/` to your project's `.gitignore` to avoid committing worktrees and local task data.

## Development

See [CLAUDE.md](CLAUDE.md) for development documentation.

```bash
# Build
cargo build

# Run tests
cargo test

# Build release
cargo build --release
```

## License

MIT
