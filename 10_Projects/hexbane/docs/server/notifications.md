---
type: project
project: Hexbane
area: server
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-08
verified: 2026-09-08
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

**No `notifications_send` RPC exists.** Cleanup on 2026-09-08 removed the six uncalled
`Send*Notification` helpers, their unused send/event DTOs, unused code constants, misleading
startup logs and `CreateNotificationContent` (an example stub that returned only message text).
The list/delete handlers and their response DTOs remain unchanged.

The live custom notification in `social.RpcRemoveFriend` still calls `nk.NotificationSend`
directly with subject `friends` and code 1. No match-result, level-up or custom system-message
sender was activated. Nakama's built-in notifications remain available. Numeric payload codes
were not remapped; the unused Go declarations were removed.

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
- `server:modules/notifications/init.go`, `rpc.go`, `types.go` — retained list/delete RPCs and response DTOs.
- `server:modules/social/rpc.go` — the only custom notification actually sent.
- `client:Application/Nakama/NakamaClientManager.cs`, `client:Application/Modules/Notifications/**` — client delivery and SDK calls.
