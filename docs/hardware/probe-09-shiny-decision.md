# Probe 09 — shiny decision dry run

Hardware run: `PokemonY_Probe09_20261007_023232_333336`

Result: **PASS_DECISION_DRY_RUN**

## Live authority

- Title ID: `0004000000055E00`
- Process: `kujira-2`
- Party Slot 1 classification: valid
- species: 656 (Froakie)
- TID/SID: 21080 / 3731
- checksum: `0xE1C3`
- EC: `0xEA0A7443`
- PID: `0x92A35695`
- nature: 5
- ability: 67
- IVs: 10 / 29 / 29 / 15 / 8 / 31
- shiny: false

## Decision

```
WOULD_RESET
```

The probe was read-only:

- controller_input_sent: false
- reset_sent: false

## Gate result

This exact live non-shiny Froakie is authorized for the next one-shot reset hardware probe. Any shiny, trainer mismatch, wrong species, or invalid PK6 remains reset-prohibited.
