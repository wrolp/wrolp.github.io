# Wrolp Terminal

A Rust-powered desktop SSH terminal and server-ops tool — multi-tab, SFTP, remote editor, session recording, Docker analysis & AI assistant.

## At a glance

| Item | Value |
| --- | --- |
| Repository | https://github.com/wrolp/wrolp |
| Mirror | https://gitee.com/wrolp/wrolp |
| Mirrored releases | https://gitee.com/wrolp/wrolp/releases |
| Current version | v0.0.13 (released 2026-11-16) |
| Stack | Tauri 2 / React 19 / TypeScript / Rust |
| License | MIT (Copyright (c) 2026 Wrolp) |
| Platforms | Windows is the primary target (release packages are MSI only); it can run and be developed on Linux/macOS |

## What it is and the problem it solves

Wrolp Terminal is a desktop terminal and server-operations tool built on Tauri 2 + React 19 + TypeScript. It brings "connect to servers, transfer files, edit files, read logs, manage containers, manage K8s, manage Git, ask an AI" into a single application window, replacing a workflow that otherwise bounces between an SSH client, an SFTP tool, an editor, a Docker dashboard and a Git GUI.

Its key difference from conventional SSH clients: SSH connections are implemented directly on the Rust side with the pure-Rust async library `russh` (with no dependency on an external `ssh` command or the OpenSSH binary), so terminal, SFTP, tunnels, Docker exec and kubectl exec all reuse the same session channel. Local and remote file browsing share a single `RemoteFs` abstraction, so a local shell and remote hosts work side by side in one UI.

## Core features

The list below groups the implemented capabilities by workflow.

### Connection and terminal

- **Multi-tab SSH terminal**: rendered with xterm.js, supporting password and SSH-key authentication, an `xterm-256color` PTY and window-size synchronisation.
- **Connection health**: configurable SSH keepalive (interval plus retry count); the tab status dot goes from green to yellow (`connection-suspect`) and then red (`connection-closed`); a "Reconnect" button reuses the same tab instance, and a monotonically increasing `session_id` prevents stale background tasks from polluting state.
- **Telnet**: plain TCP with a built-in IAC negotiation state machine (ECHO / SGA / TERMINAL-TYPE / NAWS); optional auto-login, off by default (plain-text protocol).
- **Serial (COM)**: enumerates ports and shows USB device friendly names with VID:PID; configurable baud rate, data bits, stop bits, parity and flow control; includes **baud-rate auto-detection** (trying each rate in turn and ranking them by "how much it looks like terminal text").
- **Local terminal**: opens the local shell as a top-level tab or a split pane, side by side with remote tabs; it belongs to the workspace that created it, and each group's collapsed state persists across restarts.
- **Prompt-aware terminal**: prompts are read as rendered rows, so zsh + powerlevel10k and CJK prompts keep `ls` links, command history and `cd` tracking; starship decorations are not taken for a working directory, and emoji use Unicode 11 widths.
- **Terminal appearance**: font family (the Nerd Font stays in the fallback chain), size, line height and cursor shape, with a live preview.
- **`cat` highlighting**: command output is coloured from its content as well as its file name; an unknown name falls back to the editor's language table, and `ssh_config` has its own tokenizer.
- **SSH tunnels**: local port-forwarding rules saved per connection, which can be added, edited, deleted, started and stopped from the sidebar; the backend syncs state through the `tunnel-changed` event.
- **Network scan**: scans a subnet for hosts with open SSH/Telnet ports and can create connections directly from the results.

### Files and editing

- **SFTP file management**: remote file tree, listing directories / read / write / rename / delete / create directory, upload and download; supports **pause and resume**, with `transfer-progress` events.
- **Transfer queue**: one row per file showing completed/total size and live speed; directories and drag-and-drop uploads are walked once on the Rust side with `walkdir`, with progress reported in 4 MiB chunks (single-file uploads use 64 KB chunks); **cancelling an upload cleans up the partial remote files it created**, completed files are kept, and target files that already existed are not deleted.
- **Remote editor**: embedded Monaco editing, presented in a split pane or a floating window; UTF-8 / GBK encoding auto-detection (`encoding_rs`).
- **Editor languages**: content-based detection (shebang, Vim/Emacs modeline, first-line markup) when the name says nothing, plus name-table coverage for systemd units, shell rc files, `.env.*` and `ssh_config`; an SSH Config grammar and Ctrl+/ line comments in nginx / properties / sshconfig / Makefile / Dockerfile.
- **HTML preview**: HTML files open in a sandboxed iframe, so their scripts cannot reach the application.
- **Sudo and other-user saves**: stage the bytes over SFTP and write with `sudo -S cp` on an exec channel (password only over stdin, temp file removed), or save as another verified user; a failed save stays in a banner so typed text is not lost.
- **Hex and image viewing**: binary files are viewed as a hex dump, and images are previewed inline by MIME type.
- **Network file tools**: a built-in FTP server (PASV/EPSV), an HTTP/HTTPS file server (one-click generation of a self-signed certificate), a TFTP server, plus an FTP client (no TLS / explicit / implicit) and a TFTP client.

### Operations panels

- **Docker and host analysis**: one command surface supports two kinds of host (`DockerHostRef`) — containers on an SSH session jump host (running `docker exec` over the exec channel), or the local daemon (spawning the CLI directly to avoid quoting issues); `probe_local_docker` tries `docker` and then `podman`, and distinguishes "not installed" from "installed but the daemon is not running". Features include a container list with create/delete/start/stop, one-shot and streaming logs, a three-layer analysis report, container file browsing and entering a container shell; a local `docker exec` / `kubectl exec` shell browses the container's filesystem, with the inner cwd detected after the shell opens and its directory map prefetched so a bare `ls` is coloured. There is also `analyze_host`, a read-only analysis of the server's OS/kernel/architecture/packages/installed tools, and `command_help` for looking up command help.
- **Kubernetes and Helm**: K8s mode follows the currently focused terminal to choose the host (an SSH jump host or the local kubeconfig); context/namespace selectors plus a search box, defaulting to **All namespaces**, with namespace labels stably colored by name. Ten resource kinds (pods/deployments/statefulsets/daemonsets/services/ingresses/jobs/cronjobs/configmaps/secrets) share one row model; each row supports viewing/editing and applying YAML, scaling, deleting (with an in-app confirmation), viewing/streaming logs (with ANSI colors), entering a Pod shell, browsing Pod files and port forwarding (rules can be saved); the Helm tab supports CLI detection, listing releases, viewing values and revision history, rollback, uninstall, and install/upgrade.
- **Git panel**: manages one repository at a time, with the host likewise following the focused terminal; everything calls the real `git` CLI (remote via the exec channel with `env LC_ALL=C GIT_TERMINAL_PROMPT=0 … bash -lc`, local spawned directly with `HideConsole`). It covers status (porcelain v2), line-by-line diff (2 MB cap), paginated history with commit-graph lanes, branch and remote management, stage/unstage/discard, commit/amend, fetch/pull/push (pull forces `--ff-only`, and force push only offers `--force-with-lease` and requires a second confirmation), a three-way conflict view with operation abort, stash, blame, clone (with progress), repository scanning, `git init`, and a "local ignore" that writes only to `.git/info/exclude`. An AI can draft a Conventional Commits message from the staged diff (nothing commits until Commit is pressed); clone destinations follow the URL's repository name, show git's progress, can be cancelled and clean up only the tree they created; the commit graph and branch/tag labels are coloured, history pages from 100 to 1000 with a Load all action, and repository scanning walks dot-directories. Path arguments are never shell-concatenated; the backend rejects absolute paths, drive letters, backslashes and `..`.
- **Session recording**: off by default; it can be enabled in settings or forced with the `WROLP_RECORDING=1/0` environment variable. Events are written per session to `<data dir>/recordings/<workspace>/<group>/<connection>/<timestamp>_<session8>.jsonl` (NDJSON), flushed every 5 seconds and on disconnect; SQLite stores only the index (`events_file`, `event_count`), so `wrolp.db` does not grow with recordings. A bottom panel (Recordings) lets you browse recordings, reveal them in the file manager, export to asciinema v2 (`.cast`), replay and delete, and it supports legacy-data migration and index rebuilding by directory.
- **Command sets**: reusable command groups stored in SQLite (bindable to connections), sorted by Manual / Name / Recent with a reset; snippets drag-reorder inside the connection view, each group header carries a “+” already scoped to that group, and the command field explains the `${name}` placeholder.

### AI assistant

- Two modes: **Chat** (non-streaming `ai_chat_sync`) and **Agent** (streaming plus tool calling via `run_agent_stream`); Agent can call tools on connected servers.
- **Isolated dual chat**: the AI in the inspector sidebar is an application-level conversation (it belongs to no terminal, and the backend refuses all of its shell tools); terminal-bound AI docks inside each pane and can run `run_command`, read files and "send to terminal".
- **Multimodal input**: messages can carry images (the backend receives them as `openai_content` multi-part structures), images can also be pasted, and clicking a thumbnail opens the full image.
- **Send to terminal**: AI code blocks or selected text are injected into the target pane according to connection type (SSH/local shell/serial/Telnet); it only inserts and never executes, so you can review before running.
- **Templates and resend**: a template dropdown inserts prompts; any historical message can be edited and resent; `cancel_ai_chat` aborts in-flight generation.
- **Multiple endpoints**: several `AiEndpointProfile`s can be saved (each with its own encrypted API key, URL, model and system prompt), and model lists can be fetched from `{endpoint}/v1/models`.
- **Security**: API keys are encrypted with an AES-256-GCM file vault (`vault.key` is a machine-bound file); the OS keychain is not used.

### Interaction and data

- **Split trees and OS-window panes**: the terminal area is a split-tree layout that splits horizontally or vertically, with leaf nodes holding a terminal, Docker logs or the editor; a pane can pop out into a real undecorated OS window that carries its screen and tab context, docking back keeps the scrollback, and the float window brings that pane's recording, reconnect and session-bound AI column with it. A window that takes a live pane attaches to the session instead of reconnecting; each session's open files and logs stay isolated.
- **Minimal mode**: Ctrl+Shift+Z (or the titlebar button) unmounts the titlebar, rail, sidebar, tab bar, panels, inspector and status bar so the window belongs to the terminal; the saved layout returns on exit, and `ui.minimalFullscreen` can take OS fullscreen.
- **Themed controls**: select lists and tooltips are drawn in-app instead of by the OS; long lists use the shared scrollbar thumb, tooltips follow the theme, nested titles never double up, and a menu always paints above its row. Destructive menu actions are red.
- **Status bar and drawer**: the bottom drawer can be hidden from the status bar; with it hidden, Ctrl+J opens it expanded.
- **Relocatable data directory**: settings can point the data directory at any location, migrating `wrolp.db`, `recordings/`, `connections.json` and the window configuration together; the choice is recorded in `data_root.json` at the default location and takes effect on the next start; an empty target is copied into, a non-empty one is reused, and existing data is never overwritten. `vault.key` can optionally be migrated too, making the whole directory portable.
- **Database maintenance**: shows the current size of `wrolp.db` and the space that can be reclaimed, with a "Compact now" (`VACUUM`) action.
- **Keybindings**: all shortcuts live in a single table in `src/lib/keybindings.ts`; settings allow rebinding each entry, resetting a single one, or resetting everything; reserved keys (Tab, Enter, Ctrl+C/V/X/Z/D/W) are refused.
- **Other**: custom title bar, persisted window geometry and opacity, single-instance protection, an auto-updater (GitHub Release endpoint, which requires a real `pubkey` in `tauri.conf.json`), and a Chinese/English UI (following the system language by default).

## Tech stack

**Frontend**: React 19 + TypeScript + xterm.js 5.5 (fit/serialize/webgl addons) + Monaco Editor + Vite 6 + SCSS. Dependencies are deliberately few; runtime dependencies are essentially only the Tauri API, xterm, Monaco, react-markdown/remark-gfm, uuid and similar.

**Backend**: Tauri 2 + Rust (tokio). Key crates:

- `russh` 0.64 + `russh-sftp` 3 (pure-Rust async SSH / SFTP)
- `rusqlite` 0.31 (bundled SQLite, WAL mode)
- `portable-pty`, `serialport` (local PTY and serial ports)
- `aes-gcm`, `rand`, `base64` (key vault)
- `axum` + `tower-http` + `axum-server` + `rcgen` (built-in HTTP/HTTPS file server and self-signed certificates)
- `suppaftp` (FTP), `reqwest` (AI endpoint calls), `encoding_rs`, `walkdir`, `chrono`
- Windows-specific: `windows` / `windows-core` / `webview2-com`

**IPC and storage**: Tauri `invoke` commands are the primary mechanism, and terminal output is polled from the frontend (a workaround for Windows background-task limits). Tauri events are used only for pushes the UI cannot poll — `connection-closed`, `connection-suspect`, `connection-ok`, `transfer-progress`, `tunnel-changed`, `ai-term-mark`, `native-drag-drop`, `baud-detect-progress`. All data lives in one relocatable directory: `window.json` (window, recording and keepalive settings), the encrypted `connections.json` (connections and workspaces), `wrolp.db` (session index, command sets, snippets), and `recordings/` (NDJSON recordings).

## Getting started

Environment dependencies (Linux Debian/Ubuntu; macOS and Windows need no extra system dependencies):

```bash
sudo apt-get install -y pkg-config libdbus-1-dev libssl-dev libgtk-3-dev \
  libjavascriptcoregtk-4.1-dev libsoup-3.0-dev libwebkit2gtk-4.1-dev
```

Install Rust and the frontend dependencies with yarn:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
yarn
```

Run in development:

```bash
yarn tauri dev      # full stack (Tauri starts the Vite dev server automatically)
yarn dev            # frontend only, for pure UI debugging (no Rust/SSH)
```

Build:

```bash
yarn build:clean    # build after clearing the separate target directory; output in src-tauri/target-release/release/bundle/
yarn tauri build    # incremental development build; output in src-tauri/target/
```

Release bundles build in their own target directory, so a clean is not blocked by rust-analyzer holding files open;
after a build the release target keeps only `release/`.

Publish:

```bash
gh auth login -h github.com
yarn run publish    # build first, then upload to a GitHub Release; RELEASE_SKIP_CLEAN=1 reuses existing artifacts
```

Tests:

```bash
cd src-tauri && cargo check && cargo clippy && cargo test
cd src-tauri && cargo run --bin ssh_test   # a standalone russh connectivity probe
npx playwright install && yarn test:e2e    # Playwright frontend E2E; the backend is stubbed via window.__TAURI_INTERNALS__, so no Rust build is needed
```

## Repository layout

```
├── src/                    # React frontend
│   ├── components/         # Terminals, file panel, Docker/K8s/Git panels, AI chat, floating windows, etc.
│   │   ├── terminal/       # Terminal enhancements: highlighting, completion, cwd sync, line numbers, newline detection, etc.
│   │   └── nettools/       # FTP/HTTP/TFTP client and server panels
│   ├── editor/             # Monaco initialization and custom languages (including shell syntax)
│   ├── hooks/ lib/         # Keybindings, themes, transfer queue, Git/K8s helper logic
│   ├── i18n/               # Chinese and English strings
│   └── styles/             # SCSS partials split by feature
├── src-tauri/              # Rust backend
│   ├── src/commands/       # Tauri commands split by protocol/domain (ssh, sftp, docker, k8s, git, ftp, serial, etc.)
│   ├── src/git/            # Git parsing, status, branches, conflicts, history, remotes, stash, scanning
│   ├── src/*_fs.rs         # RemoteFs abstraction: local / remote(SFTP) / docker / k8s / wsl / ftp
│   ├── src/k8s_host.rs     # Unified K8s resource model and host abstraction
│   ├── src/vault.rs        # AES-256-GCM file vault
│   ├── src/schema.sql      # SQLite schema
│   └── tauri.conf.json     # Packaging config (bundle target is msi only)
├── e2e/ui/                 # Playwright specs (over 130 spec files) and Tauri IPC stubs
├── tests/                  # Pure-logic frontend check scripts (ls parsing, paste protection, highlighting rules, etc.)
├── scripts/                # Build, publish, icons, latest.json generation
├── docs/                   # User guide (Chinese and English) and screenshots
└── README.md / README.zh.md / RELEASE_NOTES.md / RELEASE_NOTES.zh.md
```

## Who it is for

- Operations engineers and developers who need to manage several Linux servers at once from a Windows desktop, and would rather not switch between an SSH client, WinSCP, an editor, Docker Desktop, a Git GUI and a kubectl terminal.
- People who repeatedly chain actions such as "connect, check logs, edit a config file, pull code, restart a container", and want those actions to stay in one window.
- Anyone who wants an AI assistant to run diagnostic commands and read files on connected hosts, while keeping a human review step before anything executes.
- Scenarios that need session recording and replay for auditing or post-mortems (NDJSON on disk, exportable to asciinema).
- Cases where application data (including encrypted connection configuration) must be kept in a custom or portable directory.

## Before you start

- **Release packages are MSI only** (the bundle target in `tauri.conf.json` is `["msi"]`); Linux and macOS users build from source.
- **Tunnels are local port forwards**: a listening port on your machine relays traffic through the SSH session to a remote target.
- **Session recording is off by default**: once enabled, keystrokes and commands go to the local SQLite database and are never uploaded.
- **The data directory is relocatable**, but `vault.key` decrypts connection profiles and AI keys, so keep it with any backup or migration.
- **Operations panels need remote tools**: the Docker panel requires Docker or podman on the host, and the Kubernetes and Git panels run `kubectl`, `helm` and `git` through the focused terminal's session.

## Contributing

- Rust commands register in `generate_handler!` inside `lib.rs` and get a frontend wrapper in `src/commands.ts`.
- Rust structs use `#[serde(rename_all = "camelCase")]` so their JSON matches the TypeScript types.
- Formatting follows Prettier; run `yarn lint` and `yarn test:e2e` before opening a pull request.
