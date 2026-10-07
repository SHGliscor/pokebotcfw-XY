# Pokémon Y reference-probe hardware evidence

Date: 2026-10-07  
Game: Pokémon Y (English)  
Title ID: `0004000000055E00`  
Process: `kujira-2`  
Bridge: `Pokebot3DS-Luma-v0p6-n3ds-fb1`

## Firmware identity gate

Probe 00 passed after installing the unified X/Y + OR/AS firmware.

- GAME_INFO Title ID: `0004000000055E00`
- Process: `kujira-2`
- No controller input was sent.

## Trainer address proof

Candidate address: `0x08C79C3C`

Repeated read:

- TID: `21080`
- SID: `3731`

Independent proof: the same TID/SID values were decoded from the checksum-valid manually received Froakie PK6 in Party Slot 1.

Status: **hardware_verified**

## Party Slot 1 proof

Candidate base: `0x08CE1CF8`

Probe 01 observed the pre-generation/invalid slot state.

Probe 02 later observed a valid PK6 at the same address:

- Species: `656` (Froakie)
- EC: `0x4C876A10`
- PID: `0x3EC89F1F`
- Stored checksum: `0x4828`
- Recalculated checksum: `0x4828`
- TID: `21080`
- SID: `3731`
- Ability: `67`
- Nature: `4`
- Moves: `1, 45, 145, 0`
- IVs: `0, 3, 0, 3, 8, 20`
- Shiny: **false**

Status: **hardware_verified**

## RNG addresses

The corrected v1.1 probe observed:

- MT index: `517`
- MT start word at `0x08C5284C`: `0x60B87BDA`
- Indexed current MT word: `0x80119718`
- TinyMT state:
  - `0x6A28A7BE`
  - `0x7938BABA`
  - `0x7489B30B`
  - `0xD9C57930`

These remain **candidate-only**. Stability alone does not promote RNG semantics to hardware authority.

## Gate result

Plan 1 hardware proof is sufficient to promote:

- `0x08C79C3C` TID/SID → **hardware_verified**
- `0x08CE1CF8` Party Slot 1 base → **hardware_verified**

No controller-driven starter automation is authorized by this evidence alone. The next stage is the controlled LEFT-trigger/state-atlas work from the canonical pre-starter save.
