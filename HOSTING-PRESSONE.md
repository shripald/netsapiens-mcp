# Hosting the NetSapiens MCP for PressONE (plan, not deployed)

Hosted mode (`MCP_TRANSPORT=http`) lets each person add the server as a
connector in Claude and sign in with their own NetSapiens login on a
PressONE-branded page. Their NetSapiens permissions decide what the AI can do.

## Options

| | A. Single instance + file token store | B. Cloud Run + Firestore |
|---|---|---|
| Where | Mac mini (public via Tailscale Funnel or Cloudflare Tunnel) **or** Render web service with a persistent disk | Google Cloud Run, tokens in Firestore |
| Setup effort | Low | Medium (GCP project, Firestore, service account) |
| Cost | Mac mini: free. Render: paid instance + disk (a persistent disk needs a paid plan) | Pay per use; small for a handful of users |
| Survives restarts | Logins yes (file on disk). Client registrations no (in memory), so Claude may ask people to reconnect after a restart | Everything (tokens, registrations, auth codes) |
| Scaling | One instance only (the file can't be shared) | Many instances |
| Uptime | Mac mini depends on your power and internet. Render is managed | Managed |
| Upstream fit | Works today | The setup upstream uses and tests in CI |

**Must be public HTTPS.** Claude's custom connectors (claude.ai, Desktop,
mobile) are reached from Anthropic's cloud, not from your computer, so a
private Tailscale address alone won't work. On the Mac mini, use **Tailscale
Funnel** or **Cloudflare Tunnel** to publish one HTTPS hostname, for example
`mcp.pressone.net`.

**Recommendation:** start with **A on Render** (or the Mac mini with a tunnel
if you'd rather not pay) for a small pilot. Move to **B** once more than a few
people rely on it or you need zero-downtime deploys.

### Option A settings
```
MCP_TRANSPORT=http
MCP_PERSISTENCE=file
MCP_TOKEN_STORE_PATH=/var/data/http-tokens.json   # path on the persistent disk
```
Run exactly **one** instance.

### Option B settings
```
MCP_TRANSPORT=http
MCP_PERSISTENCE=firestore
GOOGLE_CLOUD_PROJECT=<your-gcp-project>
```
The Cloud Run service account needs the Firestore "Cloud Datastore User" role.

## Settings for both options

Required:

| Variable | Value |
|---|---|
| `NETSAPIENS_API_URL` | `https://api.pressone.net` |
| `NETSAPIENS_OAUTH_CLIENT_ID` / `NETSAPIENS_OAUTH_CLIENT_SECRET` | A NetSapiens OAuth client created for this server (keep it in the host's secret store) |
| `MCP_BASE_URL` | The public URL, e.g. `https://mcp.pressone.net` |
| `MCP_SESSION_SECRET` | 64 random hex characters (`openssl rand -hex 32`). Keep it stable, or in-progress sign-ins break |

PressONE branding:

| Variable | Suggested value |
|---|---|
| `MCP_LOGIN_HEADER` | `PressONE AI: Sign In` |
| `MCP_SERVER_TITLE` | `PressONE NetSapiens` (name shown in Claude) |
| `MCP_ICON_URL` | Public URL of a square PressONE logo (PNG). Used as the favicon and login logo |
| `MCP_LOGIN_LOGO_URL` | Optional: a different logo for the login page only |
| `MCP_WEBSITE_URL` | `https://pressone.net` |

Safe defaults:

| Variable | Value | Why |
|---|---|---|
| `MCP_CONFIRM_DESTRUCTIVE` | `true` | Claude must ask the person "Yes/No" before anything destructive (ending calls, deletes) |
| `MCP_STATELESS` | `false` | The confirmation prompt needs this. It's also switched on automatically when the line above is set and this is left blank |
| `MCP_DISABLED_TOOLS` | `delete_*,remove_*` | Hides delete and remove tools entirely |
| `MCP_CORS_ORIGIN` | `https://claude.ai` | Default is `*` |
| `MCP_TOKEN_LIFETIME_HOURS` | `168` (default) | How long before a person must sign in again |
| `MCP_READ_ONLY` | `true` (optional) | Run a second, look-only instance for support staff |

## Not covered here
DNS, TLS certificates, and choosing between Render, the Mac mini, and Cloud
Run. Decide those when you're ready to deploy.
