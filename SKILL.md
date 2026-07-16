---
name: obsidian-hermes-plugin
description: |
  Install and configure the Obsidian community plugin "Hermes Agent" (by jsun2020) that connects Obsidian to a locally-running Hermes Agent gateway via the API Server endpoint. Load this skill when the user wants to chat with a local Hermes Agent from inside Obsidian, get the plugin to recognize a Hermes gateway, configure gateway URL + API key, or troubleshoot the "Cannot reach the gateway" / "Auth failed (401/403)" errors that show up in the plugin settings.
version: 1.0.0
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: obsidian, plugin, brfat, gateway, api-server, productivity
    category: productivity
    related_skills: [obsidian, hermes-agent]
---

# Obsidian "Hermes Agent" Plugin — Install & Configure

Connect the Obsidian community plugin **Hermes Agent** (id `hermes-agent`,
v0.9.1+, by [jsun2020](https://github.com/jsun2020)) to a locally-running
Hermes Agent gateway via the **OpenAI-compatible API Server** the gateway
exposes on port `8642`.

This skill covers the **canonical install path** (BRAT), the **gateway-side
configuration** (the `API_SERVER_*` env vars that turn the API server on),
and the **troubleshooting** the plugin's own README does not handle (Codex
sandbox, Windows corporate lockdown, profile-specific ports).

It does **not** cover Obsidian-vault file I/O — that's `note-taking/obsidian/`.

## Why this skill exists

The Obsidian plugin and the Hermes gateway are two separate repos
(`jsun2020/hermes-agent-obsidian-plugin` and `NousResearch/hermes-agent`).
The plugin assumes a gateway that exposes an OpenAI-compatible HTTP API
on `http://127.0.0.1:8642` with `Authorization: Bearer <API_SERVER_KEY>`
auth. Hermes Agent **does expose this**, but only when explicitly enabled
(`API_SERVER_ENABLED=true`); out of the box, the gateway only runs the
messaging adapters (Feishu/Telegram/etc.) — there is **no HTTP listener on
8642** until you turn the API server on.

This skill captures the **proven end-to-end** bring-up: enable the API
server, set the key, restart the gateway, install the plugin, verify the
`Test connection` button reports `Transport: runs` and `1 model(s) available`.

## Architecture

```
┌─────────────────┐                    ┌──────────────────────┐
│ Obsidian vault  │   /v1/runs        │  Hermes Agent        │
│ + Hermes Agent  │   /v1/models      │  gateway (systemd)   │
│   plugin        │   127.0.0.1:8642  │                      │
│                 │  Bearer <KEY>     │  + API Server        │
│   (sidebar)     │ ────────────────▶ │    (when enabled)    │
└─────────────────┘                    └──────────────────────┘
```

The plugin's "Runs" transport (`POST /v1/runs` + `GET /v1/runs/{id}/events`)
is preferred when `/v1/capabilities` advertises it; the OpenAI-compatible
`/v1/chat/completions` is the fallback. The gateway supports both.

## Step 1 — Enable the Hermes API Server

On the **machine that runs the gateway** (the one that has `hermes` on
`PATH` and the systemd unit `hermes-gateway.service` running), do:

```bash
# 1a. Turn the API server on (writes to ~/.hermes/config.yaml as API_SERVER_ENABLED: true)
hermes config set API_SERVER_ENABLED true

# 1b. Set a bearer key. This is written to ~/.hermes/.env as API_SERVER_KEY=<value>
#     (NOT config.yaml — secrets stay in .env).
hermes config set API_SERVER_KEY "zbook-obsidian-key-2026"
```

**Verify the writes landed in the right files**:

```bash
grep -E "API_SERVER_ENABLED|API_SERVER_KEY" ~/.hermes/config.yaml
# Expect: API_SERVER_ENABLED: true
# Expect: NO API_SERVER_KEY line (it's in .env, not config.yaml)

grep API_SERVER_KEY ~/.hermes/.env
# Expect: API_SERVER_KEY=zbook-obsidian-key-2026
```

If `hermes config set API_SERVER_KEY` accidentally put the key in
`config.yaml` instead of `.env`, **remove it from `config.yaml`** (it's
not the convention; key in config makes it git-trackable and risks
leaking on commit). Put it only in `.env`.

```bash
# One-time cleanup if needed:
sed -i '/^API_SERVER_KEY:/d' ~/.hermes/config.yaml
echo 'API_SERVER_KEY=zbook-obsidian-key-2026' >> ~/.hermes/.env
```

**Pick a real key** — `openssl rand -hex 32` is a reasonable choice for
fresh installs:

```bash
KEY=$(openssl rand -hex 32)
hermes config set API_SERVER_KEY "$KEY"
echo "API server key: $KEY   (paste this into the Obsidian plugin)"
```

> **Save that key** — you'll paste it into the Obsidian plugin settings
> in Step 4.

## Step 2 — Restart the gateway

The gateway reads `API_SERVER_ENABLED` and `API_SERVER_KEY` at startup, so
you must restart it.

**Constraint: you cannot restart the gateway from inside the gateway's own
process.** If you ran the gateway interactively (`hermes gateway` in a
terminal), switch to a different terminal. If you installed it as a
systemd user service (`hermes gateway install`), killing the process
triggers the `Restart=always` policy.

**For systemd-managed gateway (the Zbook-on-WSL case)**:

```bash
# Find the gateway PID
pgrep -f "hermes_cli.main gateway"
# Example output: 4366

# Send SIGTERM. The systemd unit's Restart=always kicks in within ~5s.
kill 4366

# OR (cleaner, but requires the gateway to NOT be inside a tool
# call that would also die from the restart):
systemctl --user restart hermes-gateway
```

**Important**: do NOT call `hermes gateway restart` from the same Hermes
session that's running the gateway — it self-kills the command before it
can complete. Open a **separate** shell.

After ~5s, verify the new process is up:

```bash
pgrep -af "hermes_cli.main gateway"
# Expect: one PID, running
```

## Step 3 — Verify the API server is actually listening

The gateway restart alone is not enough — you must confirm port `8642`
is now accepting connections:

```bash
# Health check (no auth needed)
curl -s http://127.0.0.1:8642/health
# Expect: {"status":"ok", ...}

# Model list (auth required — uses the key you just set)
curl -s -H "Authorization: Bearer YOUR_API_SERVER_KEY" \
     http://127.0.0.1:8642/v1/models
# Expect: {"object":"list","data":[{"id":"hermes-agent", ...}, ...]}

# Gateway log line (proves the API server loaded)
journalctl --user -u hermes-gateway | tail -20
# Expect (somewhere in the recent log):
#   [API Server] API server listening on http://127.0.0.1:8642
```

If `curl /health` returns connection refused, the gateway didn't pick up
`API_SERVER_ENABLED=true` — check that Step 1a actually wrote the
config key (not just commented it out) and that you restarted the right
process (the systemd unit, not a stray `hermes gateway` foreground).

If `curl /v1/models` returns `401`, the key in `.env` doesn't match
what you pasted into the plugin — re-check Step 1b's `grep`.

## Step 4 — Install the Obsidian plugin

### 4a. Via BRAT (recommended)

1. **Open Obsidian** on the client machine.
2. **Settings → Community plugins → Browse** → search **"BRAT"** → install
   and **enable** it.
3. **Command palette** (Ctrl/Cmd+P) → type **"BRAT: Add a beta plugin for
   testing"** → run.
4. When prompted, paste the repository URL:
   `https://github.com/jsun2020/hermes-agent-obsidian-plugin`
5. BRAT downloads the latest release and installs **Hermes Agent** as
   `hermes-agent` in your vault's `.obsidian/plugins/` folder.
6. **Settings → Community plugins** → enable **Hermes Agent**.

### 4b. Manual install (if BRAT is blocked, e.g. corporate policies)

1. Download `main.js`, `manifest.json`, and `styles.css` from the
   [latest release](https://github.com/jsun2020/hermes-agent-obsidian-plugin/releases/latest).
2. Create a new folder: `<your-vault>/.obsidian/plugins/hermes-agent/`
3. Copy the three files into it.
4. **Settings → Community plugins** → enable **Hermes Agent**.

> **Do NOT** put the files into an existing plugin folder (e.g.
> `claudian/`) — Obsidian loads only the manifest + main.js that match
> the folder name `hermes-agent`.

## Step 5 — Configure the plugin

Open **Settings → Hermes Agent** and fill in:

| Setting | Value | Notes |
|---|---|---|
| **Gateway base URL** | `http://127.0.0.1:8642` | The default. If you run multiple named profiles (each on its own port 8643-8742), use that profile's port. |
| **API key** | paste the `API_SERVER_KEY` you set in Step 1b | Must match exactly. |
| **Model** | leave empty (uses gateway default `hermes-agent`) | Or set a specific id if you have multiple. |
| **Transport** | `Auto` | Prefers the richer Runs transport; falls back to OpenAI Chat Completions. |
| **Reasoning effort** | empty / medium | Adjust to taste. |
| **Include note content** | on | Sends the full note text with the message, not just the path. |
| **Auto-approve tools** | on | Lets the agent read/write files in your vault. **This grants real filesystem access** — turn off for read-only chat. |
| **Working folder** | click the folder chip in the chat footer | Native folder picker. Use your vault root, or a subfolder for scoping. |
| **Hermes home** | auto-detect (`$HERMES_HOME`, then `~/.hermes`) | Set explicitly if auto-detect fails (e.g. portable build). |

Click **Test connection**. The expected response:

```
Connected. Transport: runs. 1 model(s) available.
```

- **"Connected"** = HTTP reach + auth OK
- **"Transport: runs"** = the gateway advertised `/v1/capabilities` and the
  Runs endpoint is reachable
- **"1 model(s) available"** = `/v1/models` returned the `hermes-agent` label

If you see `Transport: chat` instead, the gateway didn't advertise Runs —
the plugin falls back to OpenAI Chat Completions, which still works but
loses tool/reasoning event streaming.

## Step 6 — Verify with a real message

1. Open the Hermes Agent sidebar (ribbon **bot** icon, or command
   palette → "Hermes Agent: Open chat view").
2. Type `hello` and press **Enter** (Shift+Enter for newline).
3. Expect a streamed reply under "Hermes" within a few seconds.

Optional sanity checks:

- **"Send selection to Hermes"** command — select text in a note, run the
  command, the reply should reference the selection.
- **"Stop"** (square) button — cancels an in-flight stream.
- **"Send current note to Hermes"** — attaches the whole active note as
  context.
- **"Analyze vault for smart graph"** — uses the gateway to build a
  semantic relationship graph (heavy call; needs a large context window).

## Troubleshooting

### "Cannot reach the gateway"

- The gateway is not running, or the URL is wrong.
- `curl http://127.0.0.1:8642/health` from the same machine should
  return `{"status":"ok"}`. If it doesn't, the gateway isn't listening —
  re-check Step 1/2.
- If the gateway runs on a different machine, use that machine's IP,
  not `127.0.0.1`. The plugin has no concept of "remote gateway".

### "Auth failed (401/403)"

- The bearer key in plugin settings doesn't match `API_SERVER_KEY` in
  `~/.hermes/.env`.
- Re-paste carefully — no extra whitespace, no line breaks.
- Verify with: `curl -H "Authorization: Bearer $KEY" http://127.0.0.1:8642/v1/models`
  should return the model list, not 401.

### "1 model(s) available" but the model id is wrong

- The gateway advertises a meta-label (e.g. `hermes-agent`); the
  **real** underlying model is read from `~/.hermes/config.yaml`
  (`model.default`). The plugin tries to read that to display the real
  name in the chat footer.
- If the footer shows `hermes-agent` instead of the real id (e.g.
  `gpt-5.5` or your actual model), set **Hermes home** in plugin
  settings to the folder containing `config.yaml`.

### Codex sandbox blocks file reads ("two read-only permission requests were rejected")

The agent runs inside a Codex sandbox whose workspace root is the
**gateway's launch directory** (default `~/.hermes`), not your Obsidian
vault. Reads of vault paths fall **outside** the sandbox and get
auto-denied because the run is non-interactive.

**Fix**: add to `~/.codex/config.toml` (the file the gateway uses, not
Obsidian's):

```toml
approval_policy = "never"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
# writes allowed only under these roots (reads are allowed anywhere):
writable_roots = ['/home/you/Obsidian-vault']
network_access = true
```

Then **restart the gateway** (the gateway loads the config at startup,
not per-request).

On **locked-down Windows** where `sandbox_mode = "workspace-write"`
fails with `CreateProcessWithLogonW failed: 1385` (the Codex sandbox
itself can't start), the only working mode is:

```toml
sandbox_mode = "danger-full-access"
```

(No sandbox, no `CreateProcessWithLogonW`.)

### Empty/!200 from `/v1/models`

Older gateways may not expose `/v1/models`. The plugin will still work
for chat. **Workaround**: set the model id manually in plugin settings.

### Wrong port

Named profiles bind 8643-8742. Check `~/.hermes/config.yaml` under
`platforms.api_server.extra.port` and update the plugin's base URL.

### "Cannot write to a read-only property" or similar JavaScript errors

The plugin's TS sources expect `Object.freeze`-style immutability. If
you see this, the plugin version is too old or too new. Pin to v0.9.1.

## Port reference

| Service | Port | Protocol | Notes |
|---|---|---|---|
| Hermes gateway messaging adapters | (varies) | WS | Feishu/Telegram/etc. — irrelevant to this skill |
| **Hermes API Server (this skill)** | **8642** | HTTP | OpenAI-compatible `/v1/chat/completions` + (if supported) `/v1/runs` + `/v1/capabilities` + `/v1/models` + `/health` |
| Named profile 1 | 8643 | HTTP | Optional, for multi-profile |
| Named profile N | 8642 + N - 1 | HTTP | 8742 max by convention |

## Security notes

- The `API_SERVER_KEY` is the **only** thing standing between the
  internet and your local agent. Don't bind the API server to
  `0.0.0.0` unless you have a firewall. Keep `host: "127.0.0.1"`.
- With `auto-approve-tools: on`, the agent can read/write any file
  in the working folder. **This includes your `.env` files, your SSH
  keys, your browser cookies, etc.** if the working folder is your
  vault root. Scope the working folder to a subfolder if you want
  to limit blast radius.
- The plugin's chat history (`history.json`) is stored under
  `.obsidian/plugins/hermes-agent/`, **separate** from
  `data.json` so your API key isn't accidentally committed to a
  synced vault.

## Manual test checklist (copy-pasteable)

1. Gateway side: `curl http://127.0.0.1:8642/health` returns
   `{"status":"ok"}`.
2. Gateway side: `curl -H "Authorization: Bearer $KEY" http://127.0.0.1:8642/v1/models`
   returns the model list.
3. Obsidian → Hermes Agent settings → **Test connection** → expect
   "Connected. Transport: runs. 1 model(s) available."
4. Open chat, type "hello" → expect a streamed reply.
5. Stop a long reply mid-stream → expect streaming to halt.
6. With gateway stopped, send a message → expect a clear "Cannot reach
   the gateway" error.

## Verified on

- Zbook (Windows 11 + WSL2 Ubuntu 24.04)
- Hermes Agent v0.18.2 (commit `f556edc1`)
- Hermes Agent Obsidian Plugin v0.9.1
- Obsidian 1.7+ (Windows)
- Tested 2026-07-16
