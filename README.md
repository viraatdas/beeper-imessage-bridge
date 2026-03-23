# Beeper iMessage Bridge — MBA Server Setup

This repository documents the MacBook Air server configuration that bridges iMessage

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
│  Jarvis SQLite (data/jarvis_memory.db)          │
│       │                                         │
│       ▼                                         │
│  [TODO] Bridge to Jetson / main Jarvis          │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  Jetson Orin AGX (main Jarvis host)             │
│                                                 │
│  Jarvis Runtime                                 │
│    - Voice pipeline (wake word → STT → LLM)     │
│    - Tool calling (lights, blinds, etc.)        │
│    - Unified message inbox                      │
│    - Vision / identity / object memory          │
└─────────────────────────────────────────────────┘
```

## What's Running

### iMessage Read Sync (live)

The MBA runs a launchd daemon that continuously reads the macOS Messages database and syncs messages into Jarvis's unified message memory (SQLite).

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
- **mautrix-imessage** bridge (what Beeper used)
- **Shortcuts/Automator** workflows

This is not yet implemented.

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
| Jarvis repo | `~/Documents/exla/jarvis` |
| Git identity | `viraat.laldas@gmail.com` / `Viraat Das (MBA Server)` |

This is a 2012 MacBook Air repurposed as an always-on headless server. It's low-power and stays connected via Tailscale for remote SSH access. Its primary role in the Jarvis system is as the iMessage bridge — it's the only machine with an Apple ID signed into Messages and access to `chat.db`.

## Setup Steps (reproduced)

### 1. Clone Jarvis

```bash
mkdir -p ~/Documents/exla
git clone https://github.com/exla-ai/jarvis.git ~/Documents/exla/jarvis
cd ~/Documents/exla/jarvis
git config user.email "viraat.laldas@gmail.com"
git config user.name "Viraat Das (MBA Server)"
```

### 2. Install dependencies

Voice pipeline deps (`pipecat-ai`, `numpy`, NVIDIA Riva) were moved to `[project.optional-dependencies] voice` in `pyproject.toml` since they don't build on x86 macOS (llvmlite fails). The MBA only needs the base deps.

```bash
uv sync --extra dev --python 3.11
```

### 3. Configure .env

```bash
cp .env.example .env
# Defaults are fine for iMessage sync — key setting:
# JARVIS_MEMORY_SQLITE_PATH=data/jarvis_memory.db
```

### 4. Test one-shot sync

```bash
uv run --python 3.11 python scripts/imessage_sync_daemon.py --env-file .env --once
```

### 5. Install launchd daemon

```bash
bash scripts/install_imessage_sync_launchd.sh
# Or manually create plist — see docs/launchd-plist.xml
```

### 6. Verify

```bash
launchctl list | grep jarvis
tail -f ~/Library/Logs/jarvis/imessage-sync.err.log
```

## Full Disk Access

macOS protects `~/Library/Messages/chat.db` via TCC. You must grant **Full Disk Access** to the process that runs the daemon:

1. System Settings → Privacy & Security → Full Disk Access
2. Add `/Users/viraat/.local/bin/uv` (or the Python binary used by uv)
3. If running via Terminal/SSH, also add the terminal app

## Jarvis pyproject.toml Change

The voice pipeline dependencies were moved from core `[project.dependencies]` to `[project.optional-dependencies] voice`:

```toml
[project.optional-dependencies]
voice = [
    "pipecat-ai[silero]>=0.0.50",
    "openwakeword>=0.6.0",
    "numpy>=1.26",
    "sounddevice>=0.5",
    "nvidia-riva-client>=2.17.0",
    "grpcio>=1.60",
    "protobuf>=4.25",
]
```

On the Jetson (production host), install with: `uv sync --extra voice --extra dev`
On the MBA (iMessage bridge only): `uv sync --extra dev`

## Next Steps

- [ ] Implement iMessage sending via AppleScript
- [ ] Bridge MBA SQLite to Jetson (network sync, API push, or shared storage via Tailscale)
- [ ] Evaluate mautrix-imessage for full bidirectional bridge
- [ ] Add health monitoring / heartbeat for the sync daemon
