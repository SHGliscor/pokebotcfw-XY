# Pokémon Y Starter State Atlas & Froakie Selector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the proven read-only foundation into a hardware-mapped Pokémon Y starter state atlas that safely triggers the Aquacorde sequence, correlates English game data with live runtime state, detects the starter chooser, selects Froakie using the most reliable verified input method, and stops before confirmation.

**Architecture:** Offline tooling derives manifests and starter-event reference data from the user's Y base/update archives, while live probes capture RAM/module/framebuffer deltas around each event transition. A controller layer provides acknowledged short pulses with unconditional release; probes authorize each input only from a freshly proven state and never confirm the starter in this plan.

**Tech Stack:** Python 3.13, standard library, pytest, existing bridge firmware/protocol, user-supplied `Y base.zip` and `Y update.zip`, derived JSON metadata only in Git.

**Spec:** `docs/superpowers/specs/2026-10-06-pokemon-y-starter-bot-design.md`

## Global Constraints

- This plan begins only after Plan 1 Probe 00–02 hardware evidence is accepted.
- Pokémon Y only; English only.
- Canonical save baseline is normal overworld control, no dialogue, starter not received, Party Slot 1 empty, one LEFT input starts the event.
- No blind A-mashing or timing-only progression.
- Direct RAM/runtime state outranks script/message correlation; script/message correlation outranks framebuffer; timing is last.
- Touchscreen, D-pad, and Circle Pad are all candidates; reliability and verifiability outrank speed.
- Starter confirmation is prohibited throughout this plan.
- Unknown state, unexpected game identity, uncertain selection, or controller-release failure means SAFETY_HOLD/stop.
- Historical `DllPoke3Select`, triple-touch, selector coordinates, and old timing data are hypotheses only until freshly re-proven.
- Raw game archives, extracted copyrighted data, framebuffer captures, RAM dumps, and support ZIPs remain outside Git.
- Any generated metadata committed to Git must be derived/sanitized and reproducible from user-owned archives.

## Review Focus

- Base/update contain the same logical resource: update must deterministically override base and provenance must record both; Task 1 owns this test.
- Controller ACK succeeds but release ACK fails: the probe must stop and issue no subsequent action; Task 3 owns this test.
- LEFT pulse moves the player without entering the starter event: Probe 03 must classify trigger as unproven, not continue to dialogue; Task 5 owns this test.
- `DllPoke3Select` appears in an unrelated/ambiguous state: chooser readiness must require additional corroborating state evidence before any selector input; Task 7 owns this test.
- A selector input is sent but Froakie cannot be proven selected: Probe 06 must stop before confirmation and report `SELECTION_UNVERIFIED`; Task 8 owns this test.

---

### Task 1: Reproducible Pokémon Y base/update manifest and resolver

**Files:**
- Create: `game_data/__init__.py`
- Create: `game_data/manifest.py`
- Create: `game_data/resolver.py`
- Create: `game_data/cli.py`
- Test: `tests/game_data/test_manifest.py`
- Test: `tests/game_data/test_resolver.py`
- Create: `docs/game-data/y-source-workflow.md`

**Interfaces:**
- Consumes: local paths to `Y base.zip` and `Y update.zip`.
- Produces:
  - `ArchiveEntry(path: str, size: int, sha256: str, source: Literal["base","update"])`.
  - `ArchiveManifest(archive_sha256: str, entries: tuple[ArchiveEntry, ...])`.
  - `build_manifest(zip_path: Path, source: str) -> ArchiveManifest`.
  - `resolve_logical_path(path: str, base: ArchiveManifest, update: ArchiveManifest) -> ResolvedEntry`.
  - derived manifest JSON containing hashes/provenance but no copyrighted bytes.

- [ ] **Step 1: Write failing manifest/resolver tests**

Use synthetic ZIP fixtures. Assert stable SHA-256, normalized POSIX paths, duplicate-path update override, and explicit provenance.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m pytest tests/game_data -v`  
Expected: FAIL.

- [ ] **Step 3: Implement manifest and update-over-base resolver**

Reject path traversal entries and duplicate normalized paths within the same archive.

- [ ] **Step 4: Add CLI that emits only derived JSON**

CLI accepts explicit `--base`, `--update`, `--output`; it never extracts entire archives into the repository.

- [ ] **Step 5: Run tests**

Run: `python -m pytest tests/game_data -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add game_data tests/game_data docs/game-data/y-source-workflow.md
git commit -m "feat: add reproducible Y game-data resolver"
```

### Task 2: Starter-event reference catalog from user-owned game files

**Files:**
- Create: `game_data/cro_catalog.py`
- Create: `game_data/messages.py`
- Create: `game_data/starter_reference.py`
- Create: `data/generated/.gitkeep`
- Test: `tests/game_data/test_cro_catalog.py`
- Test: `tests/game_data/test_starter_reference.py`

**Interfaces:**
- Consumes: resolved Y 1.5 resource entries and explicit archive paths.
- Produces:
  - `CroRecord(name: str, source_path: str, size: int, sha256: str, metadata: dict[str, int | str])`.
  - `MessageReference(bank: str, message_id: int, text: str, source_path: str)`.
  - `StarterReference(cro_records: tuple[CroRecord, ...], english_messages: tuple[MessageReference, ...], provenance: dict[str, str])`.
  - generated `data/generated/y_1_5/starter_reference.json` only after real-source extraction.

- [ ] **Step 1: Write failing CRO catalog tests**

Synthetic archive fixture must identify named CRO entries without hard-coding old hashes as truth. Include an explicit test that a historical `DllPoke3Select` lead is tagged `reference_candidate` until real extraction verifies its current archive entry.

- [ ] **Step 2: Write failing message/reference tests**

Parser returns decoded text only when the resource format is positively identified; otherwise emits an unsupported-format diagnostic instead of guessed text.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/game_data/test_cro_catalog.py tests/game_data/test_starter_reference.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement focused extractors**

During implementation, inspect the canonical Y base/update layout and add only the parsers needed for starter-relevant CRO/message/script resources. Preserve source path and SHA-256 for every derived record.

- [ ] **Step 5: Generate starter reference metadata from the user's archives**

Expected:
- Y base/update archive hashes recorded,
- current `DllPoke3Select` presence/metadata either verified or explicitly absent,
- English starter-related message/script leads recorded where parseable,
- no raw copyrighted payload committed.

- [ ] **Step 6: Commit source code and sanitized derived metadata**

```bash
git add game_data data/generated/y_1_5/starter_reference.json tests/game_data
git commit -m "feat: derive Y starter event reference metadata"
```

### Task 3: Safe acknowledged controller with unconditional release

**Files:**
- Create: `connection/controller.py`
- Create: `connection/controller_protocol.py`
- Test: `tests/connection/test_controller.py`
- Test: `tests/connection/test_controller_protocol.py`

**Interfaces:**
- Consumes: `BridgeTransport` and exact existing bridge input protocol.
- Produces:
  - `Controller`.
  - `pulse_button(button: Button, duration_ms: int) -> InputReceipt`.
  - `pulse_dpad(direction: DpadDirection, duration_ms: int) -> InputReceipt`.
  - `pulse_cpad(x: int, y: int, duration_ms: int) -> InputReceipt`.
  - `tap_touch(x: int, y: int, duration_ms: int) -> InputReceipt`.
  - `release_all() -> InputReceipt`.
  - `ControllerReleaseError`.

- [ ] **Step 1: Write failing release-safety tests**

Assert every pulse is followed by a release command even when press ACK fails, state observation throws, or the context exits exceptionally.

- [ ] **Step 2: Add ACK/release-failure tests**

A failed release raises `ControllerReleaseError`; no later input may be sent through that controller instance.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/connection/test_controller.py tests/connection/test_controller_protocol.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement exact bridge input protocol**

Inspect the canonical firmware/working ORAS/USUM client only for transport/protocol bytes. Do not import game-specific state logic.

- [ ] **Step 5: Run tests with warnings as errors**

Run: `python -W error::ResourceWarning -m pytest tests/connection/test_controller.py tests/connection/test_controller_protocol.py -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add connection/controller.py connection/controller_protocol.py tests/connection
git commit -m "feat: add fail-closed controller pulses"
```

### Task 4: Runtime observation snapshots and state-diff engine

**Files:**
- Create: `pokemon_y/runtime_snapshot.py`
- Create: `diagnostics/state_diff.py`
- Create: `diagnostics/framebuffer.py`
- Create: `diagnostics/modules.py`
- Test: `tests/pokemon_y/test_runtime_snapshot.py`
- Test: `tests/diagnostics/test_state_diff.py`

**Interfaces:**
- Consumes: targeted RAM reader, candidate/verified address profile, optional bridge module/framebuffer capabilities.
- Produces:
  - `RuntimeSnapshot(timestamp_ns, game_info, trainer, party, rng, modules, ram_regions, framebuffer_ref)`.
  - `capture_runtime_snapshot(...) -> RuntimeSnapshot`.
  - `diff_snapshots(before, after) -> StateDiff`.
  - unsupported bridge capabilities are represented explicitly as `None` plus a diagnostic reason.

- [ ] **Step 1: Write failing snapshot/diff tests**

Assert deterministic comparison of changed bytes/regions, party classification, module set changes, and absence of fake values for unsupported capabilities.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_runtime_snapshot.py tests/diagnostics/test_state_diff.py -v`  
Expected: FAIL.

- [ ] **Step 3: Implement snapshot/diff core**

Keep framebuffer bytes outside JSON; support bundles reference their file path/hash.

- [ ] **Step 4: Map current bridge module/framebuffer read commands**

Use existing firmware lineage/reference client to implement only capabilities actually present. Unit tests pin packet shape and negative/unsupported replies.

- [ ] **Step 5: Run tests**

Run: `python -m pytest tests/pokemon_y/test_runtime_snapshot.py tests/diagnostics/test_state_diff.py -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add pokemon_y/runtime_snapshot.py diagnostics/state_diff.py diagnostics/framebuffer.py diagnostics/modules.py tests
git commit -m "feat: add Y runtime state observation"
```

### Task 5: Probe 03 — canonical LEFT trigger comparison

**Files:**
- Create: `pokemon_y/baseline.py`
- Create: `probes/probe03_left_trigger.py`
- Test: `tests/pokemon_y/test_baseline.py`
- Test: `tests/probes/test_probe03.py`
- Create: `docs/hardware/probe-03-left-trigger.md`

**Interfaces:**
- Consumes: hardware-verified Plan 1 profile, runtime snapshots, Controller.
- Produces:
  - `BaselineAssessment(proven: bool, reasons: tuple[str, ...])`.
  - `assess_starter_baseline(snapshot) -> BaselineAssessment`.
  - `run_probe03(method: Literal["dpad","cpad"], pulse_ms: int, ...) -> ProbeResult`.

- [ ] **Step 1: Write failing baseline tests**

Baseline must reject wrong game, non-empty valid party Slot 1, missing save-loaded evidence, and known active dialogue/event evidence. Unknown required evidence returns unproven, not true.

- [ ] **Step 2: Write failing trigger tests**

Probe sends exactly one LEFT pulse only after baseline proof, captures before/after snapshots, and never presses A. Movement without sufficient event-transition evidence returns `TRIGGER_UNPROVEN`.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_baseline.py tests/probes/test_probe03.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement Probe 03**

No automatic second attempt in one run. D-pad and CPAD are separate invocations so each hardware result is attributable.

- [ ] **Step 5: Hardware-test D-pad LEFT then CPAD LEFT from restored canonical save**

For each method record pulse duration, input ACK/release result, state diff, transition latency, and event-proof confidence.

- [ ] **Step 6: Promote only the winning trigger strategy**

Update sanitized state-atlas metadata with the reliable method and minimum proven pulse range; keep losing/unproven methods as diagnostics, not production fallbacks yet.

- [ ] **Step 7: Commit**

```bash
git add pokemon_y/baseline.py probes/probe03_left_trigger.py docs/hardware/probe-03-left-trigger.md tests data/state_atlas
git commit -m "test: prove Y starter LEFT trigger"
```

### Task 6: Probe 04 — dialogue/state mapper

**Files:**
- Create: `pokemon_y/state_atlas.py`
- Create: `probes/probe04_dialogue_mapper.py`
- Create: `data/state_atlas/schema.json`
- Test: `tests/pokemon_y/test_state_atlas.py`
- Test: `tests/probes/test_probe04.py`
- Create: `docs/hardware/probe-04-dialogue-mapper.md`

**Interfaces:**
- Consumes: canonical trigger result, starter reference metadata, runtime snapshots/diffs.
- Produces:
  - `AtlasState(id: str, evidence: tuple[EvidenceRule, ...], allowed_inputs: tuple[str, ...], next_states: tuple[str, ...])`.
  - `StateAtlas`.
  - mapper mode that records snapshots before/after operator-authorized dialogue advances.

- [ ] **Step 1: Write failing atlas schema tests**

Assert unique state IDs, no transition to undefined states, evidence provenance required, and `allowed_inputs` empty by default.

- [ ] **Step 2: Write failing mapper tests**

The mapper records observations but cannot automatically advance A unless that state already has an approved A transition in the atlas.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_state_atlas.py tests/probes/test_probe04.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement state-atlas model and conservative mapper**

Correlate live observations to extracted English messages/scripts where possible. Store hashes/IDs/text snippets as derived metadata; do not store full raw archives.

- [ ] **Step 5: Hardware-map the path from LEFT trigger to chooser-loading boundary**

Advance conservatively, capturing each transition. Mark any state that relies only on timing as `weak` and prohibit production automation through it until corroborated.

- [ ] **Step 6: Commit sanitized atlas observations**

```bash
git add pokemon_y/state_atlas.py probes/probe04_dialogue_mapper.py data/state_atlas docs/hardware/probe-04-dialogue-mapper.md tests
git commit -m "test: map Y starter dialogue states"
```

### Task 7: Probe 05 — chooser-ready detector

**Files:**
- Create: `pokemon_y/chooser.py`
- Create: `probes/probe05_chooser_detector.py`
- Test: `tests/pokemon_y/test_chooser.py`
- Test: `tests/probes/test_probe05.py`
- Create: `docs/hardware/probe-05-chooser.md`

**Interfaces:**
- Consumes: StateAtlas, runtime module/CRO observations, starter reference catalog.
- Produces:
  - `ChooserAssessment(ready: bool, confidence: Literal["strong","corroborated","insufficient"], evidence: tuple[str, ...])`.
  - `assess_chooser_ready(snapshot, atlas, reference) -> ChooserAssessment`.

- [ ] **Step 1: Write failing chooser tests**

Cover:
- exact current `DllPoke3Select` identity plus expected event state,
- module alone without event corroboration,
- event state without chooser module,
- stale/ambiguous module observation.

Only strong/corroborated evidence may return `ready=True`.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_chooser.py tests/probes/test_probe05.py -v`  
Expected: FAIL.

- [ ] **Step 3: Implement chooser assessment**

No controller object is needed by Probe 05; it observes and stops.

- [ ] **Step 4: Hardware-prove chooser readiness on the current Y install**

Record the current CRO identity/metadata, state-atlas ID, party state, and any framebuffer corroboration.

- [ ] **Step 5: Commit evidence**

```bash
git add pokemon_y/chooser.py probes/probe05_chooser_detector.py docs/hardware/probe-05-chooser.md tests data/state_atlas
git commit -m "test: prove Y starter chooser state"
```

### Task 8: Probe 06 — Froakie selector comparison with no confirmation

**Files:**
- Create: `pokemon_y/selection.py`
- Create: `probes/probe06_froakie_selector.py`
- Test: `tests/pokemon_y/test_selection.py`
- Test: `tests/probes/test_probe06.py`
- Create: `docs/hardware/probe-06-froakie-selector.md`

**Interfaces:**
- Consumes: proven chooser assessment, Controller, runtime snapshot/state-atlas evidence.
- Produces:
  - `SelectionMethod = Literal["touch","dpad","cpad"]`.
  - `SelectionAssessment(target: str | None, proven: bool, evidence: tuple[str, ...])`.
  - `attempt_froakie_selection(method, ...) -> ProbeResult`.
  - no confirmation API.

- [ ] **Step 1: Write failing selection-assessment tests**

Selection is proven only from target-specific runtime evidence or an approved corroborated evidence rule. A successful input ACK alone is insufficient.

- [ ] **Step 2: Write failing Probe 06 safety tests**

Assert:
- chooser must be proven before input,
- exactly one candidate selection action is sent per invocation,
- release is verified,
- no A/confirm command can be emitted,
- unverified result returns `SELECTION_UNVERIFIED`,
- wrong-target evidence returns `WRONG_SELECTION` and stops.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_selection.py tests/probes/test_probe06.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement direct-touch, D-pad, and CPAD strategy adapters**

Coordinates/directions/pulse durations live in candidate config with provenance and must not be promoted until hardware results prove them.

- [ ] **Step 5: Hardware-test methods independently**

Restore the canonical save for each test series. Collect at least five clean attempts per method once the observation itself is stable.

Measure:
- chooser proof,
- input/release success,
- Froakie proof,
- transition latency,
- retries required (must be zero for a clean attempt).

- [ ] **Step 6: Rank production candidate method**

Rank by reliability, then verifiability, then speed. Do not enable confirmation yet.

- [ ] **Step 7: Commit selector evidence and chosen primary candidate**

```bash
git add pokemon_y/selection.py probes/probe06_froakie_selector.py docs/hardware/probe-06-froakie-selector.md tests data/state_atlas config/pokemon_y.json
git commit -m "test: prove Froakie selector without confirmation"
```

### Task 9: State-atlas validation gate

**Files:**
- Create: `tests/state_atlas/test_hardware_atlas_contract.py`
- Modify: `docs/hardware/probe-03-left-trigger.md`
- Modify: `docs/hardware/probe-04-dialogue-mapper.md`
- Modify: `docs/hardware/probe-05-chooser.md`
- Modify: `docs/hardware/probe-06-froakie-selector.md`

**Interfaces:**
- Consumes: all sanitized hardware evidence from Probe 03–06.
- Produces: a machine-testable atlas path from canonical baseline through verified Froakie selection with no confirmation.

- [ ] **Step 1: Write contract test**

Assert there is exactly one approved path:
`STARTER_BASELINE → event/dialogue states → CHOOSER_READY → FROAKIE_SELECTED`.

Every edge must have:
- at least one non-timing evidence rule,
- an allowed input only where required,
- a timeout,
- a stop/recovery classification.

- [ ] **Step 2: Run contract test**

Run: `python -m pytest tests/state_atlas/test_hardware_atlas_contract.py -v`  
Expected: PASS only after actual hardware evidence has populated the atlas.

- [ ] **Step 3: Run full suite**

Run: `python -W error::ResourceWarning -m pytest -q`  
Expected: PASS.

- [ ] **Step 4: Verify repository safety**

Confirm no forbidden binary/game/dump/support artifacts are tracked.

- [ ] **Step 5: Commit final Plan 2 evidence gate**

```bash
git add tests/state_atlas docs/hardware data/state_atlas config/pokemon_y.json
git commit -m "test: validate Y Froakie pre-confirmation state atlas"
```
