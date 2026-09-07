---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, story-update]
sources: ["client:docs/opcodes/op_100_story_update.md", "server:docs/opcodes/op_100_story_update.md"]
---

# Opcode 100 — StoryUpdate (Endless Story, separate match type)

| | |
|---|---|
| Server const / client enum | `OpStoryUpdate` (`server:modules/endless_story/create_character/types.go:31`) / `ENDLESS_STORY_UPDATE` |
| Direction | Server → client, unicast, reliable |
| Match | `v2_create_character` (1 tick/s), not the duel handlers |
| Sender | `server:modules/endless_story/create_character/phase_entry_message.go:61` |
| Client handler | `client:Application/Match/Incoming/EndlessStory/StoryUpdate/StoryUpdateHandler.cs` → `GameEvents.NarrationUpdated` |

The story module is still registered at boot (`server:modules/main.go:109` → `endless_story.InitModule` → `create_character.InitModule`). It is a work-in-progress prototype that nothing in the current game flow starts; the client's `Application/EndlessStory/CreateCharacter/MatchManager.cs` calls RPC `create_character_match_story`, which creates module name `create_character` while the handler is registered as `v2_create_character` (see report).

## When

Once, on the first tick of `EntryMessageState` after the single presence joined. The payload is a static placeholder prompt.

## Payload

```json
{"narration":"string","question":{"question":"string","choices":[{"id":"string","display":"string"}]}}
```

Client DTO: `StoryUpdateMessage` (`narration`, `question: QuestionDto`).

## Source of truth in code
- `server:modules/endless_story/create_character/phase_entry_message.go` — sender and phase
- `server:modules/endless_story/create_character/init.go` — match registration and RPC
- `client:Application/Match/Incoming/EndlessStory/StoryUpdate/StoryUpdateMessage.cs` — DTO
