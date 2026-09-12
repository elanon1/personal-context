---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-12
verified: 2026-09-12
tags: [hexbane, server, social, friends, nakama]
sources: ["server:modules/social/README.md", "client:docs/Server/Social.md", "server:docs/API-REFERENCE-v2.md"]
---

# Social module (friends)

Friends are Nakama's built-in friend graph (`user_edge`); the server plugin adds only two RPCs. Chat and
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
