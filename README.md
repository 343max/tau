# my-tau

A web UI for your [Pi](https://github.com/badlogic/pi-mono) terminal session in the browser. No separate server — it runs as a Pi extension inside your existing process.

> **Note:** my-tau is a fork of [tau](https://github.com/deflating/tau) by @deflating.

![my-tau dark mode](docs/images/dark.png)

![my-tau terracotta theme](docs/images/terracotta.png)

![Settings](docs/images/settings.png)

![Commands](docs/images/commands.png)

## What it does

my-tau connects to your running Pi TUI and gives you a second view in the browser. Same session, same messages, same tools — just a different screen. Type in the terminal or the browser, both stay in sync.

- **Live mirroring** — streams messages, tool calls, and thinking blocks in real-time
- **Works on any device** — open it on your phone, tablet, or another monitor
- **Session browser** — view history from any past session
- **No extra process** — the Pi extension _is_ the server

## Install

```bash
pi install git:github.com/343max/tau
```

## Usage

1. Start Pi normally in your terminal
2. Open the URL shown in the status bar
3. That's it

Type `/my-tau` in the terminal to open the UI in your browser, or `/qr` to show a QR code and scan it to access via your phone.

## Features

### Chat

- Full markdown rendering with syntax-highlighted code blocks
- Streaming responses with typing indicator
- Image attachments (paste, drag & drop, or button)
- Copy any message with one click
- Inline diff viewer for edit tool calls (red/green lines)
- Scroll-to-bottom button with new message indicator
- Message queuing — type while the agent is working, messages queue and auto-send

### Session Management

- Browse all past sessions grouped by project
- Full-text search across all session history with highlighted snippets
- Sorted by last modified (most recent first)
- Live session marked with a green dot
- Historical sessions are read-only
- Inline session rename
- Favourite sessions, tags, and filtering

### Model & Thinking

- Model picker with search/filter and keyboard support
- Thinking level toggle (off/low/medium/high)
- Token usage percentage with context window visualiser
- Cost tracking per session

### Voice Input

- Mic button in the input area using Web Speech API (on-device dictation)
- Live transcription into the textarea
- Pulses red while recording

### File Browser

- Right sidebar with lazy-loaded file tree
- Navigate directories, open files natively
- Drag files onto the input to insert their path

### Compaction

- Manual context compaction with status display
- Auto-compaction support

### PWA

- Installable as a standalone app on iOS, Android, and macOS
- Custom app icons
- Service worker with network-first caching

## Configuration

Environment variables (set before starting Pi):

| Variable             | Default     | Description                                                                        |
| -------------------- | ----------- | ---------------------------------------------------------------------------------- |
| `MY_TAU_MIRROR_PORT` | `3001`      | Server port                                                                        |
| `MY_TAU_STATIC_DIR`  | _(bundled)_ | Override static files path                                                         |
| `MY_TAU_DISABLED`    | `0`         | Set to `1` to disable my-tau (it stays installed but won't start the server)       |
| `MY_TAU_USER`        | _(none)_    | HTTP Basic Auth username (both `MY_TAU_USER` and `MY_TAU_PASS` required to enable) |
| `MY_TAU_PASS`        | _(none)_    | HTTP Basic Auth password                                                           |

### Authentication

my-tau supports optional HTTP Basic Auth (browser-native login popup).

**1. Set credentials** — add to `~/.pi/agent/settings.json`:

```json
{
  "my-tau": {
    "user": "pi",
    "pass": "your-password"
  }
}
```

Or via environment variables: `MY_TAU_USER=pi MY_TAU_PASS=secret pi`

**2. Toggle on/off** — once credentials are configured, a "Require login" toggle appears in Settings within the my-tau web UI. Flip it on to start requiring authentication, off to open it back up. The setting persists across restarts.

Both HTTP and WebSocket connections are gated when enabled. The `/api/health` endpoint remains open for monitoring.

### Start / Stop

Control my-tau at runtime without uninstalling:

```
/my-tau-stop     Stop the server
/my-tau-start    Start it again
```

To prevent my-tau from auto-starting (e.g. in multi-session or dev container workflows):

```bash
MY_TAU_DISABLED=1 pi
```

You can still start it manually with `/my-tau-start` in that session.

## How it works

my-tau is a [Pi extension](https://github.com/badlogic/pi-mono#extensions) that starts an HTTP + WebSocket server inside the Pi process. The extension subscribes to all Pi events and forwards them to connected browser clients. Commands from the browser are executed via the extension API against the same agent session.

```
┌─────────────┐     ┌──────────────────────────────┐     ┌─────────────┐
│  Pi TUI     │     │  Pi Process                  │     │  Browser    │
│  (terminal) │◄───►│                              │◄───►│  (my-tau)   │
│             │     │  my-tau extension            │     │             │
└─────────────┘     │    ↳ HTTP + WS on :3001      │     └─────────────┘
                    └──────────────────────────────┘
```

There's no separate server to run. The extension auto-loads when Pi starts and shuts down when Pi exits.

## Development

Clone and point the extension at the local static files:

```bash
git clone https://github.com/343max/tau.git
cd tau
MY_TAU_STATIC_DIR=$(pwd)/public pi
```

Edit the files in `public/` — refresh the browser to see changes.

## License

MIT
