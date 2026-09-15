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

The server implements persistent articles and administrator writes. Dashboard and NewsScreen now fetch `get_news` through the shared client NewsService/CQRS query. The hardcoded client articles were removed. Migration `000010_welcome_news` seeds one published, highlighted English announcement (`welcome-to-hexbane`) with brief onboarding advice. Obsolete sample announcements were not copied.

The seed uses `ON CONFLICT(id) DO NOTHING`, so replay does not overwrite an administrator edit. Its down migration removes that ID (including subsequent edits). Creator/editor account IDs are null for this system seed. Deploy migration9 before10; an existing installation at9 only needs the new seed migration.

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

Errors: unauthenticated `16`, invalid request `3`, temporarily unavailable `14`. Client behavior is implemented below; C# DTOs use source-generated serialization metadata for iOS Native AOT.

## Client integration

- `Core/News/NewsArticle` maps wire fields, provides read-only properties and invariant English display dates (`Sep 15, 2026`). `NewsFeedResponse` is registered in `ClientJsonContext`.
- DI singleton `NewsService` owns one immutable feed snapshot. Transient `GetNewsQueryHandler` delegates to it, so opening Dashboard and NewsScreen shares the cache. An async semaphore coalesces concurrent misses.
- TTL is the server's suggested interval clamped to60–300 seconds (current server sends60). Errors/invalid envelopes use a5-second backoff and no stale articles. Requests have a10-second cancellation deadline. No RPC without an authenticated session. Cache keys include user ID and client instance; late responses after session/client replacement are discarded.
- Both screens request on entry and show loading, empty or retryable failure states. Retry goes through the same backoff. There is no background timer: an already open screen refreshes on re-entry or error retry. After refresh, another client's cache can add up to60 seconds to the server cache's propagation delay.
- `NewsData` retains only `SelectedId`, not article data. Each scene renders its own snapshot; reordering preserves selection by ID, withdrawal chooses the first remaining article, empty feed clears selection. Scene exit invalidates pending UI updates.
- Plain text is escaped before existing BBCode heading/bullet formatting, so remote bracket text cannot inject links or formatting. Icons still derive locally. No bundled article fallback masks a missing server deployment.

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

1. Deploy migrations `000009_news`, `000010_welcome_news` and the new plugin through the existing release process. No runtime environment variables or new services are needed.
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

`make test`, `go vet ./modules/news` and Linux plugin `make build` passed. Unit/race tests exercise 100 simultaneous readers, expiry, invalidation, failed-refresh retry suppression, strict parsing and validation. Isolated PostgreSQL 17 tests cover migration up/down, role denial/revocation, create/edit/withdraw, all article data, empty arrays, ordering/20-row limit, local invalidation and second-instance TTL refresh. Follow-up verification (2026-09-15): migration10 up/replay/down passed on isolated PostgreSQL17; replay preserves edits. A disposable Nakama3.27 loaded the Linux plugin with all migrations and passed real HTTP authentication, seeded feed, denied player write, administrator publication/withdrawal and immediate local invalidation. Production client NewsService fetched the seed from that instance.

Client build passed (0 errors,11 existing warnings). The reflection-disabled JsonAot harness passed173 checks, including100 concurrent news readers/one RPC, cache expiry, empty results, malformed payload/backoff/recovery, unauthenticated access and late logout response. `NewsVerification` passed10 UI/service checks at1360×612 and960×540; rendered screenshots inspected for dashboard, reader, empty and error states. Verification uses actual live data for the welcome screens and injected service snapshots for reorder/empty/error cases. A single ObjectDB shutdown warning remains in the Godot harness. No production deployment or physical-device export/test was performed.

Repeatable client harness: start a disposable Nakama with migrations on localhost:57350, then run `HEXBANE_IGNORE_ENV_FILE=1 DEV_AUTO_LOGIN=false NAKAMA_HOST=127.0.0.1 NAKAMA_PORT=57350 NEWS_CAPTURE=/tmp/hexbane-news-captures <Godot .NET> --path . res://Game/ScenesV3/Dev/NewsVerification.tscn`. Optional `NEWS_SIZE=960x540`. The scene refuses any other host/port. Test account exists only inside the disposable database.

## Source of truth in code

- `server:modules/news/{types,validate,cache,db,rpc,init}.go`
- `server:modules/news/{news_test,db_test}.go`
- `server:modules/main.go`
- `server:db/migrations/000009_news.{up,down}.sql`
- `client:Game/ScenesV3/News/{NewsData,NewsScreen}.cs`
- `client:Game/ScenesV3/Dashboard/DashboardScreen.cs`

- `server:db/migrations/000010_welcome_news.{up,down}.sql`
- `client:Core/News/NewsArticle.cs`, `Application/Modules/News/`, `Application/Serialization/ClientJsonContext.cs`
- `client:Game/ScenesV3/News/NewsFeedUi.cs`, `Game/DI/ServiceBootstrapper.cs`
- `client:Tests/JsonAot/JsonAotVerification.cs`, `Game/ScenesV3/Dev/NewsVerification.cs`
