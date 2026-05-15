# AGENTS.md

## Learnings

- **Naming**: Project was renamed from "tau" to "my-tau". Legacy naming intentionally persists in localStorage keys (`tau-file-sidebar`, `tau-favourites`, `tau-theme`, `tau-show-thinking`), icon filenames (`tau-*.png`), and CSS classes (`.tau-icon`, `.tau-icon-welcome`). New code should use `my-tau` but don't refactor old keys without a migration plan.
- **Status bar shortcut**: Pi TUI status bar uses `µτ` (U+00B5 Micro Sign + ASCII `t`) as the shortcut for my-tau, set via `ctx.ui.setStatus("µτ", ...)` in the extension.

## History

The original design spec (preserved in `.pi/AGENTS.md`, now merged here) imagined spawning `pi --mode rpc --no-session` as a subprocess with JSON-RPC over stdin/stdout. The actual implementation diverged: my-tau is a Pi extension that runs inside the Pi process, subscribing to events and forwarding them over WebSocket. No subprocess, no RPC mode.

## Project Overview

**tau** is a web mirror for Pi (pi-coding-agent) — a browser-based interface that mirrors Pi terminal sessions in real-time, with session management, favorites, and full-text search.

## Key Files

| File                          | Role                                                                                        |
| ----------------------------- | ------------------------------------------------------------------------------------------- |
| `extensions/my-tau-server.ts` | Backend — WebSocket + HTTP server, API endpoints, static file serving                       |
| `public/app.js`               | Main frontend controller                                                                    |
| `public/session-sidebar.js`   | Sidebar UI — project/session list, search, favorites, context menu (`SessionSidebar` class) |
| `public/style.css`            | All styles (~6,000+ lines), single file                                                     |
| `public/state.js`             | Centralized state management                                                                |
| `public/websocket-client.js`  | WebSocket communication                                                                     |

## UI Change Pattern

For changes spanning API → frontend rendering → styling, the three files to touch are:

1. `extensions/my-tau-server.ts` — add/modify API response fields
2. `public/session-sidebar.js` — update rendering logic
3. `public/style.css` — add/modify styles

## Session Storage Encoding

Session directories use an encoded path scheme:

- `--` prefix/suffix = root (`/`)
- `-` = path separator (`/`)

Example: `/Users/max/Projects/tau` → dir name `--Users-max-Projects-tau--`

## API: `/api/sessions`

Response shape:

```json
{
  "projects": [
    {
      "path": "/Users/max/Projects/tau",
      "displayPath": "~/Projects/tau",
      "dirName": "--Users-max-Projects-tau--",
      "sessions": [
        {
          "id": "...",
          "timestamp": "2026-01-15T10:30:00Z",
          "name": "Session Title",
          "firstMessage": "...",
          "file": "filename.jsonl",
          "filePath": "/full/path",
          "mtime": 1704276600000,
          "tmux": true
        }
      ]
    }
  ]
}
```

- `path` — full absolute path
- `displayPath` — shortened with `~` for home directory (computed server-side via `process.env.HOME`)
- Projects sorted by most recent session mtime
- Sessions sorted by mtime descending within each project

## CSS Architecture

- Single `public/style.css` file (~6,000+ lines)
- Mobile overrides live in a media query block later in the file (~line 3230+)
- Mobile `.project-header` rule only overrides `font-size`, `font-weight`, `padding`, `margin-top` — does **not** duplicate `text-transform` or `letter-spacing`
- Theme variables (e.g., `--accent-text`, `--text-dim`, `--bg-glass-hover`) drive colors

## Server Architecture

**Two-Tier Design** — my-tau uses two servers:
- **Control server** (fixed port 3001): serves static files (`/`), `/api/instances`, `/api/sessions`, `/api/health`, etc. Only one instance runs at a time per machine; standby sessions take over if the control server shuts down.
- **Communication server** (random OS-assigned port via `port: 0`): every pi session runs one. Serves `/comm/ws` (WebSocket) and `/comm/api/rpc` (HTTP RPC) for that specific session.

**Endpoint Routing** — Control server handles everything except `/comm/*`. The communication server only handles `/comm/ws` and `/comm/api/rpc`. Static files and session metadata APIs live on the control server; real-time events and commands live on the per-session comm server.

**Frontend Connection** — On load, the browser fetches `/api/instances` from the control server to discover each session's `commPort`. The WebSocket connects to `ws://host:commPort/comm/ws`. RPC calls derive their URL from `wsClient.url` (both WS and RPC are on the same comm server): `http://host:commPort/comm/api/rpc`.

## Sidebar Internals

- `SessionSidebar` class manages: project grouping, collapse/expand (via `collapsedProjects` Set), favorites (`localStorage` key `tau-favourites`), full-text search (300ms debounce), context menu
- Project headers use class `project-header`; sessions use `session-item`
- Active session tracked by `activeSessionFile` (the `filePath` of the session)
- **`.project-sessions` is shared by favourites** — the favourites group reuses `.project-sessions` as its container. CSS changes to `.project-sessions` cascade to both regular projects and favorites. Override with `.favourites-group .project-sessions` when styles should differ.
- **Session item HTML is duplicated in two methods** — `buildSessionItem()` (regular sessions) and `renderSearchResults()` (search hits) each construct session item HTML inline. When changing session item layout, update both unless divergence is intentional.
