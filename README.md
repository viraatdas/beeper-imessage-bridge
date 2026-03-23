# iMessage Bridge — MBA Server

Always-on MacBook Air (2012) server that syncs iMessage via macOS `chat.db` for programmatic access.

## Architecture

```
┌─────────────────────────────────────────────────┐
│  MacBook Air Server (always-on, Tailscale SSH)  │
│                                                 │
│  macOS Messages app                             │
│       │                                         │
│       ▼                                         │
│  ~/Library/Messages/chat.db  (read-only)        │
│       │                                         │
│       ▼                                         │
│  iMessage Sync Daemon (launchd)                 │
│       │  reads chat.db every 120s               │
│       ▼                                         │
│  SQLite (data/jarvis_memory.db)                 │
│       │                                         │
│       ▼                                         │
│  [TODO] API / network bridge to consumers       │
└─────────────────────────────────────────────────┘
```

## What's Running

### iMessage Read Sync (live)

A launchd daemon continuously reads the macOS Messages database and syncs messages into a local SQLite store.

- **Daemon**: `com.exla.jarvis.imessage-sync`
- **Plist**: `~/Library/LaunchAgents/com.exla.jarvis.imessage-sync.plist`
- **Logs**: `~/Library/Logs/jarvis/imessage-sync.{out,err}.log`
- **State**: `~/Library/Application Support/Jarvis/imessage_sync_state.json`
- **SQLite**: `~/Documents/exla/jarvis/data/jarvis_memory.db`
- **Interval**: 120 seconds
- **Overlap**: Re-reads last 2000 rows each pass to capture read-state updates

### iMessage Send (TODO)

Sending iMessages programmatically requires one of:
- **AppleScript** (`osascript -e 'tell application "Messages" to send ...'`)
- **Shortcuts/Automator** workflows

## MBA Server Hardware

**MacBook Air (Mid-2012)** — Model `MacBookAir5,2`

| Spec | Value |
|---|---|
| Model | MacBook Air Mid-2012 (MacBookAir5,2) |
| CPU | Intel Core i5-3427U @ 1.80 GHz (2 cores, HT) |
| RAM | 8 GB |
| Storage | 223 GB SSD (~170 GB free) |
| macOS | 15.7.4 (Sequoia) |
| Hostname | `Viraats-MacBook-Air.local` |
| User | `viraat` |
| Python | 3.11 (via uv) |
| uv | 0.10.12 |
| Access | Tailscale SSH (always-on) |
| Git identity | `viraat.laldas@gmail.com` / `Viraat Das (MBA Server)` |

This is a 2012 MacBook Air repurposed as an always-on headless server. It's low-power and stays connected via Tailscale for remote SSH access. It's the only machine with an Apple ID signed into Messages and access to `chat.db`.

## Setup

### 1. Install dependencies

```bash
uv sync --extra dev --python 3.11
```

### 2. Configure .env

```bash
cp .env.example .env
# Key setting: JARVIS_MEMORY_SQLITE_PATH=data/jarvis_memory.db
```

### 3. Test one-shot sync

```bash
uv run --python 3.11 python scripts/imessage_sync_daemon.py --env-file .env --once
```

### 4. Install launchd daemon

```bash
bash scripts/install_imessage_sync_launchd.sh
# Or manually create plist — see docs/launchd-plist.xml
```

### 5. Verify

```bash
launchctl list | grep jarvis
tail -f ~/Library/Logs/jarvis/imessage-sync.err.log
```

## Full Disk Access

macOS protects `~/Library/Messages/chat.db` via TCC. You must grant **Full Disk Access** to the process that runs the daemon:

1. System Settings → Privacy & Security → Full Disk Access
2. Add `/Users/viraat/.local/bin/uv` (or the Python binary used by uv)
3. If running via Terminal/SSH, also add the terminal app

## Next Steps

- [ ] Implement iMessage sending via AppleScript
- [ ] Expose synced messages via API for network consumers
- [ ] Add health monitoring / heartbeat for the sync daemon
