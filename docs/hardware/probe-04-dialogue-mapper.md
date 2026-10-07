# Probe 04 — Pokémon Y starter dialogue mapper

Fresh hardware mapping captured on 2026-10-07.

## Identity

- Title ID: `0004000000055E00`
- Process: `kujira-2`
- Bot controller input during mapping: **none**
- Manual transitions captured: **29**
- States captured: **30**

## Dialogue path

States 000–024 were advanced with one physical `A` per recorded transition. The sequence starts at:

- 000 — “We were just talking about you!”
- 001 — “C’mon, have a seat!”
- …
- 017 — nickname choice list
- 018/019 — nickname confirmation
- 020–024 — post-nickname dialogue leading to the starter chooser

The complete visual evidence remains in the support bundle and is not committed to Git.

## Fresh selector proof

State 025 is the generic three-starter chooser.

Manual hardware transitions:

```
025 generic chooser
  ↓ RIGHT
026 Fennekin focus
  ↓ RIGHT
027 Froakie focus
  ↓ A
028 “Choose this Pokémon?” YES/NO
  ↓ A
029 “Received Froakie!”
```

Bottom-screen SHA-256 references:

- chooser: `4eb756f98a6f5fe5f393ffbd00e9eceb08b504e7564bb32d4946cc730315f80f`
- Fennekin focus: `d1e10fb25824065600e26a046c82b503295aabb2e69664d854d423983695972a`
- Froakie focus: `aa07dd12e52bf493cf5295520e086a250988df18b18a15a8219aa19bd98f2d62`

This freshly proves the previously problematic Fennekin → Froakie transition with physical D-pad RIGHT.

## Final PK6

State 029 contained a checksum-valid Froakie:

- species: 656
- TID/SID: 21080 / 3731
- checksum: `0xE42F`
- EC: `0xEE5EE243`
- PID: `0x4A51D4A7`
- nature: 23
- ability: 67
- IVs: 16 / 7 / 20 / 13 / 13 / 24
- shiny: no

## Next gate

Probe 06 must reproduce only:

```
generic chooser → RIGHT → verified Fennekin → RIGHT → verified Froakie → STOP
```

No confirmation input is authorized in Probe 06.
