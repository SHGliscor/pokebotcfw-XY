# Probe 06 — automated Froakie selector

Hardware run: `PokemonY_Probe06_20261007_020950_069607`

Result: **PASS_FROAKIE_SELECTED**

## Identity

- Title ID: `0004000000055E00`
- Process: `kujira-2`
- Bridge: `Pokebot3DS-Luma-v0p6-n3ds-fb1`

## Automated selector proof

Start classifier: `GENERIC`

Controller actions:

```
GENERIC
  ↓ DPAD_RIGHT (100 ms hold / 150 ms settle)
FENNEKIN
  ↓ DPAD_RIGHT (100 ms hold / 150 ms settle)
FROAKIE
  ↓ RELEASE_ALL
STOP
```

Both controller sequences terminated as `COMPLETED`.

- first sequence: `56470925`
- second sequence: `56470934`
- terminal HID: neutral (`0xFFF`)
- confirmation sent: **false**
- further input sent: **false**

## Visual evidence

- generic chooser SHA-256: `4eb756f98a6f5fe5f393ffbd00e9eceb08b504e7564bb32d4946cc730315f80f`
- automated Fennekin SHA-256: `9099ec141a3344e82437a56171b85dd80b43c16f3b5591745cba017bd6552801`
- automated Froakie SHA-256: `38e873aed67ffb846e10aa2aa096e42829109be99aca1d9268b7aef876aa7542`

The selector uses cursor-position classification rather than exact animated-frame equality.

## Gate result

The production selector contract is now hardware-proven for:

```
generic chooser → Fennekin → Froakie
```

No starter confirmation was performed by Probe 06.
