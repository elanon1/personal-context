---
type: project
project: Hexbane
area: server
status: active
created: 2026-09-15
updated: 2026-09-15
verified: 2026-09-15
tags: [hexbane, news, rpc, cache, administration]
sources: ["server:modules/news", "server:db/migrations/000009_news.up.sql", "client:Game/ScenesV3/News/NewsData.cs"]
---

# Server-managed news

## Scope and client mapping

The server implements persistent articles and administrator writes. The existing client still reads the hardcoded `NewsData.Articles`; switching Dashboard and NewsScreen to RPC is a separate integration step. No sample client announcements were published or seeded: they include obsolete gameplay claims.

| Client field | API / database | Contract |
|---|---|---|
| Title | `title` | required text, ≤200 UTF-8 bytes |
| Summary | `summary` | required text, ≤500 bytes |
| Content | `content` | required plain text, ≤16384 bytes; preserve newlines, headings ending in `:` and `- ` list items |
| Date | `date` / `article_date DATE` | `YYYY-MM-DD`; client formats for display (old fixtures used `Mar 12, 2026`) |
| Category | `category` | `Announcement`, `Update`, `Event`, `Status`; client selects icon/color locally |
| Highlight | `highlight` | boolean |

Additional fields: stable `id` (1–80 lowercase ASCII letters/digits/underscore/hyphen, first character letter/digit), `published`, timestamps and creator/editor account IDs. Public output includes `id` and `updated_at`, excludes operator account IDs. Images are not currently article data; dashboard icons cycle by index, article list icons derive from category.

## `get_news`

Authenticated Nakama RPC, request `{}` (empty payload also accepted). Unknown fields rejected; request ≤256 bytes. Returns:

```json
{"success":true,"data":{"articles":[{"id":"release-2026-09-15","title":"Release notes","summary":"Today's changes.","content":"Changes:\n- Improved news delivery","date":"2026-09-15","category":"Update","highlight":true,"updated_at":"2026-09-15T12:00:00Z"}],"refresh_after_seconds":60}}
```

Returns the latest **20 published** articles, ordered by `date DESC, id DESC` (stable tie-break). Empty result is `[]`. Full content is included for immediate article opening. No archive pagination, per-user variants or scheduling: `date` is editorial display/order metadata, even a future date publishes immediately when `published=true`. Highlight affects presentation, not ordering.

Errors: unauthenticated `16`, invalid request `3`, temporarily unavailable `14`. Client should reuse the response for 60 seconds, keep selection by stable ID, handle empty feed/loading/error, and avoid refreshing on every scene transition. C# DTOs need source-generated serialization metadata for iOS Native AOT.

## `admin_upsert_news`

Requires a signed-in user present in `news_administrators`. A server HTTP key alone is insufficient. Administrator authorization and upsert use one SQL statement. No client metadata, session variable or username grants privileges.

Full replacement request (all text fields required; omitted booleans default false):

```json
{
  "id":"release-2026-09-15",
  "title":"Release notes",
  "summary":"Today's changes.",
  "content":"Changes:\n- Improved news delivery",
  "date":"2026-09-15",
  "category":"Update",
  "highlight":true,
  "published":true
}
```

Response: `{"success":true,"data":{"id":"release-2026-09-15","published":true}}`.

Repeat the same ID to edit without duplicates. To withdraw, submit the full article with `published:false`. Rows remain for later editing/republication. Unknown fields, extra JSON values, null/non-object payload, invalid dates/categories, NUL and oversized text fail validation. Request ≤128 KiB accommodates escaped JSON. Concurrent administrator edits are last-write-wins; no draft listing/editor GUI/version history is implemented.

Errors: unauthenticated `16`, malformed `3`, missing/revoked role `7`, database failure `13`. SQL errors are not returned to clients.

## Operator procedure

1. Deploy migration `000009_news` and the new plugin through the existing release process. No runtime environment variables or new services are needed.
2. Grant an existing Nakama account the role from a trusted database session (replace the example UUID with the actual account ID):

```sql
INSERT INTO news_administrators(user_id)
VALUES ('00000000-0000-0000-0000-000000000001')
ON CONFLICT DO NOTHING;
```

Revoke with `DELETE FROM news_administrators WHERE user_id = '<actual-account-uuid>';`. The next write checks the database again; roles are never cached.

3. Sign in as that account, save the full JSON above to `article.json`, and call the RPC with its Nakama bearer session token. With `NAKAMA_URL` and `NAKAMA_SESSION_TOKEN` already set locally:

```sh
curl --fail-with-body --silent --show-error \
  -H "Authorization: Bearer ${NAKAMA_SESSION_TOKEN}" \
  -H 'Content-Type: application/json' \
  --data-binary @article.json \
  "${NAKAMA_URL}/v2/rpc/admin_upsert_news?unwrap=true"
```

Raw JSON HTTP requests use Nakama's documented [unwrap option](https://heroiclabs.com/docs/nakama/server-framework/runtime-examples/server-to-server/).

The existing Nakama SDK can also call `RpcAsync(session, "admin_upsert_news", json)`. Tokens must not be committed or pasted into logs. No role was granted to a real account in this task.

## Cache and database load

- Exactly one cached serialized JSON string per server process, independent of user or input; no unbounded cache keys.
- TTL 60 seconds; cache hits execute no SQL and do not rebuild JSON. Fixed feed size and byte-limited content bound response size, memory and database result allocation.
- Concurrent misses share a mutex-protected refresh. A 3-second database deadline bounds the refresh; caller disconnection does not cancel other readers' shared refresh.
- Failed refresh is cached for 5 seconds and returns Unavailable, not stale content. This avoids one database attempt per request during outages; retry at most once per 5 seconds per instance after a failed attempt completes.
- Successful administrative writes invalidate the local cache after commit. Invalidation uses the same lock as refresh, so an overlapping old load cannot remain cached after the write returns.
- Other instances, direct SQL edits and an instance crash between write and invalidation converge on the next TTL refresh (normally ≤60 seconds, plus query duration). No Redis or cross-node broadcast.
- Partial index `(article_date DESC,id DESC) WHERE published` supports the bounded feed query. Writes and permission checks have a 3-second context timeout. The cache is a database load control, not transport-level DDoS protection.

## Verification

`make test`, `go vet ./modules/news` and Linux plugin `make build` passed. Unit/race tests exercise 100 simultaneous readers, expiry, invalidation, failed-refresh retry suppression, strict parsing and validation. Isolated PostgreSQL 17 tests cover migration up/down, role denial/revocation, create/edit/withdraw, all article data, empty arrays, ordering/20-row limit, local invalidation and second-instance TTL refresh. No production migration, HTTP smoke test or client integration has been performed.

## Source of truth in code

- `server:modules/news/{types,validate,cache,db,rpc,init}.go`
- `server:modules/news/{news_test,db_test}.go`
- `server:modules/main.go`
- `server:db/migrations/000009_news.{up,down}.sql`
- `client:Game/ScenesV3/News/{NewsData,NewsScreen}.cs`
- `client:Game/ScenesV3/Dashboard/DashboardScreen.cs`
