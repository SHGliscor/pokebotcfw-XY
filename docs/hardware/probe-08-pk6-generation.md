# Probe 08 — earliest valid Froakie PK6 generation

Hardware run: `PokemonY_Probe08_20261007_022733_874152`

Result: **PASS_VALID_FROAKIE_GENERATED**

## Identity

- Title ID: `0004000000055E00`
- Process: `kujira-2`
- Bridge: `Pokebot3DS-Luma-v0p6-n3ds-fb1`

## Confirmation and generation

Probe 08 verified the proven YES/NO confirmation screen, sent exactly one acknowledged `A`, then polled hardware-verified Party Slot 1.

- poll interval: 20 ms requested
- timeout: 8.0 s
- total samples: 143
- invalid/intermediate samples: 142
- empty samples: 0
- first checksum-valid authoritative Froakie: **6212.349 ms** after confirmation

The invalid samples were never given shiny authority.

## Generated PK6

- species: 656 (Froakie)
- TID/SID: 21080 / 3731
- checksum: `0xE1C3`
- EC: `0xEA0A7443`
- PID: `0x92A35695`
- nature: 5
- ability: 67
- IVs: 10 / 29 / 29 / 15 / 8 / 31
- moves: 1 / 45 / 145 / 0
- shiny: no

An independent re-decode of the captured raw slot returned the same valid PK6.

## Safety

- controller action: exactly one `A`
- terminal HID: neutral `0xFFF`
- reset sent: false
- further input sent: false

## Gate result

The generation poller is hardware-proven to ignore invalid/intermediate Party Slot 1 bytes and accept only a checksum-valid matching Froakie.

The observed non-shiny result is suitable for Probe 09 decision dry-run only; no automatic reset is authorized by Probe 08.
