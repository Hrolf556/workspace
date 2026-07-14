# KVM8 Google Workspace MCP: Remote Endpoint Build Plan

Status: PLAN ONLY. No changes have been made on KVM8. Every step that touches the
box is marked `[APPROVAL REQUIRED]`.

## What this delivers

A multi-client setup for the existing Google Workspace MCP server (Gemini CLI
extension, stdio) installed at `/home/george/Developer/Github/google-workspace-mcp`
on KVM8, authenticated as `rusty@rustycole.com` (Workspace domain identity; token
verified live and working on 2026-06-04).

Two tiers, because most of your clients do NOT need the hard part:

- Tier 1 (minutes, low risk): wire the local stdio command into the agents that
  run on KVM8 and the Mac Mini (OpenClaw, Codex, Claude Code, Hermes, Claude
  Desktop). No network exposure.
- Tier 2 (the real build): an OAuth-gated public HTTPS endpoint for the browser
  clients that cannot speak stdio (claude.ai web, ChatGPT, Cowork), mirroring the
  cs-voice-mcp pattern (stdio to HTTP bridge, your `mcp_oauth_proxy.py`, Tailscale
  Funnel on `kvm8.tailab446d.ts.net`).

Scope is locked to "writes minus destructive" everywhere via
`WORKSPACE_FEATURE_OVERRIDES` (see Step 1).

## Coordination with the parallel content-factory (sheets) build

This build runs alongside a separate content-factory build. They are complementary:
this MCP gives Cowork/clients Workspace read access plus Google Doc creation as
`rusty@rustycole.com`; a separate `mcp-google-sheets` service handles sheet writes.
Hard constraints to avoid collision:

- Keep Sheets READ-ONLY. Do NOT enable `sheets.write` via overrides. That group ships
  zero tools, so enabling it only widens the OAuth scope on the shared
  `rusty@rustycole.com` token for no usable capability. Sheet writes belong to the
  other build.
- Keep Docs write ON (`docs.create`, `docs.writeText`). The content factory creates
  output Docs through this MCP. The override string in Step 1 already leaves Docs
  write enabled.
- Ports/paths: this build owns bridge `8021`, oauth proxy `8023`, Funnel prefix `/gw`.
  The sheets connector owns `8025` (MCP), `8027` (oauth proxy), Funnel prefix
  `/sheets`. Do not use the sheets-owned ports or path.
- Both builds reuse `mcp_oauth_proxy.py` by copying (never modifying the cs-voice
  original). When Tier 2 is live, record the handoff artifacts (see the Handoff
  section) so the sheets connector mirrors the mapping under `/sheets`.

## BUILD STATUS (2026-06-04)

- Tier 1: DONE. `gw-mcp.env` + `gw-mcp-stdio.sh` created in the KVM8 repo; scope lock
  verified (52 tools, destructive absent, docs write present). Wired and live:
  OpenClaw (mcp.servers + gateway restarted), Codex (`[mcp_servers.google-workspace]`),
  Claude Code (`✓ Connected`), Hermes (Mac Mini, ssh-wrapped), Claude Desktop (Mac Mini,
  ssh-wrapped). REMAINING: Gemini `extensions link` (blocks on an interactive Y prompt —
  Rusty runs it himself).
- Tier 2 KVM8 downstream: BUILT + verified (2026-06-04), loopback-only, nothing public.
  `gw-mcp-bridge.service` = mcp-proxy on 127.0.0.1:8021 (stdio->streamable-http; chosen
  over supergateway because supergateway binds 0.0.0.0 with no host flag, mcp-proxy
  defaults to 127.0.0.1). `gw-mcp-bearer.service` = gw_bearer_proxy.py on 127.0.0.1:8023
  (plain `Authorization: Bearer GW_DOWNSTREAM_SECRET` check -> forwards to :8021/mcp).
  Both enabled, Linger=yes. Chain verified: no-auth/bad-bearer=401, good-bearer=200+serverInfo.
  Secret in ~/.cloudflare/gw-downstream.env (600) on KVM8 AND the Mac. Reusable proxy
  script: gw_bearer_proxy.py; run wrapper: gw-bearer-run.sh.
- Tier 2 worker + public exposure: JOINT step, still pending. Worker = the content-factory
  shared template (do NOT write a separate one). Funnel route (ONE route, since OAuth is in
  the worker): `tailscale funnel --bg --set-path /gw/mcp http://127.0.0.1:8023/mcp` — HELD.
  Blockers: Rusty `wrangler login` + the shared GitHub OAuth app callback `cf-google-workspace.../callback`.

## Architecture (corrected to match your real stack)

```
google-workspace stdio server  (node dist/index.js, single identity, token on KVM8)
        |
        |  Tier 1: launched directly by local agents over stdio
        |  Tier 2: launched once behind an HTTP bridge
        v
  supergateway  (stdio -> Streamable HTTP)   127.0.0.1:8021/mcp
        v
  mcp_oauth_proxy.py  (reused from cs-voice)  127.0.0.1:8023
        |   does the MCP OAuth dance, single-user auto-approve
        v
  Tailscale Funnel  https://kvm8.tailab446d.ts.net/gw/...   (public HTTPS, real TLS)
        |
        +--> claude.ai web, ChatGPT, Cowork   (native OAuth /mcp)
        +--> OpenClaw / Hermes / Codex / Claude Code  (optional: via mcp-remote, or just use Tier 1 stdio)

Optional outer layer: a Cloudflare worker (cloudflare-mcp.stevelang771.workers.dev
pattern) that proxies to the Funnel URL. Source lives off-KVM8 (the on-box repo only
has cloudflare/shim/). Add only if you want a workers.dev hostname in front.
```

Ports chosen to avoid collisions with cs-voice (which uses 8000-8003, 8010, 8088, 8090):
google-workspace bridge `8021`, oauth proxy `8023`. The parallel sheets connector
reserves `8025`/`8027` and Funnel `/sheets`; do not use those.

---

## Step 0 — Inspect the cs-voice originals before copying [read-only]

Reuse, do not reinvent. Read these on KVM8 so the gw- copies match exactly:

- `/home/george/services/cs-voice-mcp/bin/mcp_oauth_proxy.py` (the OAuth front door)
- `/home/george/services/cs-voice-mcp/oauth-proxy.env` (env keys; values are secret)
- `/home/george/.config/systemd/user/cs-voice-mcp-oauth-proxy.service`
- `/home/george/.config/systemd/user/cs-voice-mcp.service` and `-internal.service`
- current `tailscale funnel status` (note how `/mcp` -> :8001 and `/token` `/register` -> :8003 are mapped; you will mirror that mapping with a `gw/` prefix)

If you also want the outer Cloudflare worker layer, find `cloudflare/worker/src/index.ts`
on the machine it was deployed from (not on KVM8) and copy its structure.

---

## Step 1 — Lock scope and confirm the binary [APPROVAL REQUIRED]

Decided posture: keep writes, drop the destructive ones. The override string:

```
gmail.send:off,gmail.sendDraft:off,calendar.deleteEvent:off,drive.trashFile:off
```

Do NOT add `sheets.write:on` (owned by the parallel sheets build; zero tools here, only
widens token scope). Leave the docs group ON (the content factory creates Docs through
this MCP). These offs are tool-level (subtractive): they unregister the tools but do not
shrink the token's existing OAuth scopes, so no re-consent is triggered.

Put it where every launch inherits it. Create an env file (chmod 600, backed up if it exists):

```
/home/george/Developer/Github/google-workspace-mcp/gw-mcp.env
```

Contents:

```
WORKSPACE_FEATURE_OVERRIDES=gmail.send:off,gmail.sendDraft:off,calendar.deleteEvent:off,drive.trashFile:off
```

Sanity-check the tool list reflects the lock (destructive tools should be absent).
Run from the repo dir:

```
cd /home/george/Developer/Github/google-workspace-mcp && set -a && . ./gw-mcp.env && set +a && printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"c","version":"1"}}}' '{"jsonrpc":"2.0","method":"notifications/initialized"}' '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | timeout 20 node workspace-server/dist/index.js 2>/dev/null | tail -1
```

Confirm `gmail_send`, `gmail_sendDraft`, `calendar_deleteEvent`, `drive_trashFile`
do NOT appear. (Underscore names because `--use-dot-names` is omitted for non-Gemini clients.)

---

## Tier 1 — Local stdio clients (do this first)

This gets OpenClaw, Codex, Claude Code, Hermes, and Claude Desktop working with no
network exposure. Each client just launches the binary. Credentials resolve from the
repo dir automatically (token path is anchored to `gemini-extension.json`).

Optional convenience launcher (so the override is always applied), chmod +x:

```
/home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh
```

```
#!/usr/bin/env bash
set -euo pipefail
cd /home/george/Developer/Github/google-workspace-mcp
set -a; . ./gw-mcp.env; set +a
exec node workspace-server/dist/index.js
```

### OpenClaw (KVM8) [APPROVAL REQUIRED]

Back up `~/.openclaw/openclaw.json` first, edit with a JSON parser (never sed/echo).
Add under `mcp.servers`:

```json
"google-workspace": {
  "command": "/home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh",
  "args": [],
  "connectionTimeoutMs": 30000
}
```

Restart the gateway: `systemctl --user restart openclaw-gateway`

### Codex (KVM8 and/or Mac Mini) [APPROVAL REQUIRED]

Back up `~/.codex/config.toml`, add:

```toml
[mcp_servers.google-workspace]
command = "/home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh"
args = []
```

### Claude Code (KVM8) [APPROVAL REQUIRED]

```
claude mcp add google-workspace -s user -- /home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh
```

### Hermes (Mac Mini, not KVM8) [APPROVAL REQUIRED]

Hermes runs on the Mac Mini; the server is on KVM8. Use an SSH-wrapped stdio command.
Back up `~/.hermes/config.yaml`, add under `mcp_servers`:

```yaml
google-workspace:
  command: ssh
  args: ["kvm8", "/home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh"]
```

Verify with `hermes mcp list`.

### Claude Desktop (Mac Mini / Mac Air) [APPROVAL REQUIRED]

Same SSH-wrapped pattern in the Desktop MCP config:

```json
"google-workspace": {
  "command": "ssh",
  "args": ["kvm8", "/home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh"]
}
```

### Gemini CLI (KVM8) [APPROVAL REQUIRED]

The native path. The clone is already built and authenticated; link it:

```
gemini extensions link /home/george/Developer/Github/google-workspace-mcp
```

Gemini uses dot tool names by design; no override needed there.

---

## Tier 2 — OAuth public endpoint for browser clients (claude.ai web, ChatGPT, Cowork)

These cannot speak stdio. They need an OAuth-gated HTTPS MCP URL.

### CORRECTED DESIGN (supersedes the auto-approve proxy idea) — HELD pending shared spec

The Funnel `mcp_oauth_proxy.py` auto-approves `/authorize` with no per-user check: the
only thing stopping any claude.ai/ChatGPT user who learns the URL from connecting is URL
obscurity (fine for cs-voice's read-only corpus, NOT for write-capable admin Workspace
access). The real per-user gate is the GitHub-login Cloudflare worker
(`ALLOWED_LOGINS = ["Hrolf556"]`). So the front door MUST be a GitHub-gated worker.

Both existing workers (cs-voice `cloudflare/worker`, `cloudflare-mcp` repo) SELF-HOST
their tools and do not proxy to a downstream, so this needs a NEW custom worker that
reuses the GitHub OAuth machinery (`github-handler.ts`, `oauth-upstream.ts`,
`workers-oauth-utils.ts`) but replaces the McpAgent with a fetch-proxy to KVM8.

This is HELD: the content-factory build is authoring a shared worker-proxy spec that both
google-workspace and sheets will use (one GitHub OAuth app, callback per worker, one KV
approach, two routes/instances). Do not build the worker until that spec lands.

Secure chain once built:
`claude.ai/ChatGPT -> GitHub-gated worker (Hrolf556) -> Funnel /gw/mcp -> bearer proxy (:8023, secret only the worker holds) -> supergateway bridge (:8021) -> google-workspace stdio`.

### KVM8 interface contract (feed this to the shared worker spec)

- Downstream the worker proxies to: `https://kvm8.tailab446d.ts.net/gw/mcp` (Streamable HTTP).
- Funnel `/gw/mcp` -> bearer proxy `127.0.0.1:8023` requiring `Authorization: Bearer <GW_DOWNSTREAM_SECRET>` -> bridge `127.0.0.1:8021`.
- Bridge (BUILT): `mcp-proxy --host 127.0.0.1 --port 8021 --pass-environment -- /home/george/Developer/Github/google-workspace-mcp/gw-mcp-stdio.sh` (the launcher applies the gw-mcp.env scope lock). mcp-proxy chosen over supergateway: supergateway has no host flag and binds 0.0.0.0; mcp-proxy binds 127.0.0.1 natively. Serves streamable-http at /mcp.
- Ports owned: 8021 (bridge), 8023 (bearer proxy). Funnel prefix `/gw`. (Sheets owns 8025/8027 + `/sheets`.)
- Tool-name separator: underscore (no `--use-dot-names`): `gmail_search`, `docs_create`, etc.
- Identity: single shared `rusty@rustycole.com`; gate = worker `ALLOWED_LOGINS=["Hrolf556"]`.
- Worker needs: secrets `GITHUB_CLIENT_ID/SECRET`, `COOKIE_ENCRYPTION_KEY`, `GW_DOWNSTREAM_SECRET`, var `GW_DOWNSTREAM_URL`; an `OAUTH_KV` namespace; GitHub OAuth app callback `<worker-host>/callback`.
- Deploy blocked on: `wrangler login` (not authed on the Mac), the GitHub OAuth app callback, KV + secrets.

### Step 2a — stdio to HTTP bridge [APPROVAL REQUIRED] (build when spec lands)

Use supergateway (Node, matches your stack). Streamable HTTP output exposes `/mcp`.

Systemd user unit `~/.config/systemd/user/gw-mcp-bridge.service`:

```ini
[Unit]
Description=google-workspace MCP stdio->HTTP bridge (loopback)
After=network.target

[Service]
Type=simple
WorkingDirectory=%h/Developer/Github/google-workspace-mcp
EnvironmentFile=%h/Developer/Github/google-workspace-mcp/gw-mcp.env
ExecStart=/usr/bin/npx -y supergateway --stdio "node workspace-server/dist/index.js" --outputTransport streamableHttp --port 8021 --host 127.0.0.1
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=default.target
```

Downstream MCP URL becomes `http://127.0.0.1:8021/mcp`. Loopback-only, so only the
OAuth proxy can reach it (no bearer needed between them).

### Step 2b — OAuth front door (reuse mcp_oauth_proxy.py) [APPROVAL REQUIRED]

Copy the proven proxy and give it its own state + env. Do not modify the cs-voice original.

```
cp /home/george/services/cs-voice-mcp/bin/mcp_oauth_proxy.py /home/george/Developer/Github/google-workspace-mcp/gw_oauth_proxy.py
```

Env file `~/Developer/Github/google-workspace-mcp/gw-oauth-proxy.env` (chmod 600):

```
OAUTH_PROXY_PORT=8023
OAUTH_PROXY_BIND=127.0.0.1
OAUTH_PROXY_PUBLIC_BASE_URL=https://kvm8.tailab446d.ts.net/gw
OAUTH_PROXY_DOWNSTREAM_URL=http://127.0.0.1:8021/mcp
OAUTH_PROXY_DOWNSTREAM_TOKEN=
OAUTH_PROXY_STATE_DIR=%h/.local/share/gw-mcp-oauth
OAUTH_PROXY_TOKEN_TTL_SECONDS=604800
```

Note: match `OAUTH_PROXY_PUBLIC_BASE_URL` and the Funnel path mapping to whatever the
proxy expects for its `.well-known`, `/authorize`, `/token`, `/register`, `/mcp`
sub-paths. cs-voice splits `/mcp` and `/token` `/register` across two ports/paths;
replicate that exact split with a `gw/` prefix. Confirm by reading the proxy's route
table (Step 0). It may need a Python venv with `aiohttp` (reuse the cs-voice venv or
make a small one).

Systemd user unit `~/.config/systemd/user/gw-mcp-oauth.service`:

```ini
[Unit]
Description=google-workspace MCP OAuth front door (claude.ai/ChatGPT/Cowork)
After=gw-mcp-bridge.service
Requires=gw-mcp-bridge.service

[Service]
Type=simple
WorkingDirectory=%h/Developer/Github/google-workspace-mcp
EnvironmentFile=%h/Developer/Github/google-workspace-mcp/gw-oauth-proxy.env
ExecStart=%h/services/cs-voice-mcp/venv/bin/python %h/Developer/Github/google-workspace-mcp/gw_oauth_proxy.py
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=read-only

[Install]
WantedBy=default.target
```

Enable: `systemctl --user daemon-reload && systemctl --user enable --now gw-mcp-bridge gw-mcp-oauth`

### Step 2c — Tailscale Funnel routes [APPROVAL REQUIRED]

Add gw routes to the existing funnel (do not disturb cs-voice routes). Mirror the
cs-voice path-to-port mapping. Example shape (confirm exact sub-paths from the proxy):

```
tailscale funnel --bg --set-path /gw/mcp http://127.0.0.1:8021/mcp
tailscale funnel --bg --set-path /gw http://127.0.0.1:8023
```

The OAuth metadata, authorize, token, and register endpoints are served by the proxy
on :8023 under the `/gw` base. Verify with `tailscale funnel status`.

### Step 2d — Connect the browser clients

- claude.ai web / Cowork: Settings, Connectors, Add custom connector, URL
  `https://kvm8.tailab446d.ts.net/gw/mcp`. It will run the OAuth dance; the proxy
  auto-approves the single user.
- ChatGPT: add as a custom MCP connector with the same URL.
- Headless agents that you would rather route through HTTP than local stdio: use
  `mcp-remote` pointing at the same URL, exactly like your existing
  `mcp-remote ... https://mcp.vercel.com` entries.

### Optional Step 2e — Cloudflare worker layer

If you specifically want a `workers.dev` hostname in front (to match
`cloudflare-mcp.stevelang771.workers.dev`), deploy a worker that proxies to
`https://kvm8.tailab446d.ts.net/gw/mcp`. Copy the structure of the cs-voice worker
(`cloudflare/worker/src/index.ts`, located on the machine it was deployed from). This
is cosmetic ingress; the Funnel URL already works for all browser clients.

---

## Verify and harden

- `systemctl --user status gw-mcp-bridge gw-mcp-oauth`
- `curl -s https://kvm8.tailab446d.ts.net/gw/.well-known/oauth-protected-resource` returns metadata.
- One real tool call from claude.ai (for example list calendars) to confirm end to end.
- Confirm destructive tools are absent in each client's tool list.
- Keep the master key and token at `600` (already are). Consider moving the master key
  out of the repo dir to a `700` dir readable only by george; it is currently next to
  the encrypted token.
- The OAuth proxy is the only thing protecting a privileged Workspace identity on the
  public internet. The single-user auto-approve means anyone who completes the GitHub
  login path the proxy enforces gets in. Confirm the proxy's allow check is tied to
  your identity, mirroring cs-voice's `Hrolf556`-only gate, before going public.

## Handoff to the parallel content-factory (sheets) build

When Tier 2 is live, record these so the sheets connector can mirror the pattern under
`/sheets` without re-deriving it:

- Exact path of the copied proxy (`gw_oauth_proxy.py`) and its env file.
- Systemd user unit names (`gw-mcp-bridge.service`, `gw-mcp-oauth.service`).
- The Funnel `/gw` sub-path mapping actually used: which sub-path goes to the bridge
  (`/mcp`) versus the proxy (`/token`, `/register`, `/authorize`, `.well-known/...`),
  matching whatever the proxy expects.
- The Python venv used (reused cs-voice venv vs a dedicated one) and its `aiohttp` dep.

## Decisions and unknowns to resolve during build

1. Is `rusty@rustycole.com` your Workspace super-admin? If yes, weigh whether the
   public endpoint should exist at all, or stay tailnet-only (Tier 1 + tailnet HTTP).
2. Exact OAuth proxy route topology (path split) must be read from the cs-voice
   original; the gw mapping mirrors it.
3. supergateway streamable-HTTP flag names can vary by version; pin a version and
   confirm `/mcp` is served.
4. Whether to also build a static-key REST + stdio shim (like cf-voice-shim) for
   headless agents, or just use Tier 1 local stdio for them (simpler; recommended).
5. Whether the outer Cloudflare worker layer is wanted, and where its source lives.

## Conventions reminder for whoever executes this

- Back up any config before editing; edit JSON/TOML/YAML with a parser, never sed/echo.
- Long or multi-step runs go in a named tmux session with a paired monitor script.
- `git pull --rebase --autostash` before any orchestrated run.
- Do not broad-pattern kill; target PIDs or named tmux sessions.
- Civilian time in any status output.
