# Probe 10 recovery map — title to canonical baseline

Hardware mapper run: `PokemonY_Probe04_20261007_030152_899176`

## Fresh recovery sequence

```
TITLE_READY / PRESS START
  ↓ physical A
CONTINUE
  ↓ physical A
SAVE_LOADED_AT_STARTER_POSITION
  ↓ physical LEFT
FRIEND_DIALOGUE_01
```

Probe 04 sent no bot controller input.

The mapper captured four states and three manual transitions. Party Slot 1 remained invalid/empty throughout, and the party-window bytes were identical across the four captures.

## Canonical baseline proof

State 002 is the saved Aquacorde overworld position. It also matches the earlier Probe 03 pre-LEFT scene within normal framebuffer variation. State 003 freshly proves that one LEFT from state 002 enters the starter event.

## Recovery automation contract

The recovery-only validation probe may:

1. recognize TITLE_READY,
2. press A once,
3. recognize CONTINUE,
4. press A once,
5. recognize SAVE_LOADED_AT_STARTER_POSITION,
6. release all and STOP.

It must not send LEFT.

Visual matching is tolerant rather than exact-hash based. A derived low-resolution scene signature is used so ordinary framebuffer variation does not reject the correct state. Two consecutive observations of each target state are required before progression.
