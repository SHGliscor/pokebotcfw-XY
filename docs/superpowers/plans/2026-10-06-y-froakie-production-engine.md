# Pokémon Y Froakie Generation & Production Hunt Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the hardware-proven pre-confirmation state atlas into a production-capable Froakie shiny-hunting engine that confirms only a proven Froakie selection, captures the first valid generated PK6, safely decides shiny status, resets normal results back to the canonical save, and reaches the 100-cycle reliability gate.

**Architecture:** Confirmation, PK6 generation, shiny decision, reset, and recovery are first implemented as one-shot probes. Only after each one is hardware-proven are they composed into a strict finite-state machine with explicit state evidence, retry/reset/HOLD policies, atomic controller release, compact production logging, and an irreversible SHINY_HOLD path.

**Tech Stack:** Python 3.13, pytest, existing bridge/controller core, hardware-verified state atlas, Gen 6 PK6 parser, standard-library logging/json/zipfile/time, simple local alert sound abstraction.

**Spec:** `docs/superpowers/specs/2026-10-06-pokemon-y-starter-bot-design.md`

## Global Constraints

- Plan 1 and Plan 2 hardware gates must already pass.
- Production confirmation is allowed only from the freshly proven `FROAKIE_SELECTED` state.
- Valid Froakie authority requires checksum-valid PK6, species `656`, trainer IDs matching the active save, and structurally valid data.
- PK6 is the shiny authority; RNG data is observational only in this plan.
- Shiny detection must cancel queued actions, release all controls, disable reset, persist evidence, alert, and remain in SHINY_HOLD until explicit user intervention.
- Unknown/invalid memory, wrong species, trainer mismatch, uncertain selection, controller-release failure, repeated recovery failure, or uncertain shiny status route to SAFETY_HOLD.
- Timing optimizations may be applied only after reliability evidence; fixed timing cannot override runtime evidence.
- No Qt/UI work.
- No automatic continuous mode before 5-, 20-, then 100-cycle reliability gates.
- Raw RAM/support/PK6 artifacts remain ignored by Git; only sanitized summaries/derived metrics are committed.

## Review Focus

- A stale `FROAKIE_SELECTED` snapshot is reused after the chooser state changes: confirmation must re-read and re-prove selection immediately before input; Task 1 owns this test.
- Party Slot 1 briefly contains invalid/intermediate bytes during generation: the poller must wait for a checksum-valid Froakie and never classify invalid intermediate bytes as shiny; Task 2 owns this test.
- A valid PK6 belongs to a different trainer/species: decision must route to SAFETY_HOLD, never reset; Task 3 owns this test.
- Reset input succeeds but canonical baseline never returns: recovery must stop after the state-specific timeout/retry budget rather than spam reset/Continue inputs; Task 4 owns this test.
- Shiny is detected while an input/retry is queued: SHINY_HOLD must cancel that queue before any later reset/action executes; Task 5 owns this test.

---

### Task 1: Probe 07 — one verified Froakie confirmation

**Files:**
- Create: `probes/probe07_froakie_confirm.py`
- Create: `pokemon_y/confirmation.py`
- Test: `tests/pokemon_y/test_confirmation.py`
- Test: `tests/probes/test_probe07.py`
- Create: `docs/hardware/probe-07-confirm.md`

**Interfaces:**
- Consumes: current live snapshot, proven chooser state, proven Froakie selection, Controller.
- Produces:
  - `ConfirmationAssessment(authorized: bool, reasons: tuple[str, ...])`.
  - `authorize_froakie_confirmation(snapshot, atlas) -> ConfirmationAssessment`.
  - `run_probe07(...) -> ProbeResult`.

- [ ] **Step 1: Write failing authorization tests**

Assert confirmation is denied for stale snapshots, wrong target, unverified target, chooser not ready, wrong game, non-empty unexpected party, and expired state evidence.

- [ ] **Step 2: Write failing Probe 07 tests**

Immediately before confirm, Probe 07 captures a fresh snapshot and re-runs authorization. On success it sends exactly one approved confirmation action, verifies release, captures post-input snapshots, and never resets.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_confirmation.py tests/probes/test_probe07.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement confirmation authorization and one-shot probe**

Use only the Plan 2 hardware-ranked selection/confirmation action. Do not add fallback A-spam.

- [ ] **Step 5: Hardware-run Probe 07 once from freshly proven Froakie selection**

Expected: one confirmation, clean release, observation continues, no reset.

- [ ] **Step 6: Commit sanitized result**

```bash
git add pokemon_y/confirmation.py probes/probe07_froakie_confirm.py docs/hardware/probe-07-confirm.md tests
git commit -m "test: prove one-shot Froakie confirmation"
```

### Task 2: Probe 08 — earliest valid Froakie PK6 generation point

**Files:**
- Create: `pokemon_y/generation.py`
- Create: `probes/probe08_pk6_generation.py`
- Test: `tests/pokemon_y/test_generation.py`
- Test: `tests/probes/test_probe08.py`
- Create: `docs/hardware/probe-08-pk6-generation.md`

**Interfaces:**
- Consumes: party reader, trainer IDs, PK6 classifier, optional RNG candidate reader.
- Produces:
  - `GenerationObservation(first_valid_ns: int | None, samples: int, pokemon: PK6 | None, invalid_samples: int)`.
  - `wait_for_generated_froakie(..., timeout_s: float, poll_ms: int) -> GenerationObservation`.
  - `run_probe08(...) -> ProbeResult`.

- [ ] **Step 1: Write failing generation tests**

Feed sequences:
- empty → invalid intermediate → valid Froakie,
- empty → valid wrong species,
- empty → valid wrong trainer,
- empty forever through timeout.

Only checksum-valid species 656 with matching TID/SID is accepted.

- [ ] **Step 2: Assert invalid intermediate data has no shiny authority**

Test that no invalid/intermediate sample calls the shiny decision function.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_generation.py tests/probes/test_probe08.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement fast targeted polling and timing capture**

Record monotonic timestamps, sample count, invalid intermediate count, first-valid latency, and available MT/TinyMT snapshots around confirmation/first valid PK6.

- [ ] **Step 5: Hardware-run generation probe across several Froakie receives**

Measure earliest stable poll interval and first-valid generation point without resetting automatically.

- [ ] **Step 6: Commit sanitized generation timing evidence**

```bash
git add pokemon_y/generation.py probes/probe08_pk6_generation.py docs/hardware/probe-08-pk6-generation.md tests
git commit -m "test: map Froakie PK6 generation point"
```

### Task 3: Probe 09 — shiny decision dry run and Froakie authority

**Files:**
- Create: `pokemon_y/decision.py`
- Create: `probes/probe09_shiny_dry_run.py`
- Test: `tests/pokemon_y/test_decision.py`
- Test: `tests/probes/test_probe09.py`

**Interfaces:**
- Consumes: `PK6`, active `TrainerIds`.
- Produces:
  - `Decision = Literal["WOULD_RESET","WOULD_HOLD","SAFETY_HOLD"]`.
  - `decide_froakie_result(pokemon: PK6, trainer: TrainerIds) -> Decision`.

- [ ] **Step 1: Write failing decision tests**

Assert:
- valid non-shiny species 656 matching trainer → `WOULD_RESET`,
- valid shiny species 656 matching trainer → `WOULD_HOLD`,
- wrong species → `SAFETY_HOLD`,
- trainer mismatch → `SAFETY_HOLD`.

- [ ] **Step 2: Write dry-run probe tests**

Probe 09 prints/persists the decision but has no reset/controller dependency.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_decision.py tests/probes/test_probe09.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement authority/decision logic**

Do not add target Nature/IV filters yet; this plan's decision is shiny safety only.

- [ ] **Step 5: Hardware-run on a valid received Froakie**

Expected for an ordinary starter: `WOULD_RESET`; actual reset remains disabled.

- [ ] **Step 6: Commit**

```bash
git add pokemon_y/decision.py probes/probe09_shiny_dry_run.py tests
git commit -m "feat: add Froakie shiny authority decision"
```

### Task 4: Probe 10 — one-shot reset and canonical-baseline return

**Files:**
- Create: `pokemon_y/reset.py`
- Create: `probes/probe10_one_shot_reset.py`
- Test: `tests/pokemon_y/test_reset.py`
- Test: `tests/probes/test_probe10.py`
- Create: `docs/hardware/probe-10-reset.md`

**Interfaces:**
- Consumes: Controller, live game state, canonical baseline assessor, only a pre-authorized non-shiny decision.
- Produces:
  - `ResetResult(reset_sent: bool, baseline_recovered: bool, elapsed_s: float, reasons: tuple[str, ...])`.
  - `perform_soft_reset_once(...) -> ResetResult`.
  - `run_probe10(...) -> ProbeResult`.

- [ ] **Step 1: Write failing reset authorization tests**

Reset is prohibited unless the current encounter decision is exactly valid non-shiny `WOULD_RESET`. Shiny/SAFETY_HOLD/unknown cannot invoke reset.

- [ ] **Step 2: Write failing reset choreography tests**

Pin the exact soft-reset controller chord using the working Gen 6 bridge client/hardware proof, then model title/Continue/save-load observations as state waits rather than fixed sleeps. Timeout causes stop, not repeated uncontrolled reset.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/pokemon_y/test_reset.py tests/probes/test_probe10.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement one-shot reset with bounded state waits**

Controller is released before and after reset chord. Continue/title inputs are individually state-authorized as their detection signals become available.

- [ ] **Step 5: Hardware-run exactly one non-shiny reset**

Expected:
- normal received Froakie authorizes reset,
- title/save route completes,
- canonical starter baseline is re-proven,
- probe stops.

- [ ] **Step 6: Commit sanitized reset evidence**

```bash
git add pokemon_y/reset.py probes/probe10_one_shot_reset.py docs/hardware/probe-10-reset.md tests
git commit -m "test: prove one-shot Y starter reset recovery"
```

### Task 5: Production state machine, recovery classes, SAFETY_HOLD, and SHINY_HOLD

**Files:**
- Create: `starter/__init__.py`
- Create: `starter/states.py`
- Create: `starter/engine.py`
- Create: `starter/recovery.py`
- Create: `starter/holds.py`
- Create: `starter/action_queue.py`
- Create: `diagnostics/alert.py`
- Test: `tests/starter/test_engine.py`
- Test: `tests/starter/test_recovery.py`
- Test: `tests/starter/test_holds.py`

**Interfaces:**
- Consumes: all hardware-proven modules from Plans 1–3 Tasks 1–4.
- Produces:
  - `StarterState` enum matching the approved state graph.
  - `StarterEngine.step(snapshot) -> EngineOutcome`.
  - `RecoveryClass = Literal["A_RETRY","B_RESET","C_HOLD"]`.
  - `enter_safety_hold(reason: str) -> HoldRecord`.
  - `enter_shiny_hold(pokemon: PK6, evidence: ...) -> HoldRecord`.
  - cancellable `ActionQueue` with no execution after HOLD.

- [ ] **Step 1: Write failing transition-table tests**

Every state defines entry evidence, allowed action, success transition, timeout, retry budget, and recovery class. Undefined transitions route to SAFETY_HOLD.

- [ ] **Step 2: Write failing SHINY_HOLD tests**

Inject a shiny result while reset/retry actions are queued. Assert queue cancellation occurs before release-all/hold persistence and no queued action can execute afterward.

- [ ] **Step 3: Write failing recovery tests**

Class A retries only explicitly idempotent actions within a small budget. Class B resets only from hardware-approved states. Class C never sends progression input.

- [ ] **Step 4: Run tests and verify failure**

Run: `python -m pytest tests/starter -v`  
Expected: FAIL.

- [ ] **Step 5: Implement state machine and hold paths**

SHINY_HOLD order is:
cancel queue → release all → neutralize CPAD/touch → disable reset path → persist metadata/raw PK6 diagnostics → alert → remain stopped.

- [ ] **Step 6: Run starter tests**

Run: `python -W error::ResourceWarning -m pytest tests/starter -v`  
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add starter diagnostics/alert.py tests/starter
git commit -m "feat: add fail-closed Froakie hunt state machine"
```

### Task 6: Probe 11 — one complete automatic Froakie cycle

**Files:**
- Create: `probes/probe11_full_cycle.py`
- Create: `diagnostics/cycle_metrics.py`
- Test: `tests/probes/test_probe11.py`
- Test: `tests/diagnostics/test_cycle_metrics.py`
- Create: `docs/hardware/probe-11-full-cycle.md`

**Interfaces:**
- Consumes: StarterEngine and hardware-verified state atlas.
- Produces:
  - one cycle `STARTER_BASELINE → ... → valid non-shiny Froakie → RESET → STARTER_BASELINE`,
  - `CycleMetrics` with per-state timestamps/retries and total duration.

- [ ] **Step 1: Write failing full-cycle simulation tests**

Use deterministic fake snapshots to prove normal cycle, shiny termination, wrong-species hold, selector failure hold, and reset timeout hold/recovery behavior.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m pytest tests/probes/test_probe11.py tests/diagnostics/test_cycle_metrics.py -v`  
Expected: FAIL.

- [ ] **Step 3: Implement one-cycle harness**

It must stop after one baseline-to-baseline normal cycle or immediately on either HOLD.

- [ ] **Step 4: Run full unit suite**

Run: `python -W error::ResourceWarning -m pytest -q`  
Expected: PASS.

- [ ] **Step 5: Hardware-run one complete cycle**

Inspect support bundle and metrics before allowing repeated cycles.

- [ ] **Step 6: Commit sanitized cycle evidence**

```bash
git add probes/probe11_full_cycle.py diagnostics/cycle_metrics.py docs/hardware/probe-11-full-cycle.md tests
git commit -m "test: prove one complete automatic Froakie cycle"
```

### Task 7: Reliability runner and 5/20/100 cycle gates

**Files:**
- Create: `starter/runner.py`
- Create: `diagnostics/session_metrics.py`
- Create: `probes/probe12_reliability.py`
- Test: `tests/starter/test_runner.py`
- Test: `tests/diagnostics/test_session_metrics.py`
- Create: `docs/hardware/probe-12-reliability.md`

**Interfaces:**
- Consumes: StarterEngine, CycleMetrics.
- Produces:
  - `run_cycles(limit: int) -> SessionMetrics`.
  - stop immediately on SHINY_HOLD or SAFETY_HOLD.
  - reliability counters: completed cycles, wrong confirmations, unverified confirmations, held-input failures, missed valid PK6 reads, false shiny classifications, unexplained transitions, retries, state durations.

- [ ] **Step 1: Write failing runner tests**

Assert exact cycle limit, immediate stop on either HOLD, counters never hide a failed/held cycle, and continuous/unbounded mode is rejected until the 100-cycle gate file/status is present.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m pytest tests/starter/test_runner.py tests/diagnostics/test_session_metrics.py -v`  
Expected: FAIL.

- [ ] **Step 3: Implement bounded reliability runner**

Allowed limits initially: 5, 20, 100. No unlimited mode yet.

- [ ] **Step 4: Hardware-run 5 clean cycles**

Review metrics and all support evidence. Any automation fault resets the reliability progression after diagnosis/fix.

- [ ] **Step 5: Hardware-run 20 clean cycles**

Same zero-fault criteria.

- [ ] **Step 6: Hardware-run 100+ consecutive valid cycles**

Acceptance requires:
- 0 wrong-starter confirmations,
- 0 unverified confirmations,
- 0 held-input failures,
- 0 missed valid PK6 reads,
- 0 false shiny classifications,
- 0 unexplained state transitions.

A genuine shiny that correctly enters SHINY_HOLD is a successful safe termination.

- [ ] **Step 7: Commit sanitized reliability summary**

```bash
git add starter/runner.py diagnostics/session_metrics.py probes/probe12_reliability.py docs/hardware/probe-12-reliability.md tests
git commit -m "test: complete Froakie reliability validation"
```

### Task 8: Enable production continuous Froakie mode and optimize from measurements

**Files:**
- Create: `starter/continuous.py`
- Modify: `probe_cli.py`
- Create: `RUN_FROAKIE.bat`
- Create: `docs/hardware/froakie-production.md`
- Test: `tests/starter/test_continuous.py`

**Interfaces:**
- Consumes: successful 100-cycle gate and measured per-state timings.
- Produces:
  - `run_froakie_hunt(...) -> HoldRecord | SessionSummary`.
  - no cycle limit required, but all existing safety/HOLD semantics remain mandatory.

- [ ] **Step 1: Write failing production-gate tests**

Continuous mode refuses to start without a valid 100-cycle gate record matching the current verified profile/state-atlas version.

- [ ] **Step 2: Write failing timing tests**

Optimized debounce/poll values may only reduce waits inside hardware-measured safe ranges; they cannot remove state evidence checks.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/starter/test_continuous.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement continuous mode**

Production logging is compact per cycle; verbose RAM/framebuffer/support capture activates on HOLD, recovery, invalid PK6, selector failure, watchdog timeout, or shiny.

- [ ] **Step 5: Apply timing optimization from CycleMetrics**

Optimize largest measured unnecessary waits first. Re-run bounded reliability tests after every timing-class change.

- [ ] **Step 6: Final verification**

Run: `python -W error::ResourceWarning -m pytest -q`  
Expected: PASS on Windows and Linux CI.

Verify no forbidden binary/private/game artifacts are tracked.

- [ ] **Step 7: Commit**

```bash
git add starter/continuous.py probe_cli.py RUN_FROAKIE.bat docs/hardware/froakie-production.md tests/starter/test_continuous.py
git commit -m "feat: enable production Froakie continuous hunt"
```
