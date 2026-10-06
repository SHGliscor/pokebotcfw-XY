# pokebotcfw-XY

Fresh Pokémon X/Y shiny-hunting automation for a CFW Nintendo 3DS using the existing Pokebot RAM/input bridge.

## Development status

Pokémon **Y** is the first validation target. The first production milestone is a fully state-driven **Froakie starter hunt** from the canonical Aquacorde save point.

Current phase: **design and implementation planning**.

## Core rules

- Fresh hardware proof outranks historical X/Y work.
- PK6 validity and live RAM state outrank timing assumptions.
- Unknown state means fail closed.
- A starter is never confirmed unless the target selection is proven.
- A valid shiny permanently disables automatic reset for that encounter.
- No Qt/UI work until the backend is hardware-proven.
- English-only first implementation.
- Pokémon X validation follows Pokémon Y.

## Repository safety

This repository intentionally does **not** contain Nintendo game files, update archives, firmware binaries, save files, raw RAM dumps, support bundles, or other copyrighted/private binaries.

Canonical Pokémon Y base/update data and raw hardware evidence remain outside Git. Only source code, tests, documentation, derived metadata, hashes, sanitized fixtures, and state definitions belong here.

## Design

See:

- `docs/superpowers/specs/2026-10-06-pokemon-y-starter-bot-design.md`

Implementation plans will live under:

- `docs/superpowers/plans/`

## License / disclaimer

This project is for interoperability, automation, and research with hardware/software the user controls. No copyrighted Pokémon game content is distributed by this repository.
