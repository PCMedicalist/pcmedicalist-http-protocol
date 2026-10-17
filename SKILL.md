---
name: pcmedicalist-http-protocol
description: Use when building or auditing PCMedicalist HTTP/API surfaces — status codes, method semantics, keep-alive, long polling, ETags, OpenAPI, request limits, compression.
version: 1.0.0
author: PCMedicalist
license: CC-BY-4.0 (PCMedicalist attribution required)
metadata:
  hermes:
    tags: [http, api, rest, protocol, pcmedicalist]
    related_skills: [pcmedicalist-api-versioning, pcmedicalist-cdn-caching, pcmedicalist-cache-control-headers, pcmedicalist-cors-policy, pcmedicalist-csrf, pcmedicalist-webhooks-hmac, pcmedicalist-grpc, pcmedicalist-server-sent-events]
---

# PCMedicalist HTTP Protocol

## When to Use
- Designing or reviewing any PCMedicalist REST/HTTP API (baseLINE, agent gateways, x402 endpoints).
- Choosing correct status codes / methods for a new endpoint.
- Adding ETag / conditional requests, compression, or request-size limits.
- Documenting an API with OpenAPI and keeping it in sync with code.
- Debugging keep-alive exhaustion, oversized payloads, or cache-vs-freshness mismatches.

## Portfolio Reality
baseLINE exposes a JSON REST API over Express (PM2, single VPS) with Privy/Redis auth, plus x402-gated endpoints and Telegram/Discord bot webhooks. The 0x-agents fleet calls internal APIs. Current gaps: no OpenAPI doc, inconsistent status codes, no ETag/conditional GET, no explicit request-size limit (only the Express body parser default). This skill standardizes the HTTP layer. Related: `pcmedicalist-api-versioning`, `pcmedicalist-cache-control-headers`, `pcmedicalist-cdn-caching`.

## Topics Covered
1. HTTP Status Codes
2. HTTP Method Semantics
3. Keep-Alive
4. Long Polling
5. ETags
6. OpenAPI Specs
7. Backward Compatibility
8. Request Size Limits
9. Compression

Each topic: **Rule → Pattern → Validate → Fix**.

---

## 1. HTTP Status Codes
**Rule:** Use semantically correct codes so clients/proxies/CDNs behave. Don't return `200` for errors or `404` for auth failures.

**Pattern (correct mapping):**
| Situation | Code |
|---|---|
| Success, resource created | `201 Created` (+ `Location`) |
| Accepted, async (outbox/shed) | `202 Accepted` |
| No content (delete success) | `204 No Content` |
| Client input invalid | `400 Bad Request` / `422 Unprocessable` |
| Unauthenticated | `401 Unauthorized` |
| Forbidden (authz) | `403 Forbidden` |
| Not found (and not existence-leaking — see security-fundamentals #21) | `404` or `401` |
| Rate limited / shed | `429 Too Many Requests` (+ `Retry-After`) |
| Overloaded (critical path) | `503 Service Unavailable` |
| Gateway/upstream dead | `502 Bad Gateway` |

```ts
app.post('/api/tips', requireAuth, (req,res) => {
  const parsed = TipSchema.safeParse(req.body);
  if (!parsed.success) return res.status(422).json({ errors: parsed.error.flatten() });
  // ...
  return res.status(201).location(`/api/tips/${tip.id}`).json(tip);
});
```

**Validate:** Hit each endpoint with valid + invalid input; the code matches the table. A 429 carries `Retry-After`. No `200` for errors, no `500` for client mistakes.

**Fix:** Grep for `res.status(200)` on error branches. Replace `500` thrown from input bugs with `400/422`. Standardize via an error-handling middleware that maps domain errors → codes.

---

## 2. HTTP Method Semantics
**Rule:** Methods have meaning — GET is safe/idempotent (no side effects, cacheable), POST creates, PUT replaces, PATCH partial, DELETE removes. Never put state-changing logic in GET (CSRF, prefetch, crawler hazards).

**Pattern:**
```
GET    /api/tips           -> list (safe, cacheable)
GET    /api/tips/:id       -> read
POST   /api/tips           -> create (non-idempotent)
PUT    /api/tips/:id       -> replace whole
PATCH  /api/tips/:id       -> partial update (idempotent if repeated)
DELETE /api/tips/:id       -> remove
```
Never use GET for actions: `GET /api/tips/:id/confirm` is wrong → use `POST`. Idempotency keys (`Idempotency-Key` header) make POST retries safe (see `pcmedicalist-idempotency`).

**Validate:** A prefetch/crawler hitting all GET routes causes zero state change. Repeating a PATCH with the same body yields the same final state (idempotent). No `GET` with side effects in the route table.

**Fix:** Audit routes: any GET that writes → convert to POST/PATCH. Add CSRF protection to state-changing methods (see `pcmedicalist-csrf`). Document idempotency expectations per endpoint.

---

## 3. Keep-Alive
**Rule:** Reuse TCP connections (keep-alive) to avoid TLS+TCP handshake per request — critical for high-frequency agent-internal calls and outbound API (Bankr, on-chain RPC).

**Pattern (Node agent / outbound):**
```ts
import https from 'https';
const agent = new https.Agent({ keepAlive: true, maxSockets: 50, maxFreeSockets: 10 });
// pass agent to fetch/axios for connection reuse
const res = await fetch(url, { agent } as any);
// server side (nginx): keepalive_timeout 65; keepalive_requests 1000;
```
Set `Connection: keep-alive` (HTTP/1.1 default) and tune `maxSockets` so you don't exhaust FDs. On the server, ensure the proxy keeps upstream connections alive.

**Validate:** Under load, `ss -tn | wc -l` (or `netstat`) shows a bounded, reused connection count, not a new socket per request. Time-to-first-byte drops after the first request (no repeated handshake).

**Fix:** If you see connection churn (FD exhaustion / `EMFILE`), enable `keepAlive` on the outbound agent and raise `maxSockets`. On nginx/Express, confirm `keepalive_timeout` is set (not 0). Don't disable keep-alive "for simplicity" in prod.

---

## 4. Long Polling
**Rule:** A client holds a GET open until an event occurs or a timeout; server responds with the data or 304/no-content. Use only when WebSockets/SSE aren't available. Prefer SSE (topic not here — see `pcmedicalist-server-sent-events`) or WS (`pcmedicalist` has the skill) for true streaming.

**Pattern:**
```ts
app.get('/api/tips/:id/wait', async (req, res) => {
  const timeout = setTimeout(() => res.status(304).end(), 30000); // no update
  const unsub = subscribe(`tip:${req.params.id}`, (data) => {
    clearTimeout(timeout); unsub();
    res.status(200).json(data);
  });
  req.on('close', () => { clearTimeout(timeout); unsub(); });  // cleanup on client disconnect
});
```
Always set a max hold time (30s) and clean up on `close` or you leak listeners/sockets. Use a short poll loop on the client.

**Validate:** Client opens the request; event arrives within the hold → 200 with data. No event → 304 at 30s, client reconnects cleanly. No socket/listener leak after 100 open/close cycles (`listenerCount` stable).

**Fix:** If long-poll handlers don't clean up on `close`, add the cleanup. Cap concurrency (each open poll holds a connection — count against keep-alive limits). Migrate to SSE/WS for persistent streams.

---

## 5. ETags
**Rule:** ETags enable conditional requests (`If-None-Match`) → `304 Not Modified` → no body transfer → bandwidth + origin savings. Pair with cache-control for immutable-ish assets.

**Pattern:**
```ts
app.get('/api/creators/:id', async (req, res) => {
  const c = await getCreator(req.params.id);
  const etag = crypto.createHash('sha1').update(JSON.stringify(c)).digest('hex');
  res.setHeader('ETag', `"${etag}"`);
  if (req.headers['if-none-match'] === `"${etag}"`) return res.status(304).end();
  res.json(c);
});
```
For static assets, let the CDN/file server compute ETags; for API JSON, hash the payload (or a `version`/`updatedAt` field — cheaper than hashing). Strong ETag = byte-exact; Weak (`W/`) = semantically equivalent.

**Validate:** First GET returns `ETag`. Second GET with `If-None-Match: <etag>` → `304` and zero body bytes. Changing the resource → new ETag → `200` with new body.

**Fix:** If repeated identical GETs transfer full bodies, add ETags + `If-None-Match` handling. Don't put ETags on endpoints that must always be fresh (balances) unless the value is included. See `pcmedicalist-cache-control-headers` for the cache pairing.

---

## 6. OpenAPI Specs
**Rule:** Every public/internal API has a machine-readable contract (OpenAPI 3.x). It's the source of truth for clients, codegen, and validation. No "undocumented endpoint" in prod.

**Pattern:**
```yaml
openapi: 3.0.3
info: { title: baseLINE API, version: 1.4.0 }
paths:
  /api/tips:
    post:
      operationId: createTip
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/TipInput' }
      responses:
        '201': { description: created, content: { application/json: { schema: { $ref: '#/components/schemas/Tip' } } } }
        '422': { description: validation error }
components:
  schemas:
    TipInput: { type: object, required: [creatorId, amountWei], properties: { creatorId: { type: string, format: uuid }, amountWei: { type: string } } }
```
Generate from code (NestJS/tspec) or maintain `openapi.yaml` + validate requests with a middleware (express-openapi-validator) so the spec is enforced, not decorative.

**Validate:** `openapi.yaml` exists, parses, and `express-openapi-validator` rejects a request that violates a schema (proving spec == enforcement). A client generated from the spec can call the API.

**Fix:** If there's no OpenAPI, start one (even partial) and enforce it. Add a CI check that the spec validates and (ideally) that code matches. Keep version in sync with `pcmedicalist-semantic-versioning` + `pcmedicalist-api-versioning`.

---

## 7. Backward Compatibility
**Rule:** Once an API is consumed (bot, frontend, partner), changing/removing a field or behavior breaks clients. Version deliberately; deprecate, don't yank.

**Pattern:**
- Additive changes (new optional field) = safe, no version bump needed but document.
- Breaking changes (remove/rename field, change type, new required) = new version (`/v2/...`), keep `/v1` until consumers migrate (sunset window).
- Mark deprecated fields with `deprecated: true` in OpenAPI + a log warning when used.
- Never change a response code's meaning for an existing endpoint.

**Validate:** A client built against the old spec still works against the current deployment (additive-only on the live version). Breaking changes only appear under the new version path. No consumer broke during a non-major deploy.

**Fix:** If a deploy broke a client, you changed a contract without versioning — restore the old behavior or issue a new version and communicate a sunset. Adopt `pcmedicalist-api-versioning` with a deprecation policy. Add contract tests (consumer-driven) to CI.

---

## 8. Request Size Limits
**Rule:** Unbounded request bodies = memory-exhaustion DoS. Cap payload size per route (and globally).

**Pattern:**
```ts
import express from 'express';
app.use(express.json({ limit: '1mb' }));                       // global cap
app.use('/api/avatar', express.json({ limit: '5mb' }));        // route-specific larger
// reject oversized with 413 automatically (body-parser does this)
// also enforce at the proxy: nginx client_max_body_size 5m;
```
Set the global limit tight (e.g. 1MB for JSON APIs) and only widen specific routes (uploads). Enforce at the edge (nginx/Vercel) too so oversized requests never reach the app.

**Validate:** POST a 10MB body to a 1MB-limited route → `413 Payload Too Large`, app memory unaffected. A legit 500KB request succeeds. The limit is set at both app and proxy.

**Fix:** If `express.json()` has no `limit` (default 100KB but easy to forget on raw parsers), add explicit limits. Add proxy-level limits. Reject, don't buffer-and-crash, on oversize.

---

## 9. Compression
**Rule:** Compress text payloads (JSON, HTML, CSS, JS) to cut bandwidth + latency; never compress already-compressed formats (images, video, gzip'd assets) — waste of CPU.

**Pattern:**
```ts
import compression from 'compression';
app.use(compression({ filter: (req,res) => {
  if (res.getHeader('Content-Type')?.toString().includes('image')) return false; // skip binaries
  return compression.filter(req, res); // default: compress text types
}, threshold: 1024 }));
// also enable at the CDN/edge (brotli preferred over gzip)
```
Use **brotli** (preferred) or gzip; set `threshold` so tiny bodies aren't compressed. Ensure `Content-Encoding` is set and the client can negotiate (`Accept-Encoding`).

**Validate:** `curl -H 'Accept-Encoding: br' https://baseline.click/api/creators` returns `Content-Encoding: br` with a smaller payload. Binary assets are NOT compressed (no wasted CPU). CPU overhead is negligible vs bandwidth gain.

**Fix:** If responses are uncompressed, add `compression` middleware + edge brotli. Exclude binaries. Verify `Accept-Encoding` negotiation works end-to-end (CDN must pass through / compress).

---

## Validation Checklist
- [ ] Status codes match the semantics table; no 200-for-error (1)
- [ ] No GET with side effects; idempotency keys on POST (2)
- [ ] Keep-alive enabled on client agent + server (3)
- [ ] Long-poll has timeout + close cleanup (4)
- [ ] ETags + 304 on cacheable GETs (5)
- [ ] OpenAPI spec exists + enforced (6)
- [ ] Breaking changes versioned, not yanked (7)
- [ ] Request-size limits at app + proxy (8)
- [ ] Compression on text, skipped on binaries (9)

## Fix Recipes
- **"Wrong codes everywhere"** → topic 1: central error→code mapper.
- **"GET doing writes"** → topic 2: move to POST/PATCH + CSRF.
- **"FD exhaustion under load"** → topic 3: enable keepAlive + tune maxSockets.
- **"Huge bandwidth on repeat reads"** → topics 5 + `pcmedicalist-cache-control-headers`: ETags + cache-control.
- **"Client broke after deploy"** → topics 6–7: OpenAPI + versioning discipline.
- **"OOM on big uploads"** → topic 8: request-size limits.
- **"Slow transfers"** → topic 9: enable brotli.

## Cross-References
- `pcmedicalist-api-versioning` — topic 7 versioning
- `pcmedicalist-cache-control-headers` — topics 5, 9 cache pairing
- `pcmedicalist-cdn-caching` — edge compression/cache
- `pcmedicalist-cors-policy` — cross-origin surface
- `pcmedicalist-csrf` — state-changing methods (topic 2)
- `pcmedicalist-webhooks-hmac` — inbound webhook validation
- `pcmedicalist-grpc` — binary RPC alt to REST
- `pcmedicalist-server-sent-events` / `pcmedicalist-websockets` — streaming vs long-poll (topic 4)
- `pcmedicalist-idempotency` — POST idempotency keys (topic 2)
- `pcmedicalist-security-fundamentals` — topic 21 (404 vs 401 enumeration)
