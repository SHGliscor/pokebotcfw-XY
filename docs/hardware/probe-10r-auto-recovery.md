# Probe 10R — automatic title/Continue recovery

Hardware run: `PokemonY_Probe10R_20261007_031100_846914`

Result: **PASS_BASELINE_RECOVERED**

## Identity

- Title ID: `0004000000055E00`
- Process: `kujira-2`
- PID: `49`
- Bridge: `Pokebot3DS-Luma-v0p6-n3ds-fb1`

## Automatic recovery proof

```
TITLE_READY
  ↓ A
CONTINUE
  ↓ A
SAVE_LOADED_AT_STARTER_POSITION
  ↓ RELEASE_ALL
STOP
```

Each visual state was required to match the hardware-derived scene classifier twice consecutively before progression.

### TITLE_READY

- consecutive matches: 2
- first A sequence: `840911386`
- terminal state: `COMPLETED`

### CONTINUE

- consecutive matches: 2
- second A sequence: `840911392`
- terminal state: `COMPLETED`

### SAVE_LOADED_AT_STARTER_POSITION

- consecutive matches: 2
- canonical baseline recovered: **true**
- post-load Party Slot 1: invalid/empty
- LEFT sent: **false**

## Safety

- buttons sent: exactly `A, A`
- no LEFT
- no starter-event progression
- recovery stopped on the saved Aquacorde overworld baseline
- controller returned neutral

## Task 4 gate

Together with the preceding Probe 10 hardware proof of the retained `L+R+START+SELECT` reset chord and real PID/process restart, this closes the one-shot reset/recovery hardware gate:

```
valid non-shiny Froakie
→ authorized retained reset
→ Pokemon Y process restart
→ TITLE_READY
→ CONTINUE
→ SAVE_LOADED_AT_STARTER_POSITION
```

The production state machine may now compose these already-proven primitives under the same fail-closed authorization rules.
