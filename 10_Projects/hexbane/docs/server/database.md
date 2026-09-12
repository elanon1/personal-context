---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, server, database, migrations, postgres]
sources: ["server:docs/spell_system/database-v2.md", "server:docs/progression/progression.md", "client:docs/Server/progression/races_seed.sql"]
---

# Database

PostgreSQL shared with Nakama 3.27. Nakama owns its own tables and migration history (`users`, accounts, storage, leaderboards, notifications, etc.) and must migrate first; the application schema references `users(id)`. Application migrations live in `server:db/migrations/` and run with golang-migrate (`make migrate-up`, `server:Makefile:169-175`). There is **no `spells` table**: the catalog is YAML only ([[spell-system]]).

Startup order everywhere is Nakama migrations → application migrations → backend: Compose entrypoint (`server:docker-compose.yml:41-43`), Helm init containers `migrate-nakama` and `migrate-custom` (`server:helm/hexbane/templates/deployment.yaml:30-42`), and `scripts/reset_dev_db.sh:30-35`.

## Migrations

| Version | Up | Down |
|---|---|---|
| `000001_initial_schema` | `races`, `characters`, `character_spells`, `playstyles`, `playstyle_slots`, `updated_at` trigger | drops all five tables and the function |
| `000002_reference_data` | inserts the six races (values in [[progression]]) | deletes characters of those races (cascading spells/playstyles), then the races; Nakama accounts stay |
| `000003_local_tutorial` | `account_tutorials`; backfills a row per existing character with training/reward done and `progression_completed = tutorial_completed` | drops the table |

## Tables (`server:db/migrations/000001_initial_schema.up.sql`, `000003_local_tutorial.up.sql`)

### `races`
`race_id VARCHAR(50) PK`, `name UNIQUE`, `str_modifier`/`int_modifier`/`dex_modifier INT`, `casting_time_modifier REAL`, `spell_resistances JSONB '{}'`, `primary_element`/`secondary_element VARCHAR(50)`, `primary_element_bonus`/`secondary_element_bonus REAL`, `min_str`/`max_str`/`min_int`/`max_int`/`min_dex`/`max_dex INT ≥ 0` (0 = no limit, CHECK `max = 0 OR min <= max`), `traits JSONB '{}'`. Only `traits.spell_slot_bonus` affects play; the rest is creation/menu metadata.

### `characters`
- Identity: `character_id VARCHAR(255) PK` (format `char_<user_id>_<unix>`, `server:modules/character/db.go:24`), `user_id UUID UNIQUE → users(id) ON DELETE CASCADE` (one character per account), `name VARCHAR(50)`, `avatar`, `race_id → races DEFAULT 'human'`.
- Progression: `level ≥ 1 DEFAULT 1`, `experience ≥ 0`, `unspent_stat_points ≥ 0`, `magic_points`, `magic_points_spent` with CHECK `0 ≤ spent ≤ magic_points`, `spell_slots INT DEFAULT 3 CHECK BETWEEN 3 AND 7`.
- Stats: `base_strength/…` DEFAULT 133/134/133 and effective `strength/intelligence/dexterity` DEFAULT 153/154/153 (all ≥ 1). Both are persisted; effective is recomputed by the server.
- Skills: `skill_meditation`, `skill_spell_resistance`, `skill_magery REAL CHECK 0–100`.
- Record: `wins`, `losses ≥ 0`, `last_first_win_date DATE`, `last_match_date TIMESTAMPTZ`, `tutorial_completed BOOL`.
- `created_at`, `updated_at` (trigger `update_characters_updated_at`). Indexes on `wins DESC` and `race_id`.
- Duel HP/mana are runtime constants (200/100), not columns.

### `character_spells`
`(character_id → characters CASCADE, spell_id VARCHAR(50)) PK`, `learned_at`, `learned_at_level ≥ 1`. Index on `spell_id`. Starters chosen at creation are inserted in the same transaction as the character (`server:modules/character/db.go:49-75`); purchases insert with `ON CONFLICT DO NOTHING` and debit MP in one transaction (`db.go:248-293`). Standard spells never have rows.

### `playstyles` / `playstyle_slots`
`playstyles(playstyle_id PK, character_id → characters CASCADE, name TEXT non-empty, created_at)`; `playstyle_slots(playstyle_id → playstyles CASCADE, slot_number 1–7, spell_id NULL = empty) PK (playstyle_id, slot_number), UNIQUE (playstyle_id, spell_id)`. Server validation before every write, inside a transaction that locks the character row: slot number ≤ the character's `spell_slots`, spell exists in the catalog and is not standard, no duplicate spell, spell owned, selected count ≤ entitlement (`server:modules/playstyle/db.go:84-130`).

### `account_tutorials`
`user_id UUID PK → users(id) CASCADE`, `training_completed`, `reward_claimed`, `progression_completed BOOL`, `reward_receipt JSONB '{}'`, `updated_at`. Account-scoped so training can precede character creation; rows are upserted by the `tutorial` RPC (`server:modules/character/tutorial.go:47`). `complete_progression` also sets `characters.tutorial_completed = TRUE`.

Spell ids in `character_spells` and `playstyle_slots` intentionally have no foreign key (the catalog is not in SQL); raw SQL can insert invalid ids, so writes must go through server validation.

## Test accounts on the cluster (2026-09-09)

After the authorized reset, the cluster had no `@test.pl` accounts. Ran `NAKAMA_URL=https://hexbane.elanon.pl bash scripts/seed_dev_accounts.sh` at the user's request. All six accounts now exist: `{human,elf,dark_elf,shadow,gnome,orc}@test.pl`, password `123123123`, each with its corresponding level-1 character. Human owns four starters; the other five own three each (19 ownership rows total). A second run successfully authenticated each account and detected existing characters without duplicates.

**Latest state (same day):** at the user's request, removed all six seeded characters and their cascading spell/loadout data, preserving all six login accounts/passwords. Scoped tutorial reset matched zero rows. Stopped/restarted the backend around deletion to clear in-memory match state. Each test account now has zero characters and can create one through the client.

Seeding is **manual**: no CI, Docker startup or Helm hook calls this script. Normal restarts/deployments retain accounts in PostgreSQL; another full database reset requires rerunning the seed command. This verification covers seeding and login/character retrieval only; schema descriptions elsewhere in this older note were not re-audited here. See [[infra-and-deploy]] for current migration version 5 and deployment evidence.

## Development commands (`server:Makefile`)

| Command | Effect |
|---|---|
| `make db-reset` | `scripts/reset_dev_db.sh`: stops nakama, drops and recreates only `DB_NAME` (refuses `postgres`/`template*`), runs Nakama migrations then application migrations via one-off containers, restarts nakama. Does not remove volumes. Build the image first (`make compose-build`). Overrides: `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DOCKER_COMPOSE`. |
| `make db-seed` | `scripts/seed_dev_accounts.sh`: creates `{human,elf,dark_elf,shadow,gnome,orc}@test.pl` (password `123123123`, username `<race>_test`) through Nakama email auth and `create_character`; repeatable. Starters `firebolt, mend, poison` (+`cleanse` for Human), so a fresh seed yields 19 ownership rows. Overrides: `NAKAMA_URL`, `NAKAMA_SERVER_KEY`, `DEV_ACCOUNT_PASSWORD`. |
| `make db-clear-characters` | `scripts/clear_dev_characters.sh`: stops nakama if running, deletes all `characters` (cascades spells/playstyles), resets every `account_tutorials` row to false/`{}`, restarts nakama if it was running. Accounts, races and migration history survive. |
| `make migrate-up` / `make migrate-down` | apply all pending / roll back one application migration against `DB_URL` (defaults `postgres:localdb@localhost:5432/nakama`). |
| `make test-db` | `scripts/test_db_schema.sh` against a disposable `TEST_DB_URL`. |
| `scripts/test_db_api.sh` | API smoke test; refuses to run unless `TEST_DB_CONTAINER=hexbane-schema-test`. |

Recorded verification (2026-09-06, from the previous database doc, not re-run on 2026-09-07): fresh startup, idempotent up, full down/up preserving Nakama users, six races, FK and MP CHECK rejection, two reset runs, repeatable seed, MP purchase/duplicate/insufficient cases, seventh-slot entitlement and playstyle rollback on an injected constraint failure; integration tests run with `go test -mod=mod -tags=integration ./modules/playstyle`.

## Fallback migrations (2026-09-12)

Versions 4/5 already contain the current progression redesign, `character_match_locks` and idempotent `character_match_rewards`; fallback does not replace them. Version6 adds `fallback_assignments`, `fallback_queue`, `fallback_queue_rate`, `fallback_queue_requests`, indexes and state/identity constraints. Version7 adds immutable `bot_personas`, unique active `bot_persona_leases` and `bot_persona_history`. Persona records reference races, not Nakama users; human IDs belong to queue participants. See [[fallback-opponents]]. The original versions1–3 table description above is historical and not a complete current schema inventory.

## Match actions and result receipts (2026-09-12)

Migration8 adds `match_opponent_actions` (actor FK to users, UUID target with internal kind, action/status CHECKs, match+actor+action PK). The existing `character_match_rewards.result` now optionally includes exact `match_result` opcode50 JSON, committed with the reward. No extra result ledger or reward migration. Historical receipts without that field cannot be recovered from current character state. See [[op_50_game_over]] and [[rpcs]].

## Source of truth in code

- `server:db/migrations/000001_initial_schema.up.sql` — tables, constraints, trigger
- `server:db/migrations/000002_reference_data.up.sql` — race rows
- `server:db/migrations/000003_local_tutorial.up.sql` — `account_tutorials`
- `server:modules/character/db.go`, `server:modules/playstyle/db.go`, `server:modules/character/tutorial.go` — every write path and its validation
- `server:scripts/reset_dev_db.sh`, `seed_dev_accounts.sh`, `clear_dev_characters.sh`, `server:Makefile` — development database operations
- `server:docker-compose.yml`, `server:helm/hexbane/templates/deployment.yaml` — migration ordering at startup
