# MCP Tailscale Funnel Gateway — Systems Design Doc v1.0

**Author:** Collaborative (Dan + Claude)
**Date:** 2026-03-28
**Status:** Draft
**Scope:** Expose all local MCP servers over Tailscale Funnel with bearer token auth

---

## 1. Problem Statement

Dan's MCP infrastructure runs as local stdio processes — tools are only available on the
Windows desktop where Claude Code or Claude Desktop is running. From Android, claude.ai
web, or any other surface, write-capable tools (Gmail label ops, Hearth DB writes) are
inaccessible. The hosted Gmail MCP in claude.ai provides read-only access, but the full
12-tool Gmail server and 7-tool Hearth server are desktop-locked.

**Goal:** Make every local MCP server available on every surface (claude.ai web, Claude
Desktop, Claude Code, Android) by exposing them as authenticated HTTPS endpoints via
Tailscale Funnel.

---

## 2. Current State

### 2.1 MCP Servers

| Server | Location | Transport | Port | Tools | Auth |
|--------|----------|-----------|------|-------|------|
| Hearth | `Hearth/mcp-server/` | stdio (Desktop/Code) + streamable-http (Funnel) | 8080 | 7 | None |
| Gmail  | `gmail-mcp-server/` | stdio only | N/A | 12 | N/A |
| TickTick | `TickTick_MCPServer/` | stdio only | N/A | ~15 | N/A |

### 2.2 Tailscale Funnel

- Machine: `ankylon`
- Funnel domain: `https://ankylon.tail59731d.ts.net`
- Active route: `/` → `127.0.0.1:8080` (Hearth, no auth)
- Tailscale is installed, Funnel is enabled, HTTPS cert is auto-provisioned

### 2.3 Hearth serve.py (reference implementation)

```
Hearth/mcp-server/serve.py
├── ensure_dolt_running()     ← starts Dolt if not listening on 3306
├── imports mcp from server   ← the FastMCP instance
└── mcp.run(transport="streamable-http", host="0.0.0.0", port=8080, path="/mcp")
```

Auto-starts via Windows Task Scheduler → `start-hidden.vbs` → `python serve.py`.

### 2.4 Gmail server.py

```
gmail-mcp-server/mcp_gmail/server.py
├── get_gmail_service()       ← OAuth init, reads credentials.json + token.json
├── mcp = FastMCP("Gmail MCP Server")
└── 12 @mcp.tool() functions + 2 @mcp.resource() functions
```

Key dependency: `mcp==1.6.0` in the venv. Needs upgrade to `>=1.26.0` for
`streamable-http` transport and `mcp.streamable_http_app()`.

---

## 3. Target Architecture

```
                     Internet / Tailscale Funnel
                     https://ankylon.tail59731d.ts.net
                                  │
                    ┌─────────────┼─────────────┐
                    │             │              │
                 /mcp          /gmail/mcp    (future: /ticktick/mcp)
                    │             │              │
              ┌─────┴─────┐ ┌────┴────┐   ┌─────┴─────┐
              │  Hearth   │ │  Gmail  │   │ TickTick  │
              │  :8080    │ │  :8081  │   │  :8082    │
              │  Bearer   │ │ Bearer  │   │  Bearer   │
              └───────────┘ └─────────┘   └───────────┘
                   │              │
              Dolt :3306    Google OAuth
              (auto-start)  (token.json)
```

### 3.1 Routing

Tailscale Funnel supports path-based routing on port 443. Each MCP server gets its own
`--set-path` prefix:

| Path | Backend | Notes |
|------|---------|-------|
| `/` or `/mcp` | `127.0.0.1:8080` | Hearth (existing, needs auth retrofit) |
| `/gmail` | `127.0.0.1:8081` | Gmail (new) |
| `/ticktick` | `127.0.0.1:8082` | TickTick (future, out of scope) |

**Path stripping:** Tailscale Funnel strips the path prefix before forwarding.
A request to `/gmail/mcp` arrives at the Gmail backend as `/mcp`. FastMCP's default
`path="/mcp"` works without modification.

### 3.2 Port Allocation

| Port | Service | Protocol |
|------|---------|----------|
| 443 | Tailscale Funnel (HTTPS) | TLS-terminated by Tailscale |
| 3306 | Dolt sql-server | MySQL wire protocol (local only) |
| 8080 | Hearth MCP | Streamable HTTP (local, fronted by Funnel) |
| 8081 | Gmail MCP | Streamable HTTP (local, fronted by Funnel) |
| 8082 | (reserved) TickTick MCP | Future |

---

## 4. Authentication Design

### 4.1 Threat Model

Tailscale Funnel exposes endpoints to the public internet. Without auth:
- Anyone who discovers the URL can query the Hearth database (read/write)
- Anyone who discovers the URL can read/send/modify Gmail
- MCP protocol doesn't have built-in auth for streamable-http

### 4.2 Bearer Token Auth

Each server gets a unique, randomly generated bearer token stored in a `.env` file.
A Starlette middleware checks the `Authorization: Bearer <token>` header on every
request. Requests without a valid token get HTTP 401.

**Why bearer tokens (not OAuth, mTLS, or API keys)?**
- Simple to implement (20 lines of middleware)
- Supported by claude.ai remote MCP server registration (header-based auth)
- Tokens never leave the machine — `.env` files are gitignored
- Sufficient for a single-user system behind Tailscale's HTTPS
- No key rotation ceremony needed for a personal stack

### 4.3 Middleware Implementation

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class BearerAuthMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, token: str):
        super().__init__(app)
        self.token = token

    async def dispatch(self, request, call_next):
        auth = request.headers.get("authorization", "")
        if auth != f"Bearer {self.token}":
            return JSONResponse({"error": "unauthorized"}, status_code=401)
        return await call_next(request)
```

Shared as a pattern — each server has its own inline copy (no cross-repo imports).

### 4.4 Token Storage

| Server | Env file | Env var | Gitignored by |
|--------|----------|---------|---------------|
| Hearth | `Hearth/mcp-server/.env` | `HEARTH_MCP_AUTH_TOKEN` | needs `*.env` in `.gitignore` |
| Gmail | `gmail-mcp-server/.env` | `GMAIL_MCP_AUTH_TOKEN` | existing `*.env` pattern |

Tokens are 64-character hex strings generated via `python -c "import secrets; print(secrets.token_hex(32))"`.

---

## 5. Server Implementation Details

### 5.1 Gmail serve.py (new file)

```
gmail-mcp-server/serve.py
├── os.chdir() to project root     ← so credentials.json resolves
├── load .env                       ← for GMAIL_MCP_AUTH_TOKEN
├── import mcp from mcp_gmail.server
├── app = mcp.streamable_http_app() ← returns Starlette app
├── app.add_middleware(BearerAuthMiddleware, token=...)
└── uvicorn.run(app, host="0.0.0.0", port=8081)
```

**Why `streamable_http_app()` instead of `mcp.run()`?**
`mcp.run(transport="streamable-http")` calls uvicorn internally but doesn't expose
the Starlette app for middleware injection. By calling `streamable_http_app()` we get
the raw app, add auth, then run uvicorn ourselves.

**Dependency:** Requires `mcp>=1.26.0` (current venv has 1.6.0). The Hearth venv runs
1.26.0 and confirms this works. Upgrade: `pip install "mcp>=1.26.0"`.

### 5.2 Hearth serve.py (retrofit auth)

Modify existing `serve.py` to:
1. Load `.env` for `HEARTH_MCP_AUTH_TOKEN`
2. Switch from `mcp.run()` to `mcp.streamable_http_app()` + middleware + `uvicorn.run()`
3. Keep `ensure_dolt_running()` unchanged

### 5.3 OAuth Token Lifecycle (Gmail)

The Gmail server authenticates to Google via OAuth 2.0. Token flow:

```
First run (interactive):
  credentials.json → InstalledAppFlow → browser → user approves → token.json saved

Subsequent runs (headless):
  token.json loaded → access_token expired? → creds.refresh(Request()) → token.json updated

Refresh token expiry (~6 months, or if user revokes):
  creds.refresh() fails → server can't authenticate → manual re-auth needed
```

**Failure mode:** If the refresh token expires while running headless, the server will
fail silently on every Gmail API call. Mitigation: the server logs the error, and a
health check (future) alerts Dan to re-authenticate.

**Re-auth procedure:** Stop the service, run `python -c "from mcp_gmail.gmail import get_gmail_service; get_gmail_service()"` in a terminal with a browser, approve the OAuth consent, restart the service.

---

## 6. Deployment

### 6.1 Service Management

Each MCP server runs as a Windows Task Scheduler task, launched windowless via VBS:

| Task Name | VBS File | Python Target | Working Dir |
|-----------|----------|---------------|-------------|
| Hearth MCP Serve | `start-hidden.vbs` | `.venv\Scripts\python.exe serve.py` | `Hearth/mcp-server/` |
| Gmail MCP Serve | `start-hidden.vbs` | `.venv\Scripts\python.exe serve.py` | `gmail-mcp-server/` |

Task Scheduler config:
- Trigger: user logon
- Restart on failure: 3 retries, 1-minute interval
- Run hidden, no window
- Unlimited execution time

### 6.2 Tailscale Funnel Commands

Run from PowerShell (not Git Bash — bash mangles path arguments):

```powershell
# Hearth (already active, verify)
tailscale funnel --bg --set-path / http://127.0.0.1:8080

# Gmail (new)
tailscale funnel --bg --set-path /gmail http://127.0.0.1:8081
```

The `--bg` flag makes the route persistent across Tailscale restarts.

### 6.3 Client Registration

#### claude.ai (web + mobile)
Add as remote MCP server in Settings > MCP Servers:
- **Gmail:** URL `https://ankylon.tail59731d.ts.net/gmail/mcp`, Header `Authorization: Bearer <token>`
- **Hearth:** URL `https://ankylon.tail59731d.ts.net/mcp`, Header `Authorization: Bearer <token>`

#### Claude Desktop
Keep existing stdio config (local is faster, no auth overhead). The Funnel endpoints
are a fallback, not a replacement for local connections.

#### Claude Code
Same as Desktop — keep stdio. The Funnel is for surfaces that can't run local processes.

---

## 7. File Manifest

### New files (Gmail)

| File | Purpose |
|------|---------|
| `gmail-mcp-server/serve.py` | Streamable HTTP entry point with auth |
| `gmail-mcp-server/.env` | Bearer token (gitignored) |
| `gmail-mcp-server/start-hidden.vbs` | Windowless launcher |
| `gmail-mcp-server/gmail-serve-task.xml` | Task Scheduler template |

### Modified files (Hearth)

| File | Change |
|------|--------|
| `Hearth/mcp-server/serve.py` | Add auth middleware, switch to `streamable_http_app()` + uvicorn |
| `Hearth/mcp-server/.env` | New file — bearer token (gitignored) |
| `Hearth/.gitignore` | Add `*.env` if not present |

### Unchanged files

- `gmail-mcp-server/mcp_gmail/server.py` — no changes
- `gmail-mcp-server/mcp_gmail/gmail.py` — no changes
- `gmail-mcp-server/mcp_gmail/config.py` — no changes
- `Hearth/mcp-server/server.py` — no changes
- `Hearth/mcp-server/run.py` — no changes (stdio path unaffected)

---

## 8. Verification Plan

### 8.1 Local smoke test (pre-Funnel)

```bash
# Start Gmail server
cd gmail-mcp-server && .venv/Scripts/python.exe serve.py

# Test with auth (should return MCP session init)
curl -X POST http://127.0.0.1:8081/mcp \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{"capabilities":{}},"id":1}'

# Test without auth (should return 401)
curl -X POST http://127.0.0.1:8081/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{"capabilities":{}},"id":1}'
```

### 8.2 Funnel test (post-Funnel)

```bash
# Same tests but via Funnel URL
curl -X POST https://ankylon.tail59731d.ts.net/gmail/mcp \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{"capabilities":{}},"id":1}'
```

### 8.3 End-to-end (claude.ai)

1. Register Gmail MCP in claude.ai settings
2. Open claude.ai on Android
3. Ask Claude to list Gmail labels — should work
4. Ask Claude to create a test label — should work
5. Delete test label manually

### 8.4 Hearth auth retrofit

1. Restart Hearth serve.py with auth
2. Verify existing Funnel URL returns 401 without token
3. Verify it works with token
4. Update any clients that use the Funnel URL

---

## 9. Future Considerations

- **TickTick MCP:** Same pattern, port 8082, path `/ticktick`. Blocked on converting
  the Node.js server to support HTTP transport.
- **Health monitoring:** A simple cron job that curls each endpoint and alerts on failure.
  Could run as a Claude Code scheduled task.
- **Token rotation:** Not urgent for a single-user system. If needed, update `.env` and
  restart the service + update claude.ai config.
- **Consolidated gateway:** If the number of MCP servers grows beyond 3-4, consider a
  single reverse proxy (Caddy) that handles auth + routing, with MCP servers on localhost
  without individual auth. Premature for now.

---

## 10. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| OAuth refresh token expires | Low (6mo+) | Gmail server stops working | Monitor for auth errors, re-auth procedure documented in Section 5.3 |
| Bearer token leaked | Low | Full MCP access | Tokens in .env (gitignored), regenerate + restart if compromised |
| Tailscale Funnel goes down | Low | Remote access lost | Local stdio still works for Desktop/Code. Funnel auto-recovers on Tailscale restart |
| Port conflict on startup | Low | Server won't bind | Task Scheduler retries 3x. Ports are well above ephemeral range |
| mcp package upgrade breaks gmail server | Medium | Server won't start | Pin to `mcp>=1.26.0,<2.0` in requirements |
