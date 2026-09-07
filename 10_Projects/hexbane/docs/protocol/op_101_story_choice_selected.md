---
type: project
project: Hexbane
area: protocol
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, protocol, opcode, story-choice-selected]
sources: ["client:docs/opcodes/op_101_story_choice_selected.md", "server:docs/opcodes/op_101_story_choice_selected.md"]
---

# Opcode 101 — StoryChoiceSelected (Endless Story, separate match type)

| | |
|---|---|
| Server const / client enum | `OpStoryChoiceSelected` (`server:modules/endless_story/create_character/types.go:32`) / `ENDLESS_STORY_CHOICE` |
| Direction | Client → server |
| Match | `v2_create_character` |
| Server handler | `server:modules/endless_story/create_character/phase_entry_message.go:72-84` |
| Client sender | `client:Application/Match/Outgoing/EndlessStory/ChoiceSelected/ChoiceSelectedCommandHandler.cs` |

## Payload

```json
{"choice_id":"string"}
```

Server struct `story_model.ChoiceSelected` (`server:modules/endless_story/story_model/question.go:26`). The client additionally serializes `MatchId` (no `JsonPropertyName`, so as `"MatchId"`), which the server ignores.

## Server behaviour

Parses and logs the choice; nothing else happens (the transition to `CreateCharacterState` is commented out, `phase_entry_message.go:67-69`).

## Source of truth in code
- `server:modules/endless_story/create_character/phase_entry_message.go` — handler
- `client:Application/Match/Outgoing/EndlessStory/ChoiceSelected/ChoiceSelectedCommand.cs` — payload
