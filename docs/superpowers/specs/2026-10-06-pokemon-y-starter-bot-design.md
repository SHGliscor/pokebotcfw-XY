# Pokémon Y Starter-First Bot Design

**Date:** 2026-10-06  
**Project:** Pokebot3DS-Y  
**Primary game:** Pokémon Y  
**Primary validation target:** Froakie  
**Language:** English only  
**Console:** CFW New Nintendo 3DS XL using the existing Pokebot RAM/input bridge firmware

## 1. Purpose

Build a fresh, standalone Pokémon Y shiny-hunting backend that begins with the Kalos starter sequence and is designed for reliable unattended operation. The first production milestone is a Froakie hunt that can start from a canonical Aquacorde save, trigger the starter event, navigate the long dialogue/cutscene sequence, select and verify Froakie, decode the generated PK6, decide shiny status, safely reset a normal result, and return to the same known baseline.

The implementation must prioritize correctness and state verification over blind timing. Long fixed delays may be used only as minimum debounce/safety waits after state evidence; they must not be the primary source of truth for progression.

## 2. Scope

### 2.1 In scope for the first development phase

- Pokémon Y only.
- English game language only. The baseline probe must confirm the English configuration when a reliable language flag/message-bank signal is available; otherwise the run is explicitly configured as English and the extracted English script/message correlation must match the observed event flow before production automation is enabled.
- Reuse of the existing proven 3DS `boot.firm` RAM/input bridge.
- Fresh PC/Python backend implementation.
- Canonical Aquacorde starter save with normal overworld control and no dialogue open.
- One LEFT input from D-pad or Circle Pad to trigger the starter sequence.
- Froakie as the first hardware-development target.
- Chespin and Fennekin support after the shared starter engine is proven with Froakie.
- Game-file analysis using the user's canonical `Y base.zip` and `Y update.zip`.
- Runtime RAM/CRO/state discovery and validation.
- Gen 6 PK6 decryption, validation, decoding, and shiny decision.
- State-machine-driven starter automation.
- Fail-closed recovery and shiny HOLD behavior.
- Standalone probes, logs, RAM snapshots, and support ZIPs.
- Timing instrumentation and post-reliability optimization.

### 2.2 Deferred until the starter backend is proven

- Normal wild encounters.
- Hordes / Sweet Scent.
- Friend Safari.
- Chain fishing.
- Eligible static/gift/fossil hunts.
- Eggs.
- Poké Radar.
- Advanced RNG-assisted early rejection.
- Pokémon X parity.
- Multi-language support.
- Qt UI, dashboards, statistics pages, Discord integration, persistent shiny history, theming, and visual polish.

## 3. Canonical runtime baseline

The first production starter hunt assumes a save made at the closest practical point before the Kalos starter event in Aquacorde Town.

The canonical baseline is:

- Pokémon Y is running.
- The save has loaded completely.
- Normal overworld player control is available.
- No dialogue box is open.
- The starter has not yet been received.
- Party Slot 1 is expected to be empty.
- A single deliberate LEFT input from D-pad or Circle Pad begins the starter event.

The bot must prove this baseline before sending the first LEFT input. If the baseline cannot be proven, automation must not begin.

## 4. Source-of-truth hierarchy

The runtime state machine must rank evidence in this order:

1. Valid live PK6 / direct RAM state.
2. Freshly hardware-verified runtime state or CRO/module identity.
3. Freshly correlated game-script/message state.
4. Controller acknowledgement and verified release.
5. Framebuffer confirmation when required as a fallback or corroborating signal.
6. Timing assumptions.

A lower-ranked signal may never override contradictory higher-ranked evidence.

External projects and old Pokebot notes are references, not production authority. Hardware proof on the current Pokémon Y installation is required before a candidate address, state, selector behavior, or recovery path becomes production-authorized.

## 5. Canonical data and reference inputs

### 5.1 User game data

Use these as the canonical Pokémon Y game-file source set:

- `Y base.zip`
- `Y update.zip`

The update is resolved over the base wherever both provide the same logical resource. The extraction pipeline must be reproducible and hash its inputs and relevant outputs.

Relevant resource classes include:

- CRO/module data.
- English message archives.
- Event/script resources.
- Map/zone/event placement data.
- Executable/code references when useful for state discovery.

No copyrighted game archives are redistributed by the project. Generated metadata, hashes, and derived state maps may be stored.

### 5.2 External technical references

Use, cross-check, and cite internally as engineering references:

- PKHeX: PK6 structure, encryption/decryption, field semantics, checksum, shiny logic.
- PokeReader: Gen 6 RAM/RNG/address leads.
- PKMN-NTR and related Gen 6 NTR projects: runtime state, control, and offset leads where available.
- 3DSRNGTool / Gen 6 RNG research: MT/TinyMT behavior and later optimization research.
- Historical Pokebot X/Y notes: useful leads only; all critical claims must be freshly re-proven.

## 6. Known identity and candidate address leads

These values are reference candidates until the fresh hardware probe validates them.

### 6.1 Game identity

- Pokémon Y Title ID: `0004000000055E00`
- Process: `kujira-2`

The bot must fail closed if the running game identity does not match the expected Pokémon Y profile.

### 6.2 RAM candidate leads

- TID/SID: `0x08C79C3C`
- Party base: `0x08CE1CF8`
- Party stride: `484` bytes
- Stored PK6 size: `232` bytes
- Wild base: `0x081FF744`
- Poké Radar chain: `0x08D1B2B8`
- Initial seed: `0x08C52844`
- MT index: `0x08C52848`
- MT state start: `0x08C5284C`
- TinyMT state: `0x08C52808`

For discovery, probes must capture surrounding memory windows rather than assuming an exact pointer is correct. Production code may only consume a candidate address after fresh hardware validation.

## 7. Runtime architecture

The backend is divided into independent responsibilities.

### 7.1 Connection layer

Responsibilities:

- Connect to the 3DS bridge.
- PING / GAME_INFO.
- Targeted memory reads.
- Controller command transport.
- Controller acknowledgement handling.
- Deterministic socket cleanup.
- No game-state decisions.

### 7.2 State reader

Responsibilities:

- Read game identity.
- Read trainer identity.
- Read party memory.
- Enumerate or identify relevant runtime modules/CROs when supported.
- Read candidate script/message state.
- Read RNG state for observation.
- Return structured state only.
- Never press buttons.

### 7.3 Controller layer

Responsibilities:

- Short button pulses.
- D-pad pulses.
- Circle Pad pulses with explicit neutral release.
- Touchscreen tap/release.
- Ack tracking.
- Mandatory release verification.
- No game-state decisions.

### 7.4 PK6 layer

Responsibilities:

- Gen 6 PK6 crypto/decryption.
- Block order handling.
- Checksum validation.
- Structural validity.
- Species, EC, PID, TID/SID, nature, ability, gender, IVs, moves, nickname, level when available.
- Shiny calculation.
- Test-vector parity against PKHeX.

### 7.5 Starter state machine

Responsibilities:

- Authorize inputs only from proven states.
- Enforce allowed transitions.
- Enforce per-state timeout and recovery policy.
- Coordinate selection verification.
- Wait for first valid generated PK6.
- Route normal results to reset, shiny results to SHINY_HOLD, and uncertain states to SAFETY_HOLD.

## 8. Starter state model

The initial production state graph is:

```text
BOOT_CHECK
→ WAIT_FOR_GAME
→ VERIFY_Y
→ WAIT_FOR_SAVE_LOADED
→ VERIFY_STARTER_BASELINE
→ TRIGGER_STARTER_EVENT
→ ADVANCE_DIALOGUE
→ WAIT_FOR_CHOOSER
→ SELECT_FROAKIE
→ VERIFY_FROAKIE
→ CONFIRM_STARTER
→ WAIT_FOR_PK6
→ VALIDATE_PK6
→ DECIDE_RESULT
   ├─ NORMAL → RESET → WAIT_FOR_SAVE_LOADED
   ├─ SHINY  → SHINY_HOLD
   └─ ERROR  → SAFETY_HOLD
```

Every state must define:

- Entry conditions.
- Allowed inputs.
- Success conditions.
- Timeout.
- Retry allowance.
- Recovery class.
- Diagnostic data to preserve on failure.

## 9. Dialogue and script strategy

The production bot must not be a long sequence of fixed sleeps and A presses.

The offline extraction layer will attempt to correlate:

- starter event scripts,
- English dialogue/message entries,
- runtime module/CRO states,
- live RAM values,
- and framebuffer evidence where necessary.

The desired runtime model is:

```text
known dialogue/script state
→ allowed input
→ wait for a different proven state
```

rather than:

```text
sleep fixed duration
→ press A
```

If exact message identifiers are not exposed in a reliable live form, the implementation may gate dialogue using a combination of script state, CRO/module state, and carefully bounded timing. Framebuffer checks are a fallback/corroborating mechanism, not the preferred primary detector.

## 10. Starter chooser strategy

Froakie is the first development target because it exercises movement away from the default/center selection and reproduces the prior failure point.

The chooser is its own subsystem and must test these methods independently:

1. Direct touchscreen selection.
2. D-pad navigation.
3. Circle Pad navigation.

Selection strategy is ranked by:

1. Reliability.
2. Verifiability.
3. Speed.

The chooser must prove the resulting target before confirmation. Sending an input successfully is not proof of selection.

Production confirmation is forbidden unless runtime evidence resolves to:

```text
selected_starter = FROAKIE
```

or an equally strong target-specific state.

Historical leads such as `DllPoke3Select` and old triple-touch behavior are instrumentation targets only until freshly re-proven.

## 11. Froakie authority and shiny decision

The first production shiny decision is based on the first checksum-valid generated PK6 in the expected party slot.

For a valid Froakie result, require:

- PK6 checksum valid.
- Species ID `656`.
- Trainer IDs match the active save.
- Structure is otherwise plausible/valid.

Then decode and record at minimum:

- EC.
- PID.
- TID/SID.
- Nature.
- Ability.
- Gender.
- IVs.
- Moves when available.
- Shiny result.

A generated PK6 is the initial authority for shiny/non-shiny decisions. RNG prediction may be researched later but cannot replace PK6 authority until separately validated over a large hardware sample.

## 12. Recovery model

### 12.1 Class A — safe retry

Allowed only when the current state remains known and the same action is demonstrably idempotent/safe.

Examples:

- transient RAM read timeout,
- missed controller acknowledgement,
- delayed dialogue transition,
- chooser input not registering while chooser state remains proven.

Retry counts must be small and state-specific.

### 12.2 Class B — reset to canonical baseline

Allowed only after hardware testing proves soft reset is safe from that state.

Possible examples:

- known dialogue drift,
- chooser state lost before confirmation,
- post-confirm timeout before a valid PK6 appears.

The recovery path is:

```text
release all inputs
→ soft reset
→ reload save
→ prove canonical baseline
→ resume
```

### 12.3 Class C — hard safety HOLD

Required for any state where further input could lose a shiny, confirm the wrong target, or operate on untrusted memory.

Examples:

- wrong species,
- invalid checksum after expected generation completion,
- TID/SID mismatch,
- uncertain shiny status,
- game identity change,
- implausible RAM state,
- stuck/held controller state,
- repeated recovery failure,
- unverified starter selection.

## 13. SHINY_HOLD behavior

When a valid shiny Froakie is detected, the backend must atomically transition to SHINY_HOLD:

1. Cancel queued actions.
2. Release every button.
3. Neutralize Circle Pad.
4. Release touchscreen.
5. Disable reset path.
6. Persist encounter metadata.
7. Save raw PK6 and diagnostic state.
8. Trigger shiny alert sound.
9. Remain stopped until explicit user intervention.

There is no automatic transition from SHINY_HOLD back to RESET.

## 14. Controller discipline

Default controller behavior is short acknowledged pulses followed by mandatory release.

Examples:

```text
press A
→ wait acknowledgement
→ release A
→ verify release
```

```text
CPAD_LEFT for measured pulse
→ neutral
→ verify neutral
```

```text
touch(x,y)
→ release touch
→ verify release
```

Long holds are prohibited unless a specific hardware test proves they are required and safe.

## 15. Hardware probe sequence

The first implementation is a standalone research package with no Qt UI and no appdata writes.

### Probe 00 — connection and game identity

- PING.
- GAME_INFO.
- Verify Pokémon Y Title ID/process.
- Measure basic read latency.
- Send no input.

### Probe 01 — trainer and party

- Read candidate TID/SID.
- Dump candidate party memory and surrounding window.
- Confirm Party Slot 1 is empty/invalid at baseline.
- Send no input.

### Probe 02 — baseline state atlas entry

Capture the canonical save state:

- relevant RAM windows,
- runtime module/CRO state,
- trainer data,
- party state,
- RNG state,
- optional framebuffer.

### Probe 03 — LEFT trigger

- Prove baseline.
- Send one controlled LEFT pulse.
- Release immediately.
- Prove the starter event began.
- Compare D-pad and Circle Pad approaches.

### Probe 04 — dialogue mapper

- Observe the sequence while dialogue is advanced conservatively.
- Record live RAM/module/RNG/framebuffer changes at each transition.
- Correlate live states with extracted English messages/scripts.

### Probe 05 — chooser detector

- Prove chooser-ready state.
- Identify strong runtime evidence, preferably module/CRO or direct chooser state.
- Send no selection input.

### Probe 06 — Froakie selector

Test touchscreen, D-pad, and Circle Pad independently.

For each method:

- prove chooser ready,
- send the candidate selection action,
- release,
- verify Froakie selection,
- stop before confirmation.

### Probe 07 — first confirmation

- Prove Froakie selected.
- Confirm exactly once using the validated action.
- Observe without resetting.

### Probe 08 — PK6 generation

- Poll Party Slot 1 rapidly after confirmation.
- Capture the first checksum-valid Froakie PK6.
- Record generation timing and RNG snapshots.
- Cross-check decode fields.

### Probe 09 — shiny decision dry run

- Calculate shiny status.
- Print `WOULD_RESET` or `WOULD_HOLD`.
- Do not reset.

### Probe 10 — one-shot reset

- On a valid non-shiny Froakie, release controls and soft reset.
- Reload save.
- Prove canonical baseline.
- Stop.

### Probe 11 — one full automatic cycle

- Baseline → LEFT → dialogue → chooser → Froakie → verify → confirm → PK6 → normal → reset → baseline.
- Stop after one full cycle.

### Probe 12 — reliability ramp

- 5 clean cycles.
- 20 clean cycles.
- 100 clean cycles.
- Only then enable continuous mode.

## 16. Probe diagnostics and support bundles

Every standalone probe must create a timestamped support bundle with enough evidence to diagnose failures without rerunning blindly.

Expected structure:

```text
support/
  log.txt
  result.json
  game_info.json
  ram/
  pk6/
  rng/
  framebuffer/
```

The launcher packages this into a timestamped ZIP automatically.

The probe must close sockets/transports deterministically, including on Python 3.13/Windows, to avoid leaked asyncio/proactor transport warnings.

## 17. Performance strategy

Reliability is completed before aggressive timing optimization.

For each state transition, record:

- state entry time,
- earliest proven exit condition,
- actual input time,
- transition latency,
- unnecessary wait time.

The production engine should then operate on:

```text
state becomes valid
+ minimum proven debounce margin
→ next allowed input
```

rather than worst-case fixed sleeps.

Research probes may perform broad RAM reads and framebuffer capture. Production code should use small targeted reads and poll only signals relevant to the current state.

Suggested polling classes after measurement:

- critical short transition: approximately 20–50 ms,
- normal dialogue state: approximately 50–100 ms,
- long cutscene wait: approximately 100–250 ms.

These are starting ranges only and must be tuned from real N3DS measurements.

## 18. RNG observation and later optimization

During generation probes, record available MT/TinyMT state around:

- canonical baseline,
- before chooser,
- before Froakie confirmation,
- immediately after confirmation,
- first valid PK6.

Initially RNG is observational only.

A later optimization may attempt earlier non-shiny rejection if a deterministic prediction can be proven against a large sample of actual generated PK6 results. Such a path remains optional and may never reduce shiny safety.

## 19. Project structure

Initial repository structure:

```text
Pokebot3DS-Y/
├── README.md
├── requirements.txt
├── RUN_PROBE.bat
├── config/
├── connection/
├── gen6/
├── pokemon_y/
├── game_data/
├── probes/
├── starter/
├── diagnostics/
├── data/
│   ├── extracted/
│   ├── generated/
│   └── state_atlas/
├── tests/
└── docs/
    └── superpowers/
        ├── specs/
        └── plans/
```

Research probes may be verbose and exploratory. Production modules may only depend on signals promoted to hardware-verified status.

Address/state metadata should carry provenance and verification status so speculative leads cannot silently become production authority.

## 20. UI boundary

There is no Qt UI during the research/probe phase.

The early workflow is intentionally:

- `.bat` launcher,
- console output,
- JSON logs,
- support ZIPs.

A separate clean Pokémon Y application/UI begins only after the Froakie backend can continuously hunt safely.

The eventual UI consumes stable backend APIs and contains no direct RAM/state-machine logic.

Preferred future navigation:

- Dashboard.
- Hunt.
- Statistics.
- Settings.

Visual design and persistent statistics are out of scope until the backend is proven.

## 21. Post-Froakie expansion order

After the Kalos starter engine is production-capable:

1. Generalize Chespin/Fennekin/Froakie target selection.
2. Normal wild encounters.
3. Hordes / Sweet Scent.
4. Friend Safari.
5. Chain fishing.
6. Eligible static/gift/fossil encounters.
7. Eggs.
8. Poké Radar.
9. Advanced RNG-assisted optimization.
10. Pokémon X validation/parity.

Each hunt mode gets its own hardware-validation sequence rather than inheriting unproven assumptions from starters.

## 22. Acceptance criteria for the first production phase

The first production phase is complete only when the Froakie backend demonstrates:

- 100+ consecutive completed valid Froakie cycles without an automation fault; a genuine shiny that correctly enters SHINY_HOLD is a successful safe termination rather than a failed validation cycle.
- 0 wrong-starter confirmations.
- 0 unverified confirmations.
- 0 held-input failures.
- 0 missed valid PK6 reads.
- 0 false shiny classifications.
- 0 unexplained state transitions.
- All tested invalid/unknown states route to a safe HOLD or an explicitly validated reset-to-baseline recovery.

The system must also demonstrate that deliberately injected bad-state conditions fail closed.

## 23. Non-negotiable design rules

- Fresh hardware proof outranks old project notes.
- PK6 validity outranks timing assumptions.
- The bot never confirms a starter selection it cannot prove.
- Unknown state means stop or validated recovery, never input spam.
- Shiny detection permanently disables automatic reset for that encounter.
- Research instrumentation may be heavy; production runtime should be minimal.
- No GUI work before the backend milestone is proven.
- English-only first implementation.
- Pokémon Y first; Pokémon X comes later.