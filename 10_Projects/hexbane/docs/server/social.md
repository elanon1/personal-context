---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-17
verified: 2026-09-17
tags: [hexbane, server, social, friends, nakama]
sources: ["server:modules/social/README.md", "client:docs/Server/Social.md", "server:docs/API-REFERENCE-v2.md"]
---

# Social module (friends)

Friends are Nakama's built-in friend graph (`user_edge`); the server plugin adds `find_friend`, `remove_friend` and, since 2026-09-17, the duel-invitation family (`friend_status`, `duel_invite`, `duel_invites`, `duel_invite_reply`, `push_device`; see below). Chat and
status/presence are **not implemented** anywhere: the `social_*` RPC family and every chat/status helper
in `server:modules/social/rpc.go` are commented out (`server:modules/social/init.go:22-104`), and the client
never calls `FollowUsersAsync`/`JoinChatAsync`.

## Server RPCs (`server:modules/social/init.go:12-20`)

### `find_friend` (`rpc.go:11`)
- `{"name":"Merlin"}` → exact match on `characters.name` **or** `users.username` (`db.go:10-38`).
- OK: `{"success":true,"message":"Friend Found!","data":{"character_id","user_id","name","username"}}`.
- Fail (`"data":{}`): `Player not found`, `Cannot find yourself`, `Failed to find friend`, `Invalid request format`, `Authentication required`.
- No block/privacy check (`db.go:35` todo).

### `remove_friend` (`rpc.go:308`)
- `{"user_ids":["…"],"usernames":["…"]}`, at least one non-empty (`types.go:132-135`).
- Calls `nk.FriendsDelete(userID, username, user_ids, usernames)` — removes friends **or unblocks**.
- Then, for each id in `user_ids` only (not `usernames`), sends a persistent notification with
  **code 1**, subject `"friends"`, content `{"title":"Friend Removed","message":"Your friend has removed you from their friends list"}`, sender = caller (`rpc.go:337-348`).
- Response `{"success":true,"message":"Friend Removed","data":{"success":true,"message":"Friend removed successfully"}}`.

## Client usage (`client:Application/Modules/Social/`)

| Action | Implementation |
|---|---|
| Search | RPC `find_friend` (`Queries/FindFriend/FindFriendQueryHandler.cs:42-52`); `success:false` → `FriendNotFoundException`, swallowed by `SocialService.FindFriend` → `null`. |
| Add | SDK `client.AddFriendsAsync(session, userIds, usernames)` (`Commands/AddFriend/AddFriendCommandHandler.cs:36`). Nakama auto-accepts when both sides add. |
| List | SDK `client.ListFriendsAsync(session, state, 100, cursor)` (`Queries/GetFriends/GetFriendsQueryHandler.cs:43`). |
| Block | SDK `client.BlockFriendsAsync` (`Commands/BlockUser/BlockUserCommandHandler.cs:36`, not awaited). |
| Remove | RPC `remove_friend` (`Commands/RemoveFriend/RemoveFriendCommandHandler.cs:36-42`). |
| In-match flag | RPC `friend_status` merged into `FriendDto.InMatch` by `GetFriendsQueryHandler` (state 0 only). |
| Invite / inbox / reply | RPCs `duel_invite`, `duel_invites`, `duel_invite_reply` via `SocialService` (`Services/SocialService.cs`); inbox polled by `client:Game/Autoloads/DuelInvitationInbox.cs`. |
| Push token | RPC `push_device` from `client:Game/Autoloads/MobilePushRegistration.cs` (mobile only). |
| `get_users` | `Queries/GetUsers/GetUsersQueryHandler.cs:44` calls an RPC the server does not register; nothing dispatches this query. Dead. |

Friend states (Nakama): `0` friends, `1` invite sent, `2` invite received, `3` blocked (`server:modules/social/types.go:8-13`).
Nakama itself emits the friend notifications: code `-2` (request), `-3` (accepted); the plugin's
`SendFriendRequestNotification`/`SendFriendAcceptedNotification` helpers are unused (see [[notifications]]).

## Dropped from the old docs
`social_set_status`, `social_follow_users`, `social_unfollow_users`, `social_search_users`,
`social_add_friend`, `social_block_user`, `social_remove_friend`, `social_list_friends`,
`social_join_chat`, `social_send_chat_message`, `social_leave_chat`, `social_list_chat_messages`,
`social_list_channel_users` — all unregistered. Status via account metadata: not implemented.

## Match-authorized opponent actions (2026-09-12)

Authenticated `match_opponent_action {match_id,action:"add_friend"|"report"}` resolves the opponent only from the caller’s immutable completed result receipt. It accepts no client-provided opponent identity. Success: `{success:true,data:{match_id,action,opponent_id,status}}`; add_friend has common `pending` status, report `reported`. No opponent kind or bot flag is serialized. Missing/foreign/historical receipt without the opponent payload → NotFound(5), malformed→3, unauthenticated→16, internal→13.

Migration8 `match_opponent_actions` stores one action per `(match_id,actor_user_id,action)` with internal human/persona attribution. Real-account friend requests invoke Nakama FriendsAdd; persona requests remain locally pending and never fabricate acceptance or online activity. Reports for either type are recorded internally. There is no moderation dashboard/notification delivery for these report rows yet. Persona pending requests do not appear in the current SDK-backed Friends list; a unified list/profile surface remains a product follow-up, not proof of indistinguishability.

Concurrent retries serialize under a transaction advisory lock. Once committed they return the saved response without repeating FriendsAdd. A DB commit failure after successful Nakama FriendsAdd can cause that idempotent operation to be invoked again; external delivery is not transactionally atomic with the action row. GameOver sends both actions through this RPC and shows Invite sent/Report sent. Current lobby/result UI has no opponent profile link; training still hides opponent actions.

## Source of truth in code
- `server:modules/social/init.go` — the two registered RPCs (and the commented-out rest).
- `server:modules/social/rpc.go`, `db.go`, `types.go` — handlers, name lookup, request/response structs.
- `client:Application/Modules/Social/Services/SocialService.cs` — the operations the UI actually uses.
- `client:Application/Modules/Social/Commands/*`, `Queries/*` — SDK vs RPC per operation.


## Friend duel invitations and push (HEX-30, 2026-09-17)

Source: `server:modules/social/invitations.go` (RPCs, registered from `init.go` via `registerInvitations`), `server:modules/social/push.go` (delivery), migration `000013_friend_invites` ([[database]]). Exact request/response contract: [[rpcs]] → *Multi-character and friend duel invitations*.

**Model.** `friend_duel_invites(invite_id UUID PK, sender_id, recipient_id → users CASCADE, state pending|accepted|declined, match_id, created_at, expires_at = created + 2 min)`. An invitation is server-owned: the client only names a friend, never a match or an opponent identity. Creation and acceptance lock both `users` rows in sorted id order (the same order character selection and queue admission use), require `user_edge.state = 0` in both directions and a **selected** character on both sides that holds no active `character_match_locks` row and no live `fallback_queue` lease. Senders are rate-limited to 5 invitations per minute. Acceptance creates the `normal` match with `friend_users` (see [[matchmaking]]) and reserves both characters for 2 minutes so a stray queue join cannot steal either mage before both clients arrive; the normal acquisition path extends those locks when the players join.

**Delivery.** Three channels, all best effort except the table itself:
1. Nakama persistent notifications, code **20** `Duel invitation` (`{invite_id,sender_name,expires_at}`, sender = inviter) and **21** `Duel ready` (`{invite_id,match_id}`, both players). The client does not handle these codes yet; they are there for a future socket-driven inbox.
2. The inbox RPC `duel_invites`, polled by the client every 5 s — the source of truth that survives offline recipients and missed sockets.
3. Provider push for minimised phones, `sendInvitePush`: up to 10 devices per recipient from `push_devices` refreshed within 90 days. Android → FCM HTTP v1 with a service-account JWT (`HEXBANE_FCM_SERVICE_ACCOUNT` = path to the JSON key), high priority, channel `duel_invites`, TTL 120 s. iOS → APNs with an ES256 provider token (`HEXBANE_APNS_KEY_FILE` P-256 `.p8`, `HEXBANE_APNS_KEY_ID`, `HEXBANE_APNS_TEAM_ID`, `HEXBANE_APNS_TOPIC`, `HEXBANE_APNS_SANDBOX=true` for the sandbox host), expiry = invitation expiry. A platform whose variables are unset is skipped silently; `UNREGISTERED` / `BadDeviceToken` / 410 delete the token, other failures only log. **No credentials are configured anywhere yet** (local compose, Helm, GitOps), so push is dormant until an operator mounts them.

**Verified 2026-09-17** over HTTP on the rebuilt local stack (`server:scripts/e2e_invitations.py`, two seeded accounts): invite → recipient inbox `pending`; decline → hidden from inbox, second reply 400; re-invite → accept creates match + locks, replay of accept returns the same match; sender replying → 403; unknown id → 404; invite while locked → 9; `friend_status` shows the busy friend; the recipient's notification list contains codes 20, 20, 21. Not exercised: two live Godot clients joining the created match on phones, real FCM/APNs delivery.
