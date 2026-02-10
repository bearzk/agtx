# AGTX - Terminal Kanban for Coding Agents

A terminal-native kanban board for managing multiple coding agent sessions (Claude Code, Aider, etc.) with isolated git worktrees.

## Quick Start

```bash
# Build
cargo build --release

# Run in a git project directory
./target/release/agtx

# Or run in dashboard mode (no git project required)
./target/release/agtx -g
```

## Architecture

```
src/
├── main.rs           # Entry point, CLI arg parsing, AppMode enum
├── tui/
│   ├── mod.rs        # Re-exports
│   ├── app.rs        # Main App struct, event loop, rendering (largest file)
│   ├── board.rs      # BoardState - kanban column/row navigation
│   └── input.rs      # InputMode enum for UI states
├── db/
│   ├── mod.rs        # Re-exports
│   ├── schema.rs     # Database struct, SQLite operations
│   └── models.rs     # Task, Project, TaskStatus enums
├── tmux/
│   └── mod.rs        # Tmux server "agtx", session management
├── git/
│   ├── mod.rs        # is_git_repo helper
│   └── worktree.rs   # Git worktree create/remove/list
├── agent/
│   └── mod.rs        # Agent definitions, detection, spawn args
└── config/
    └── mod.rs        # GlobalConfig, ProjectConfig, ThemeConfig, MergedConfig
```

## Key Concepts

### Task Workflow
```
Backlog → Planning → Running → Review → Done
            ↓           ↓         ↓        ↓
         worktree    Claude    PR opens  cleanup
         + Claude    working   (can      (keep
         planning             resume)    branch)
```

- **Backlog**: Task ideas, not started
- **Planning**: Creates git worktree at `.agtx/worktrees/{slug}`, starts Claude Code in planning mode
- **Running**: Claude is implementing (sends "proceed with implementation")
- **Review**: PR is opened. Can move back to Running to address feedback (resumes Claude session)
- **Done**: PR merged/closed, cleanup worktree + tmux (branch kept for potential reopen)

### Claude Session Resume
- When Claude starts, session is renamed with `/rename {task_id}`
- When resuming from Review → Running, uses `claude --resume {task_id}`
- This preserves full conversation context across task lifecycle

### Database Storage
All databases stored centrally (not in project directories):
- macOS: `~/Library/Application Support/agtx/`
- Linux: `~/.config/agtx/`

Structure:
- `index.db` - Global project index
- `projects/{hash}.db` - Per-project task database (hash of project path)

### Tmux Architecture
- All agent sessions run in tmux server named `agtx` (`tmux -L agtx`)
- Window naming: `{project}:task-{title_slug}`
- Separate from user's regular tmux sessions
- View sessions: `tmux -L agtx list-windows`

### Theme Configuration
Colors configurable via `~/.config/agtx/config.toml`:
```toml
[theme]
color_selected = "#FFFF99"     # Selected elements (light yellow)
color_normal = "#00FFFF"       # Normal borders (cyan)
color_dimmed = "#666666"       # Inactive elements (gray)
color_text = "#FFFFFF"         # Text (white)
color_accent = "#00FFFF"       # Accents (cyan)
color_description = "#E8909C"  # Task descriptions (rose)
```

## Keyboard Shortcuts

### Board Mode
| Key | Action |
|-----|--------|
| `h/l` or arrows | Move between columns |
| `j/k` or arrows | Move between tasks |
| `o` | Create new task |
| `Enter` | Open task popup (tmux view) |
| `i` | Edit task (backlog only) |
| `x` | Delete task |
| `d` | Show git diff for task |
| `m` | Move task forward (advance workflow) |
| `r` | Resume task (Review → Running) |
| `/` | Search tasks (jumps to and opens task) |
| `e` | Toggle project sidebar |
| `q` | Quit |

### Task Popup (tmux view)
| Key | Action |
|-----|--------|
| `Ctrl+j/k` | Scroll up/down |
| `Ctrl+g` | Jump to bottom |
| `Ctrl+q` | Close popup |
| Other keys | Forwarded to tmux/Claude |

### PR Creation Popup
| Key | Action |
|-----|--------|
| `Tab` | Switch between title/description |
| `Ctrl+s` | Create PR and move to Review |
| `Esc` | Cancel |

## Code Patterns

### Ratatui TUI
- Uses `crossterm` backend
- State separated from terminal for borrow checker: `App { terminal, state: AppState }`
- Drawing functions are static: `fn draw_*(state: &AppState, frame: &mut Frame, area: Rect)`
- Theme colors accessed via `state.config.theme.color_*`

### Error Handling
- Use `anyhow::Result` for all fallible functions
- Use `.context()` for adding context to errors
- Gracefully handle missing tmux sessions/worktrees

### Database
- SQLite via `rusqlite` with `bundled` feature
- Migrations via `ALTER TABLE ... ADD COLUMN` (ignores errors if column exists)
- DateTime stored as RFC3339 strings

### Background Operations
- PR description generation runs in background thread
- PR creation runs in background thread
- Uses `mpsc` channels to communicate results back to main thread
- Loading spinners shown during async operations

### Claude Integration
- Uses `--dangerously-skip-permissions` flag
- Polls tmux pane for "Yes, I accept" prompt before sending acceptance
- Sends `/rename {task_id}` after Claude starts for session resume capability

## Building

```bash
cargo build --release
```

Dependencies require:
- Rust 1.70+
- SQLite (bundled via rusqlite)
- tmux (runtime dependency)
- git (runtime dependency)
- gh CLI (for PR operations)
- claude CLI (Claude Code)

## Testing

```bash
cargo test
```

## Common Tasks

### Adding a new task field
1. Add field to `Task` struct in `src/db/models.rs`
2. Add column to schema and migration in `src/db/schema.rs`
3. Update `create_task`, `update_task`, `task_from_row` in schema.rs
4. Update UI rendering in `src/tui/app.rs`

### Adding a new theme color
1. Add field to `ThemeConfig` in `src/config/mod.rs`
2. Add default function and update `Default` impl
3. Use `hex_to_color(&state.config.theme.color_*)` in app.rs

### Adding a new agent
1. Add to `known_agents()` in `src/agent/mod.rs`
2. Add spawn args handling in `build_spawn_args()`
3. Add resume args if supported in `build_resume_args()`

### Adding a keyboard shortcut
1. Find the appropriate `handle_*_key` function in `src/tui/app.rs`
2. Add match arm for the new key
3. Update help/footer text if visible to user

### Adding a new popup
1. Add state struct (e.g., `MyPopup`) in app.rs
2. Add `Option<MyPopup>` field to `AppState`
3. Add rendering in `draw_board()` function
4. Add key handler function `handle_my_popup_key()`
5. Add check in `handle_key()` to route to handler

## Future Enhancements
- Auto-detect Claude idle status (show spinner when working)
- Reopen Done tasks (recreate worktree from preserved branch)
- Support for additional agents (Aider, Codex)
- Notification when Claude finishes work
