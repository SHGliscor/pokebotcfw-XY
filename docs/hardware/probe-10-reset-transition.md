# Probe 10 — one-shot reset transition

Hardware run: `PokemonY_Probe10_20261007_025211_824831`

Result: **PASS_RESET_TRANSITION_CAPTURED**

## Pre-reset authority

- title ID: `0004000000055E00`
- process: `kujira-2`
- PID before reset: `45`
- live decision: `WOULD_RESET`
- species: 656 (Froakie)
- TID/SID: 21080 / 3731
- checksum: `0xE1C3`
- PID: `0x92A35695`

## Reset controller proof

- retained HID latch: `0xCF3`
- chord: `L + R + START + SELECT`
- requested latch: 1.25 s
- observed latch duration: 1275.706 ms
- initial latch state: `IN_PROGRESS`
- release HID: `0xFFF`

## Reset effect

Fresh hardware observations proved all required effects:

- old Pokémon Y process disappeared
- Party Slot 1 ceased being a valid PK6
- Pokémon Y reappeared
- PID changed `45 → 47`
- first Y reappearance observed at about 2163.7 ms after reset observation began

The reset was therefore proven independently of controller acknowledgement.

## Post-reset capture

No progression input was sent after reset.

The framebuffer capture occurred during the Pokémon Y startup/opening animation, before the stable title/Continue state. Therefore:

- reset transition: **hardware_verified**
- title/Continue recovery: **not yet hardware_verified**
- canonical baseline recovered: false

The next mapping step is zero-bot-input Probe 04 from the post-reset title flow using one physical input per recorded transition.
