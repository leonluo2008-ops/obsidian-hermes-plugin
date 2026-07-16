# obsidian-hermes-plugin

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) **skill** that installs and configures
the [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin)
for Obsidian, bridging Obsidian to a locally-running Hermes gateway via the gateway's
OpenAI-compatible API Server.

**Hermes Agent skill**: [`productivity/obsidian-hermes-plugin/SKILL.md`](./SKILL.md)
(15.3 KB · 380 lines)

## What it does

This skill is a **single-file, drop-in procedure** for Hermes Agent that captures
the proven end-to-end bring-up flow:

1. **Enable the gateway's API Server** (`API_SERVER_ENABLED=true`, `API_SERVER_KEY=...`)
2. **Restart the gateway** (systemd-aware, with the SIGTERM propagation guard)
3. **Verify port 8642** is actually listening (`/health`, `/v1/models`, journalctl)
4. **Install the Obsidian plugin** (via [BRAT](https://github.com/TfTHacker/obsidian42-brat) — recommended)
5. **Configure the plugin** (baseUrl, API key, transport, working folder)
6. **Verify with a real chat message**

Plus a dedicated **Troubleshooting** section covering the 6 problems the plugin's
own README does not handle:

- `Cannot reach the gateway`
- `Auth failed (401/403)`
- Codex sandbox blocking vault file reads (`workspace-write` config + Windows
  `CreateProcessWithLogonW 1385` corporate-lockdown fallback)
- Wrong port for named profiles
- Empty `/v1/models` on older gateways
- Empty real-model-id in the chat footer (Hermes home path)

## Install

Drop the SKILL.md into your Hermes Agent skills directory:

```bash
# Linux / macOS / WSL
mkdir -p ~/.hermes/skills/productivity/obsidian-hermes-plugin
curl -fsSL https://raw.githubusercontent.com/leonluo2008-ops/obsidian-hermes-plugin/main/SKILL.md \
  -o ~/.hermes/skills/productivity/obsidian-hermes-plugin/SKILL.md

# Or via Hermes CLI (if you prefer the registry path)
hermes skills install https://github.com/leonluo2008-ops/obsidian-hermes-plugin
```

Then **restart your Hermes session** (or run `/reload-skills` in-session) so the
new skill loads.

## Use

Once loaded, trigger phrases like:

- *"Install the Obsidian Hermes plugin"*
- *"Connect my Obsidian to Hermes"*
- *"Obsidian plugin 401 unauthorized"*
- *"Fix the Cannot reach the gateway error in Obsidian"*

…will surface this skill and run through the bring-up checklist.

## What this skill is NOT

- **Not** a Hermes plugin itself — it's a `SKILL.md` (procedural knowledge) for
  the Hermes agent.
- **Not** the Obsidian plugin — that's
  [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin).
- **Not** a fork of Hermes Agent — it only depends on the gateway's
  `API_SERVER_*` env vars, which Hermes ships in v0.18.2+.

## Architecture

```
┌─────────────────┐                    ┌──────────────────────┐
│ Obsidian vault  │                    │  Hermes Agent        │
│ + Hermes Agent  │   /v1/runs        │  gateway (systemd)   │
│   plugin        │   /v1/models      │                      │
│                 │   127.0.0.1:8642  │  + API Server        │
│   (sidebar)     │  Bearer <KEY>     │    (when enabled)    │
└─────────────────┘ ──────────────────▶└──────────────────────┘
```

The plugin's "Runs" transport (`POST /v1/runs` + `GET /v1/runs/{id}/events`) is
preferred when `/v1/capabilities` advertises it; OpenAI-compatible
`/v1/chat/completions` is the fallback. Hermes Agent gateway supports both.

## Verified on

| Component | Version | Date |
|---|---|---|
| Zbook host | Windows 11 + WSL2 Ubuntu 24.04 | 2026-07-16 |
| Hermes Agent | v0.18.2 (commit `f556edc1`) | 2026-07-16 |
| Obsidian | 1.7+ | 2026-07-16 |
| Obsidian plugin (target) | v0.9.1 | 2026-07-16 |
| Obsidian plugin repo | [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin) | latest at install time |

## License

MIT — see [LICENSE](./LICENSE).

## Related

- [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin) —
  the Obsidian plugin this skill installs
- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) —
  the agent whose gateway this skill connects to
- [obsidian42-brat](https://github.com/TfTHacker/obsidian42-brat) —
  recommended way to install the plugin from a GitHub URL