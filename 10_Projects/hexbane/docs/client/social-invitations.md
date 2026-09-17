---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-17
updated: 2026-09-17
verified: 2026-09-17
tags: [hexbane, client, social, invitations, push]
sources: ["client:Game/ScenesV3/Social/SocialScreen.cs", "client:Application/Modules/Social/", "client:Game/Autoloads/DuelInvitationInbox.cs", "client:Game/Autoloads/MobilePushRegistration.cs", "client:addons/HexbanePush/"]
---

# Social screen, duel invitations and push (HEX-30/31)

Server contract: [[social]] and [[rpcs]] → *Multi-character and friend duel invitations*.

## Social screen (`SocialScreen.cs`)
- **No mock data any more** (HEX-31): friends and requests come only from Nakama (`ListFriendsAsync`), an empty account shows empty tabs, and a failed refresh shows *Could not refresh friends. Retrying shortly.* Race and level rows are gone because there is no friend profile endpoint; avatars come from `AvatarUrl` or stay empty.
- Presence: `Online` = Nakama `user.online`; **In match** = `friend_status` (merged into `FriendDto.InMatch`). The list refreshes every 10 s and on friend events.
- **INVITE / INVITE TO GAME** calls `ISocialService.Invite(userId)` → `duel_invite`; disabled while the friend is in a match (offline friends can be invited — the invitation waits in their inbox for 2 minutes and may reach them by push). Feedback: *Invitation sent to X. Valid for 2 minutes.* or the server error.

## Invitation inbox (`Game/Autoloads/DuelInvitationInbox.cs`)
Child of `NotificationManager`, alive on every screen. Every 5 s, while authenticated and connected, it calls `duel_invites` (after refreshing the push token) and:
- shows a `ConfirmationDialog` *"<name> invited you to a duel. Expires at hh:mm."* with **Accept / Decline** for the newest pending invitation addressed to this account (one dialog at a time, each invitation shown once per session, only while the match state is Idle / MatchEnded / Error);
- on Accept → `duel_invite_reply` → joins the returned `match_id` through `MatchManager.JoinInvitation` (the `ad_normal` keyed manager), which binds the socket, sets `MatchContext.MatchId`, joins the Nakama match and moves the status to Connecting so the normal lobby/duel flow takes over;
- for the **inviter**, the same poll notices the invitation turning `accepted` and joins the same match — no socket notification is required. Sessions are tracked per account so a character/account switch resets the seen set.
Errors surface in an `AcceptDialog`. The Nakama notifications 20/21 the server also sends are not consumed by the client yet.

## Push tokens (`MobilePushRegistration.cs`, `addons/HexbanePush/`)
- Android: Godot plugin `com.hexbane.push.HexbanePush` (`registerPush` creates the `duel_invites` channel, asks `POST_NOTIFICATIONS` on API 33+, fetches the FCM token; `PushService` stores it and receives `onNewToken`). Background FCM *notification* messages are displayed by Android itself; foreground relies on the inbox.
- iOS: `HexbanePush.xcframework` built by `addons/HexbanePush/ios/build_push.sh` from `HexbanePush.mm` (swizzles `didRegisterForRemoteNotificationsWithDeviceToken`, requests alert/sound/badge, exposes `hexbane_push_start` / `hexbane_push_token`), loaded with `DllImport("Frameworks/HexbanePush.framework/HexbanePush")`.
- The token is uploaded once per token+account with `push_device`; desktop never calls it.
- **Off by default.** The export plugin adds the AAR, `firebase-messaging:24.1.0` and the iOS frameworks only when the preset option `hexbane_push/enabled` is true; the Android AAR must first be built from `addons/HexbanePush/android` (gradle, needs a `google-services`/Firebase project) and the xcframework is git-ignored (`ios/*.xcframework/`), so rebuild it before an iOS export. The server needs the FCM/APNs variables from [[social]]; none are configured today. Enabling push is therefore an operator task: Firebase project + service account, APNs key, preset flag, rebuilds.

## Verification (2026-09-17)
Server flow verified over HTTP (see [[social]]). Client: build 0 errors, `check_aot_json.sh` OK (`SocialJsonContext` covers every invitation DTO). **Not yet done:** two Godot clients accepting an invitation and landing in the same lobby, the inbox dialog on a phone, and any real push delivery.
