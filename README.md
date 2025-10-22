# Persistent Terminal — MCP Server

**Persistent Terminal MCP Server** is a TypeScript-based server that provides **persistent, resumable terminal sessions** using `node-pty`. It’s designed for integration with Model Context Protocol (MCP) clients (e.g., Claude Desktop / Claude Code, Cursor, Cline) and is optimized for long-running commands and workflows where the client may disconnect and later resume the session.

- Built with TypeScript + `node-pty`
- Keeps shell sessions alive across client disconnects
- Web management UI (xterm.js) + REST endpoints
- Intelligent output handling (circular buffer, spinner compression, head/tail modes)
- Integrations for automated fixes and tooling

YouTube Demo: https://youtu.be/nfLi1IZxhJs  
Windows MCP Setup Guide: https://youtu.be/WYEKwTQCAnc

---

## Table of Contents

1. [Why This Project](#why-this-project)  
2. [Highlights & Features](#highlights--features)  
3. [Quick Start](#quick-start)  
4. [Local Development](#local-development)  
5. [Usage & Examples](#usage--examples)  
6. [MCP Protocol — Design & Endpoints](#mcp-protocol---design--endpoints)  
7. [REST API](#rest-api)  
8. [Web UI](#web-ui)  
9. [Configuration & Environment Variables](#configuration--environment-variables)  
10. [Security & Best Practices](#security--best-practices)  
11. [Deployment](#deployment)  
12. [Contributing](#contributing)  
13. [License & Acknowledgements](#license--acknowledgements)  

---

## Why This Project

When AI assistants or remote tools run shell commands, two problems commonly appear:

1. **Ephemeral sessions** — when the client disconnects, long-running commands terminate.  
2. **No easy resume** — re-establishing context/output reliably is hard.

This server solves both by providing **persistent PTY-backed sessions** with robust output buffering, incremental reads, and a small web UI for inspection and control. It's particularly useful for AI agents that must perform multi-step, long-running tasks while preserving context and logs.

---

## Highlights & Features

### Session Management
- Create, reuse, and kill named terminal sessions.
- Sessions continue running after client disconnect.
- Sessions idle-cleaned to prevent resource leaks.

### Output Management
- **Circular buffer** with configurable capacity (default: 10,000 lines).
- Read modes:
  - `full` — return complete buffer
  - `head` — first N lines
  - `tail` — last N lines
  - `head-tail` — N lines from the start and end
- **Incremental reads** using `since` token/offset — only new output is returned.
- **Token estimation** to help downstream AI context budgeting.

### Spinner & Animation Compression
- Detects and compresses noisy spinner animations (e.g., CLI spinners).
- Reduces log spam while preserving useful output.
- Configurable detection thresholds and behavior.

### Web Management UI
- Live terminal viewer (xterm.js) with ANSI color support.
- Real-time streaming via WebSocket.
- Browser-based command run, session list, and session control.

### Integration & Extensibility
- Built to speak the MCP protocol (for direct AI integrations).
- Optional REST endpoints for non-MCP clients.
- Hooks for automated fixes (e.g., Codex-based fix flows) and custom tooling.

### Reliability
- `wait_for_output` and stabilization logic ensure full logs are retrieved from interactive programs.
- Handles ANSI control sequences correctly.
- Automatic reconnect/recovery behavior.

---

## Quick Start

### Run Immediately (no install)
```bash
npx persistent-terminal-mcp
````

Or the REST-only variant:

```bash
npx persistent-terminal-mcp-rest
```

### Install as a Dependency

```bash
npm install persistent-terminal-mcp
```

Then import and run in TypeScript:

```ts
import { PersistentTerminalMcpServer } from 'persistent-terminal-mcp';

const server = new PersistentTerminalMcpServer({ /* options */ });
await server.start();
```

### Global Install

```bash
npm install --global persistent-terminal-mcp
# then
persistent-terminal-mcp
```

---

## Local Development

Clone the repo and run locally for development:

```bash
git clone <repo-url>
cd persistent-terminal-mcp
npm install
npm run build       # compile TypeScript → dist/
npm start           # run compiled server
```

Run from source (recommended during development):

```bash
npm run dev         # run the server against TS source with tsx
npm run dev:rest    # run the REST-only variant in dev mode
```

Examples and test scripts:

```bash
npm run example:basic    # demo: create → write → read → kill
npm run example:smart    # demo: head/tail/incremental reads
npm run example:spinner  # demo: spinner compression
npm run example:webui    # launch web UI demo
npm run test:tools       # run tool validation tests
npm run test:fixes       # run fix/regression tests
```

Enable debug logging (writes to stderr — safe for MCP stdio):

```bash
MCP_DEBUG=true npm start
```

---

## Usage & Examples

### Basic MCP Flow (conceptual)

1. Client opens an MCP session to server.
2. Client requests creation of a named terminal (or attaches to existing).
3. Client writes a command to the terminal.
4. Server runs the command inside a node-pty shell and buffers output.
5. Client may disconnect; the process keeps running.
6. Later, client reconnects and requests output since the previous offset (incremental read) or full read.

### Common Operations

* Create terminal: returns session ID
* Write: send input to the PTY (stdin)
* Read: fetch output with `mode` and `since` options
* List: get active sessions and metadata
* Kill: terminate session and child processes
* Inspect: get process tree, working dir, and environment snapshot

(See the [API section](#mcp-protocol---design--endpoints) for detailed wire format examples.)

---

## MCP Protocol — Design & Endpoints

> This project implements a custom MCP surface for managing persistent PTY sessions. The following summarizes the key messages and semantics.

### Session Model

* Each session has:

  * `sessionId` (string)
  * `name` (optional friendly name)
  * `createdAt` timestamp
  * `lastActiveAt` timestamp
  * `status` (`running`, `exited`, `errored`)
  * `buffer` — circular buffer of stdout/stderr lines + offsets

### Main Actions (high level)

* `CreateTerminal` — create a new PTY session (options: shell, cwd, env, cols/rows)
* `AttachTerminal` — attach to an existing PTY by `sessionId`
* `Write` — send input to PTY
* `Read` — read buffered output (supports `mode`, `since`, `lines`)
* `ListSessions` — enumerate active sessions
* `KillSession` — terminate session and children
* `GetMeta` — retrieve session metadata and process info

### Read Modes & Parameters

* `mode`: `full` | `head` | `tail` | `head-tail`
* `lines`: integer number of lines for `head`/`tail`
* `since`: opaque offset or numeric token indicating only output after this marker is desired
* `stabilize`: boolean/timeout to wait for output to stabilize (useful for interactive commands)

### Example: Read Tail

Client request:

```json
{
  "action": "Read",
  "sessionId": "abc123",
  "options": {
    "mode": "tail",
    "lines": 200,
    "since": "offset-456"
  }
}
```

Server response contains:

* `lines`: array of strings (output)
* `nextOffset`: token to use for subsequent incremental reads
* `truncated`: boolean — true if output was truncated due to buffer size

---

## REST API

A lightweight REST wrapper is provided for non-MCP integrations. The REST API exposes the same core features: create, write, read, list, kill.

### Example Endpoints

* `POST /sessions` — create session
* `GET /sessions` — list sessions
* `GET /sessions/:id` — get meta
* `POST /sessions/:id/write` — write to session
* `GET /sessions/:id/read` — read session output (query params for `mode`, `lines`, `since`)
* `DELETE /sessions/:id` — kill session

### Example: Read via REST

```
GET /sessions/abc123/read?mode=tail&lines=200&since=offset-456
```

Response:

```json
{
  "lines": ["line1...", "line2..."],
  "nextOffset": "offset-789",
  "truncated": false
}
```

> Note: When using the REST API in environments where request/response sizes are limited, use `tail`/`head`/`since` to avoid transferring huge logs.

---

## Web UI

A small management UI is included to inspect and interact with sessions.

Features:

* Live terminal view with xterm.js (ANSI color support)
* Session list and metadata
* Buttons to create/kill sessions and run commands
* WebSocket-backed live stream for low-latency updates

Launch UI (development):

```bash
npm run example:webui
# or if installed globally
persistent-terminal-mcp --web
```

The UI is intentionally minimal and developer-focused — it exists primarily to aid debugging and manual intervention.

---

## Configuration & Environment Variables

Configure behavior via environment variables or programmatic options when instantiating the server.

Common environment variables:

* `MCP_PORT` — default port for REST/WebSocket (if running in network mode)
* `MCP_DEBUG` — enable debug logs (writes to stderr)
* `SESSION_IDLE_TIMEOUT` — milliseconds before idle session cleanup
* `BUFFER_LINES` — number of lines to keep in the circular buffer (default 10000)
* `SPINNER_COMPRESSION` — `true|false` to enable spinner compression
* `MAX_SPAWN_RETRIES` — how many times to retry spawning a PTY before failing

Example `.env`:

```
MCP_PORT=3456
SESSION_IDLE_TIMEOUT=3600000
BUFFER_LINES=10000
SPINNER_COMPRESSION=true
```

Programmatic options (TypeScript):

```ts
const server = new PersistentTerminalMcpServer({
  bufferLines: 10000,
  idleTimeoutMs: 60 * 60 * 1000, // 1 hour
  spinnerCompression: true,
  webPort: 3456
});
```

---

## Security & Best Practices

* **Privilege separation:** Run the server as a non-root user. Avoid exposing the server on public networks without a reverse proxy and authentication.
* **Authentication:** If exposing REST or Web UI, add an authentication layer (HTTP basic auth, token-based, or integrate behind an authenticated gateway).
* **Network exposure:** Bind to `localhost` by default. If you must listen on a public interface, enable TLS and strong authentication.
* **Input validation:** Ensure clients are trusted — arbitrary command execution is sensitive. Use policy-based allowlists if necessary.
* **Resource limits:** Monitor and limit concurrent session counts, PTY memory usage, and per-session CPU/time if running untrusted workloads.

---

## Deployment

Typical deployment patterns:

* **Local dev:** `npm run dev`
* **Single-server:** run as a systemd service with a process supervisor. Use environment variables for configuration and optionally a reverse proxy for TLS.
* **Containerized:** build a small Docker image. Ensure you configure user, volumes for logs if needed, and resource limits.

Example `systemd` unit (sketch):

```ini
[Unit]
Description=Persistent Terminal MCP

[Service]
User=someuser
WorkingDirectory=/srv/persistent-terminal-mcp
ExecStart=/usr/bin/node dist/index.js
Restart=on-failure
Environment=MCP_PORT=3456
```

---

## Diagnostics & Debugging

* Enable `MCP_DEBUG=true` to log debug information to stderr.
* Use the Web UI to inspect active PTY processes and recent output.
* For crash analysis, capture the server stderr and the PTY process logs.
* The server exposes simple health endpoints in REST mode for readiness/liveness checks.

---

## Automated Fixes & Tooling (Optional)

This project contains optional automation hooks (e.g., integrations that attempt to run automated code fixes using model-driven tooling). These features are **opt-in** and typically operate like this:

1. Agent gathers context (project files, failing command logs).
2. Agent proposes a fix via a codex/LLM-powered workflow.
3. Fix is applied optionally and a diff/report is saved to `docs/fixes/`.

> CAUTION: These automated flows may modify files. Use them in controlled environments only and ensure backups or VCS protections are in place.

---

## Contributing

Contributions are welcome.

Suggested workflow:

1. Fork the repository.
2. Create a feature branch: `git checkout -b feat/my-feature`.
3. Run tests and linters locally.
4. Open a PR describing the change and rationale.

Please include tests for new features where appropriate, and keep changes small and focused.

---

## Example Troubleshooting Scenarios

* **“My command output is missing or truncated”**

  * Check buffer size (`BUFFER_LINES`). If command produces > buffer, use incremental reads (`since`) or `tail` to fetch the end of output.
  * Ensure spinner compression isn’t removing relevant lines — adjust spinner thresholds.
* **“Sessions are disappearing”**

  * Inspect `SESSION_IDLE_TIMEOUT` and confirm session wasn't auto-evicted.
* **“Web UI fails to display colors correctly”**

  * Confirm xterm.js is connected via WebSocket and the terminal app sends ANSI sequences (some programs detect non-ttys and change output).

---

## License

This project is provided under the license included in the repository. Please consult the `LICENSE` file for full details.

---

## Acknowledgements

* Inspired by needs of AI agent tooling and long-lived developer sessions.
* Uses open-source libraries such as `node-pty`, `xterm.js`, and TypeScript.

---

## Contact / Further Reading

If you want help deploying, integrating with an AI assistant, or customizing the server behavior (buffering strategy, spinner handling, or MCP wiring), open an issue or discussion on the repository.

---

### Changelog (high level)

* Initial release: persistent PTY sessions, MCP + REST interfaces, web UI, spinner compression, incremental reads.
* Future items: per-session authentication hooks, metrics/telemetry, advanced quotas and sandboxing.

https://chatgpt.com/share/68f852ca-f0ac-8013-9c12-f93355c24d60
