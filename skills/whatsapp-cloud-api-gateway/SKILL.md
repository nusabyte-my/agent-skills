---
name: whatsapp-cloud-api-gateway
description: Build a multi-tenant WhatsApp Business Cloud API gateway (FastAPI + Postgres RLS) that receives webhooks and sends compliant messages without getting numbers banned. Use for WhatsApp bots, send engines, webhook endpoints, or multi-client WhatsApp SaaS.
license: MIT
---

# WhatsApp Cloud API gateway

A server that is both a public webhook endpoint and a compliant send engine, for one
number or for many clients each with their own WhatsApp Business account.

## When to use

- "I need a WhatsApp API server" / "send messages from my own endpoint"
- "A blaster that doesn't get banned"
- Multiple clients or brands, each with their own WhatsApp number
- Inbound webhook handling, auto-reply, or a WhatsApp chatbot

**Not this skill:** the unofficial libraries. See the first pitfall.

## The fork in the road: never start with Baileys

`whatsapp-web.js` / Baileys drive a **personal WhatsApp account** over the
multi-device protocol. Numbers get banned for it, quickly and permanently, and the
ban is on the client's real business number. There is no "careful" version of this.

Meta Cloud API is free for the first 1,000 service conversations per month per WABA,
and it cannot be banned for automating. Start there.

The one legitimate use of Baileys is as a **fallback** for a number the business
already owns and cannot migrate, driven from a **residential IP** (a VPS datacenter
IP fails the QR registration step outright). If you build the hybrid, the router
must try Meta first and fall through on 4xx/5xx/timeout, with the fallback clearly
logged — never silently.

## Architecture

```
Meta ──webhook──▶ FastAPI :8083 ──▶ Postgres (RLS) ──▶ Ollama / your LLM
                        │
                        └──▶ Graph API (send)
```

Expose through an existing Cloudflare tunnel (`cloudflared`) rather than opening a
port. **The tunnel's DNS zone decides which hostnames resolve.** Check
`cloudflared tunnel list` and the zone first: a hostname in the wrong zone creates a
CNAME that never resolves, and it fails on the box itself before it fails anywhere
else.

Per-tenant webhook URL: `https://<host>/webhook/whatsapp/<slug>` — each client points
their Meta app at their own slug.

## Anti-ban mechanisms to bake in, not bolt on

1. **Opt-in store, per tenant.** Never message a number that has not opted in.
   An inbound message *is* implied opt-in; store it on receipt.
2. **24-hour service window.** Free-form text is only legal within 24h of the last
   inbound message. Outside it, you must use an approved message template. Enforce
   this in the send path, not in the caller.
3. **Pacing.** A per-tenant token bucket (per-minute cap), a per-recipient cooldown,
   and a daily cap. Randomized jitter between sends.
4. **Auto back-off.** On a `failed` status webhook, drop the opt-in. Continuing to
   hammer a number that blocked you escalates to a quality-rating drop.
5. **Signature verification.** `X-Hub-Signature-256` must equal
   `sha256=HMAC_SHA256(app_secret, raw_body)` — computed over the **raw bytes**,
   before any JSON parsing. Per-tenant secret.

## Multi-tenant isolation: Postgres RLS, not `WHERE tenant_id = ?`

Application-level scoping means one forgotten `WHERE` clause leaks client A's chats
into client B's panel. Use row-level security so that leak is structurally
impossible.

```sql
ALTER TABLE optins ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON optins
  USING (tenant_id = current_setting('app.current_tenant_id', true)::uuid);
```

Tenant resolution order: path slug → `X-Api-Key` → `X-Tenant-Slug`.

### The pin that must not use SET LOCAL

**This is the single most common silent breaker.** `SET LOCAL app.current_tenant_id = $1`
fails two ways:

1. Postgres rejects a bound parameter as the right-hand side of `SET` →
   `syntax error at or near "$1"`.
2. `SET LOCAL` is **transaction-scoped** and is reset the moment a `commit()` runs.
   Store helpers commit mid-session, so every later query in that session loses the
   pin — and RLS then returns **zero rows, not an error**. You get empty lists and
   no exception to chase.

Pin with a session-level `set_config` instead:

```python
await session.execute(
    text("SELECT set_config('app.current_tenant_id', :tid, false)"),  # false = session
    {"tid": str(tenant_id)},
)
```

Admin bypass is a separate, literal-valued setting and may stay `true`-scoped:

```sql
CREATE POLICY admin_bypass ON optins
  USING (current_setting('app.bypass_rls', true) = 'on');
```

The app connects as a **non-superuser** role — superusers bypass RLS entirely, which
silently defeats the whole design.

## Schema

Four tables carry the product: `tenants` (credentials, model, prompt, pacing knobs),
`optins`, `events`, `daily_counter`. Each has `tenant_id` and an isolation policy.
The `tenants` table itself is admin-only.

## Build order (each step fails loudly if the previous one is wrong)

1. Write the files locally: `app/{config,db,store,whatsapp,llm,main}.py`, `Dockerfile`,
   `docker-compose.yml`, `.env.example`, `schema.sql`.
2. Create the Postgres role and database. **Quote the password** in SQL — a bareword
   is parsed as an identifier. Write the `.sql` locally and `scp` it rather than
   piping a heredoc through SSH, which mangles `$$` and quotes.
3. `docker compose up -d --build`.
4. `init_schema()` runs on startup — idempotent DDL plus RLS policies.
5. Verify in this order: `/health`, create a tenant, add an opt-in, dry-run a send,
   then a signed inbound webhook.
6. Add the tunnel ingress and `route dns` per hostname.

## Pitfalls

- **`postgresql://` silently selects psycopg2.** An async engine needs
  `postgresql+asyncpg://`. With the bare scheme SQLAlchemy picks psycopg2, which is
  not installed, and the container crash-loops with empty health responses while
  `docker logs` shows `ModuleNotFoundError: psycopg2`. Fix the URL first — it masks
  every other DB error.
- **asyncpg rejects multi-statement `execute()`.** Each `execute()` compiles to one
  prepared statement, so `CREATE TABLE a ...; CREATE TABLE b ...;` raises
  `cannot insert multiple commands into a prepared statement`. Loop over DDL
  statements one call each. This bites `init_schema()` hardest: the schema looks
  right and the gateway dies on first boot.
- **Containers on a bridge network cannot reach host services.** Docker FORWARD drops
  container→host-bridge traffic, so `host.docker.internal` times out. Use
  `network_mode: host` to reach `127.0.0.1:5432` and `127.0.0.1:11434`. The `ports:`
  mapping is then ignored, which is fine for a single-service host.
- **A placeholder password in the live `.env`** connects to nothing. Copying
  `.env.example` verbatim gives `password authentication failed`. Verify the login
  with `psql -tAc "SELECT 1"` before trusting `/health`, and confirm the role owns the
  database and can `CREATE TABLE`.
- **`python-multipart` is required** for any `Form(...)` parameter — FastAPI raises
  at import otherwise.
- **Webhook handler and background task need separate sessions.** Don't share a
  session across the request/background boundary; open one in the handler, `close()`
  in `finally`, and open another in the task with its own tenant pin.
- **`background_tasks.add_task(coro)`** takes the coroutine function's result
  directly — do not wrap in `asyncio.run`.
- **Fresh DNS takes minutes to resolve from the box itself.** Test the new hostname
  with `curl --resolve`, or against `127.0.0.1` locally, before declaring it broken.
- **`token_configured: true` means an env var is set**, not that the token is valid.
  Send one real message before going live.
- **No `rsync` on Windows git-bash.** Use `scp -r src/. user@host:/opt/app/` (the
  trailing `/.` copies contents) or `tar -czf - … | ssh host 'tar -xzf - -C dest'`.
  Never overwrite a live `.env` with the example one.

## Verification checklist

- [ ] `/health` returns 200 and reports the DB reachable
- [ ] A second tenant cannot read the first tenant's rows (test the leak, not the code)
- [ ] Unsigned webhook is rejected with 403
- [ ] Outbound send outside the 24h window is refused unless a template is supplied
- [ ] Opt-in is removed after a `failed` status webhook
- [ ] Pacing cap returns 429 rather than sending
