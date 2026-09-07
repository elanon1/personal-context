---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, server, notifications, nakama]
sources: ["server:modules/notifications/README.md", "client:docs/Server/Notifications.md", "server:docs/API-REFERENCE-v2.md"]
---

# Notifications module

Thin wrapper over Nakama's built-in notification system. No custom tables.

## RPCs (`server:modules/notifications/init.go:15-23`)

| RPC | Handler | Request | Response |
|---|---|---|---|
| `notifications_list` | `rpc.go:13` | `{"limit":20,"cacheable_cursor":""}`; limit ≤0 or >100 → 10 | `{"success":true,"message":"…","notifications":[{"id","subject","content","code","sender_id","create_time","persistent"}],"cacheable_cursor":"…"}` |
| `notifications_delete` | `rpc.go:65` | `{"notification_ids":["…"]}` (non-empty) | `{"success":true,"message":"Notifications deleted successfully"}`; ids are scoped to the caller |

Errors: `{"success":false,"error":"request_failed","message":"…"}`. `create_time` is formatted
`2006-01-02T15:04:05Z` (UTC, no fractional seconds). `content` is the JSON string Nakama stores.

**No `notifications_send` RPC exists** — the README still documents one, but `init.go:12-13` states
notifications are dispatched server-side only. `SendNotificationRequest` in `types.go:52-59` is dead.

## Codes (`server:modules/notifications/types.go:5-27`)

| Code | Meaning | Emitted by |
|---|---|---|
| −1 … −8 | Nakama built-ins (offline message, friend request −2, friend accepted −3, group −4/−5, friend online −6, socket closed −7, banned −8) | Nakama core |
| 1 | `NotificationCodeMatchInvite` — **actually used by `remove_friend`** as a generic "friends" notification (`server:modules/social/rpc.go:344`) | social module |
| 2 | Match result | nobody |
| 3 | Achievement | nobody |
| 4 | Level up | nobody (see [[level-up-notifications]]) |
| 5 | Spell unlocked | nobody |
| 6 / 7 | Tournament start/end | nobody |
| 8 | Leaderboard rank | nobody |
| 9 | System message | nobody |
| 10 | Challenge received | nobody |

Helper functions `SendFriendRequestNotification`, `SendFriendAcceptedNotification`,
`SendMatchInviteNotification`, `SendMatchResultNotification`, `SendSystemNotification`,
`SendFriendOnlineNotification` (`rpc.go:112-240`) exist but have **no callers** outside commented code.
The game-over phase does not send notifications (it has no `NakamaModule`).

## Client (`client:Application/Modules/Notifications/`)

- Real-time: `NakamaClientManager.SetupSocket` subscribes `socket.ReceivedNotification`
  (`client:Application/Nakama/NakamaClientManager.cs:68`) and `NotificationModule` listens after
  `ConnectionEstablished` (`client:Game/Autoloads/NotificationManager.cs`).
- List / delete use the SDK directly: `client.ListNotificationsAsync` (`Queries/ListNotifications/ListNotificationsHandler.cs:37`)
  and `client.DeleteNotificationsAsync` (`Commands/DeleteNotifications/DeleteNotificationsCommandHandler.cs:35`).
  The two custom RPCs are therefore unused by the client (comments in the query/command files still claim
  they map to them).
- Client code enum `NotificationType` (`Models/NotificationType.cs`): Nakama negatives, `SimpleNotification = 1`, `LevelUp = 4`.

## Source of truth in code
- `server:modules/notifications/init.go`, `rpc.go`, `types.go` — RPCs, codes, unused helpers.
- `server:modules/social/rpc.go:337-348` — the only custom notification actually sent.
- `client:Application/Nakama/NakamaClientManager.cs`, `client:Application/Modules/Notifications/**` — client delivery and SDK calls.
