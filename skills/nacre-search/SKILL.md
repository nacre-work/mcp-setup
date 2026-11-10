---
name: nacre-search
description: Search, read and add documents in a Nacre installation through the `nacre` MCP server, and read its answers correctly. Use whenever a question should be answered from the organization's own documents in Nacre, before ingesting or deleting anything there, and whenever a Nacre search comes back empty or a document is "not found" - both have a specific meaning here that a retry will not change.
---

# Working with Nacre

Nacre is a self-hosted knowledge index with access control built into the
search itself. The `nacre` MCP server is the installation's own `/mcp`
endpoint, and every call runs as the principal that signed in on the consent
screen: you, by default, or a service account. Results are filtered to what
that principal may read, and the filter runs inside the index rather than over
its output.

That one fact changes how to read every answer below.

## Before the first call

- **Which installation?** The plugin connects to `$NACRE_MCP_URL`, and to the
  public playground (`https://playground.nacre.work/mcp`) when it is not set.
  The playground is a demo stand: organizations are erased after 24 hours, with
  a cap of 200 documents and 500 searches a day, and no PDF upload. Never put
  anything there you want to keep, or anything private.
- **Which layers?** Call `list_layers` first. It names the layers this
  connection may read, with descriptions and document counts. Layers are
  addressed by slug.

## Searching

- `search` is hybrid: it matches meaning and exact terms both. Identifiers,
  error codes, part numbers and names match literally, so put them in the query
  as written rather than paraphrasing them.
- Narrow with `layers` (slugs) and with metadata filters when you know the keys.
- `get_document` fetches one document by id, or by external id within a layer.

## Reading the answers

**An empty result means nothing you may see matched.** It is not an error, and
rephrasing the same request will not change it. Say so, and say what was
searched.

**"Not found" means "no such document" or "not yours to read", deliberately
the same.** Nacre answers both identically so that probing ids cannot reveal
what exists. Do not retry with variations to tell them apart, and do not tell
the user a document does not exist when all you know is that you cannot see it.
"I can't find that in the layers this connection can read" is the honest
sentence.

**A short result list is real.** Because the filter runs inside the index, a
request for ten results returns up to ten results this principal may read, not
ten minus whatever was removed afterwards. Fewer results means fewer matches.

If the user expected something and it is not there, the two things to check are
the connection's consent ceiling (which layers it was approved for) and what
their account is granted. Both are settled in the installation, not here.

## Writing

Only if this connection was approved for write. A read-only client cannot
ingest or delete, and that is the default.

- **Write does not imply read.** An ingest-only connection may be unable to
  search what it just wrote. A successful ingest followed by an empty search
  can be two correct answers.
- `ingest_document` adds or updates text, or a public URL, in a layer. It is
  idempotent on `external_id`: the same id again replaces that document, a new
  id adds a second one. Reuse the id when you mean to update.
- **An ingest answers `queued`, which is not success.** Call `ingest_status`
  with the job id until it says `indexed`. `failed` comes with a reason; read it
  before sending again.
- `delete_document` takes a document out of every search at once. Confirm with
  the user before deleting anything you did not create in this session.

## When something fails

- `401`, or asked to sign in again: the token expired. Reconnect with `/mcp`.
- `403 Origin not allowed` from a browser client: the installation admits
  browser origins only through `NACRE_MCP_ALLOWED_ORIGINS`. CLI and desktop
  clients are unaffected.
- The playground forgot everything: it erases organizations after 24 hours.

More: https://nacre.work/quickstart and `docs/mcp.md` in
https://github.com/nacre-work/nacre.
