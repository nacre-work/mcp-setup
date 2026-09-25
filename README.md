# Nacre — MCP setup

Connect your AI assistant to [Nacre](https://nacre.work), the self-hosted
knowledge index with fine-grained access control. Every Nacre installation
serves MCP itself, so there is nothing to install on the server side: point
your client at **your deployment's** `/mcp` endpoint and sign in. Search
returns only what the signed-in person (or service account) may read —
the filter runs inside the index, so an agent never sees a document it was not
granted, and cannot tell it exists.

- **Server:** `nacre`
- **Transport:** streamable HTTP (`POST /mcp`, stateless) — or stdio for local development
- **URL:** `https://<your-nacre-host>/mcp`
- **Try it:** `https://playground.nacre.work/mcp` — the public demo stand (see below)
- **Auth:** OAuth 2.1 (Authorization Code + PKCE) against your installation — you'll be prompted to sign in.

> **Nacre is self-hosted; there is no hosted Nacre service.** The playground is
> a demo stand: organizations are erased after 24 hours, with a cap of 200
> documents and 500 searches a day, and no PDF upload. Use it to see the
> permission model work, not to keep anything. Deploy your own with
> `docker compose --profile demo up` — see the
> [quickstart](https://nacre.work/quickstart).

## What you can do

Six tools. What the client may actually use is bounded by the ceiling chosen
on the consent screen — which layers, and whether it may write at all. *A search
client that cannot delete a document is the default, not a setting you have to
find.*

| Tool | What it does | Needs |
|---|---|---|
| `search` | Hybrid search (meaning + exact terms — identifiers, error codes, names) over the layers you may read. Optional layer and metadata filters. | read |
| `list_layers` | The layers you can read, with descriptions and document counts (paged). | read |
| `get_document` | One document by id, or by external id within a layer. | read |
| `ingest_document` | Add or update text or a URL in a layer; idempotent on `external_id`, asynchronous. | write |
| `ingest_status` | What became of an ingest: `indexed`, or `failed` with a reason. | write |
| `delete_document` | Tombstone a document — it leaves results immediately. | write |

In Nacre `write` does not imply `read`: an ingest-only client cannot read back
what it wrote. That is deliberate.

---

## Install in Claude Code (plugin)

```
/plugin marketplace add nacre-work/mcp-setup
/plugin install nacre@nacre
/reload-plugins
```

The plugin connects to `$NACRE_MCP_URL`, and to the public playground when that
variable is not set. Point it at your own installation before starting Claude
Code:

```bash
export NACRE_MCP_URL=https://nacre.example.com/mcp
```

Then run `/mcp` and complete the sign-in when prompted.

> Prefer not to use the marketplace? Add the server directly:
> ```
> claude mcp add --transport http nacre https://nacre.example.com/mcp
> ```

---

## Connect from other clients

Same URL, same sign-in — only the config location differs. Replace
`nacre.example.com` with your installation (or use `playground.nacre.work` to
try it).

### Claude Desktop / claude.ai connectors

1. Open **Settings → Connectors → Add custom connector**.
2. Name: `Nacre`
3. URL: `https://nacre.example.com/mcp`
4. Save, connect, and approve on your installation's consent screen.

### Cursor

`~/.cursor/mcp.json` or `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "nacre": {
      "url": "https://nacre.example.com/mcp"
    }
  }
}
```

### VS Code (GitHub Copilot / MCP) and other clients

```json
{
  "servers": {
    "nacre": {
      "type": "http",
      "url": "https://nacre.example.com/mcp"
    }
  }
}
```

### Local mode (stdio, developer laptops)

`@nacre.work/mcp` runs the whole server in-process, authenticated by a
service-account key. It is not a thin client: it needs the deployment's full
environment (database, vector store, parser, embedder), so it fits a developer
running Nacre locally, not a remote installation.

```bash
set -a && . ./.env && set +a
NACRE_SERVICE_KEY=nacre_sk_… npx @nacre.work/mcp
```

An agent in local mode holds your database credentials — use Streamable HTTP for
anything that is not your own laptop.

---

## Authentication

The MCP endpoint is an OAuth 2.1 resource server:

- An unauthenticated call answers `401` with a `WWW-Authenticate` header naming
  the RFC 9728 document at `https://<host>/.well-known/oauth-protected-resource`.
- The authorization server is your installation's own API: RFC 8414 discovery,
  RFC 7591 dynamic client registration, Authorization Code + **PKCE (S256)**.
  An installation can name its own identity provider instead.
- Tokens are audience-bound to that installation's MCP endpoint and refused
  anywhere else.
- On the consent screen you choose whether the client acts **as you** (the
  default — it sees exactly what you see, and stops when your account is
  disabled) or as a service account, and set its ceiling: which layers, and
  read or write.

Compliant MCP clients discover all of this from the server URL.

---

## Troubleshooting

- **`401` / asked to sign in again** — the token expired; reconnect in the client.
- **A search returns nothing you expected** — Nacre answers "no permission" and
  "no such document" identically, on purpose. Check which layers the connection
  was approved for (consent ceiling) and what your account is granted.
- **`403 Origin not allowed` from a browser client** — the installation admits
  browser origins only through `NACRE_MCP_ALLOWED_ORIGINS`. Desktop and CLI
  clients need no CORS.
- **The playground forgot everything** — it erases organizations after 24 hours.

## Links

- Website: https://nacre.work
- Quickstart: https://nacre.work/quickstart
- Source (Apache-2.0): https://github.com/nacre-work/nacre — MCP details in `docs/mcp.md`
- Demo: https://demo.nacre.work
- Contact: https://nacre.work/contact · security: security@nacre.work
