# Pokémon Y Foundation & Reference Probe Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fresh, read-only Pokémon Y backend foundation and standalone reference probe that proves bridge connectivity, game identity, trainer/party candidate reads, PK6 decoding, baseline capture, and deterministic support-bundle output without sending any controller input.

**Architecture:** A small synchronous Python core separates bridge transport, game identity, memory reads, PK6 parsing, and diagnostics. Probe 00/01 compose those modules but remain strictly read-only; hardware-derived addresses are represented as unverified candidates until the probe output proves them.

**Tech Stack:** Python 3.13, standard-library sockets/struct/json/zipfile/hashlib/dataclasses, pytest, GitHub Actions on Windows and Linux.

**Spec:** `docs/superpowers/specs/2026-10-06-pokemon-y-starter-bot-design.md`

## Global Constraints

- Pokémon Y only for this plan.
- English-only first implementation.
- Running game identity must match Title ID `0004000000055E00` and process `kujira-2`.
- Candidate TID/SID address: `0x08C79C3C`; candidate party base: `0x08CE1CF8`; party stride: `484`; stored PK6 size: `232`.
- Fresh hardware proof outranks historical X/Y notes and external reference projects.
- No controller input is permitted in this plan.
- No Qt UI and no appdata writes.
- Unknown/invalid identity or memory state must fail closed.
- Windows/Python 3.13 socket cleanup must be deterministic.
- Raw RAM dumps, support ZIPs, PK6 blobs, game archives, firmware, and save files must remain untracked by Git.
- Production code must not promote a candidate address to verified authority; this plan only records evidence and verification status.

## Review Focus

- Wrong game/process connected: Probe 00 must stop before memory reads beyond identity and report `GAME_IDENTITY_MISMATCH`; Task 3 owns this test.
- Short/truncated bridge reply: transport must raise a typed protocol error and always close the socket; Task 2 owns this test.
- Candidate Party Slot 1 contains random/non-PK6 bytes: Probe 01 must report invalid/unknown, never shiny/valid; Task 5 owns this test.
- Support-bundle creation fails part-way: temporary output must not be mistaken for a completed bundle and the original probe result remains available; Task 6 owns this test.
- Repeated start/stop on Windows/Python 3.13: transport context must close exactly once without unclosed socket/asyncio/proactor warnings; Task 2 owns this test.

---

### Task 1: Repository foundation and immutable candidate profile

**Files:**
- Create: `pyproject.toml`
- Create: `requirements-dev.txt`
- Create: `config/pokemon_y.json`
- Create: `pokemon_y/__init__.py`
- Create: `pokemon_y/profile.py`
- Test: `tests/test_profile.py`
- Create: `.github/workflows/test.yml`

**Interfaces:**
- Consumes: approved design spec only.
- Produces: `PokemonYProfile`, `load_profile(path: Path) -> PokemonYProfile`, and a checked-in JSON profile whose addresses are explicitly marked `candidate`.

- [ ] **Step 1: Write failing profile tests**

Assert exact Title ID, process, TID/SID candidate, party-base candidate, party stride, stored PK6 size, and that candidate addresses are not marked verified.

- [ ] **Step 2: Run profile tests and verify failure**

Run: `python -m pytest tests/test_profile.py -v`  
Expected: FAIL because profile module/config do not exist.

- [ ] **Step 3: Implement the minimal typed profile loader**

Implement:
- `VerificationStatus = Literal["candidate", "hardware_verified"]`
- `AddressCandidate(address: int, status: VerificationStatus, sources: tuple[str, ...])`
- `PokemonYProfile(...)`
- `load_profile(path: Path) -> PokemonYProfile`

Keep profile parsing strict: malformed hex strings, missing keys, or an unsupported verification value raise `ValueError`.

- [ ] **Step 4: Add CI for Python 3.13**

Workflow runs `python -m pytest -q` on `windows-latest` and `ubuntu-latest` using Python 3.13.

- [ ] **Step 5: Run the focused tests**

Run: `python -m pytest tests/test_profile.py -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml requirements-dev.txt config/pokemon_y.json pokemon_y tests/test_profile.py .github/workflows/test.yml
git commit -m "feat: add Pokemon Y candidate profile"
```

### Task 2: Deterministic bridge transport and protocol errors

**Files:**
- Create: `connection/__init__.py`
- Create: `connection/errors.py`
- Create: `connection/transport.py`
- Create: `connection/protocol.py`
- Test: `tests/connection/test_transport.py`
- Test: `tests/connection/test_protocol.py`

**Interfaces:**
- Consumes: host/port supplied by probe CLI.
- Produces:
  - `BridgeTransport(host: str, port: int, timeout_s: float)` context manager.
  - `request(payload: bytes, *, expected_min_bytes: int = 1) -> bytes`.
  - `BridgeProtocolError`, `BridgeTimeoutError`, `BridgeConnectionError`.
  - Protocol helpers `encode_ping() -> bytes` and `parse_ping_reply(data: bytes) -> PingReply`.

- [ ] **Step 1: Write failing transport lifecycle tests**

Tests must prove:
- socket closes once on success,
- socket closes once when recv raises,
- truncated reply raises `BridgeProtocolError`,
- timeout maps to `BridgeTimeoutError`,
- repeated context-manager use does not retain a closed socket.

Use an injected fake socket factory; no live network.

- [ ] **Step 2: Run tests and verify failure**

Run: `python -m pytest tests/connection -v`  
Expected: FAIL because transport/protocol modules do not exist.

- [ ] **Step 3: Implement `BridgeTransport` with dependency-injected socket factory**

Use blocking sockets in this foundation plan to avoid unnecessary asyncio/proactor lifetime complexity. `close()` must be idempotent and invoked by `__exit__` on every path.

- [ ] **Step 4: Implement PING protocol helpers from the existing bridge protocol evidence**

Do not invent packet bytes from memory. During implementation, inspect the canonical Pokebot firmware/previous bridge client reference and pin the exact request/reply structure in protocol tests.

- [ ] **Step 5: Run transport/protocol tests**

Run: `python -m pytest tests/connection -v`  
Expected: PASS with no ResourceWarning when repeated under `-W error::ResourceWarning`.

- [ ] **Step 6: Commit**

```bash
git add connection tests/connection
git commit -m "feat: add deterministic bridge transport"
```

### Task 3: GAME_INFO and read-only Probe 00

**Files:**
- Create: `connection/game_info.py`
- Create: `probes/__init__.py`
- Create: `probes/probe00_connection.py`
- Create: `diagnostics/result.py`
- Test: `tests/connection/test_game_info.py`
- Test: `tests/probes/test_probe00.py`

**Interfaces:**
- Consumes: `BridgeTransport`, `PokemonYProfile`.
- Produces:
  - `GameInfo(title_id: str, process_name: str, raw: bytes)`.
  - `read_game_info(transport: BridgeTransport) -> GameInfo`.
  - `run_probe00(transport: BridgeTransport, profile: PokemonYProfile) -> ProbeResult`.
  - `ProbeResult(status: str, details: dict[str, object], errors: tuple[str, ...])`.

- [ ] **Step 1: Write failing GAME_INFO parsing tests**

Cover a valid Y reply, wrong Title ID, wrong process name, and truncated/malformed reply.

- [ ] **Step 2: Write failing Probe 00 behavior tests**

Assert:
- valid Y returns `PASS`,
- wrong game returns `GAME_IDENTITY_MISMATCH`,
- identity mismatch performs no memory-read call,
- no controller method exists in Probe 00 dependencies.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/connection/test_game_info.py tests/probes/test_probe00.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement GAME_INFO parsing and Probe 00**

Probe 00 performs PING, GAME_INFO, identity verification, and basic read-latency timing only. It never constructs a controller object.

- [ ] **Step 5: Run focused tests**

Run: `python -m pytest tests/connection/test_game_info.py tests/probes/test_probe00.py -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add connection/game_info.py probes/probe00_connection.py diagnostics/result.py tests/connection/test_game_info.py tests/probes/test_probe00.py
git commit -m "feat: add read-only Pokemon Y identity probe"
```

### Task 4: Targeted memory reader, trainer candidate, and party window capture

**Files:**
- Create: `connection/memory.py`
- Create: `pokemon_y/trainer.py`
- Create: `pokemon_y/party.py`
- Test: `tests/connection/test_memory.py`
- Test: `tests/pokemon_y/test_trainer.py`
- Test: `tests/pokemon_y/test_party.py`

**Interfaces:**
- Consumes: `BridgeTransport`, candidate addresses from `PokemonYProfile`.
- Produces:
  - `read_memory(transport, address: int, size: int) -> bytes`.
  - `TrainerIds(tid: int, sid: int)`.
  - `read_candidate_trainer_ids(...) -> TrainerIds`.
  - `PartyWindow(base_address: int, slot_bytes: bytes, surrounding_bytes: bytes)`.
  - `capture_party_window(..., before: int = 0x100, after: int = 0x300) -> PartyWindow`.

- [ ] **Step 1: Write failing memory protocol tests**

Pin exact address/size serialization and exact-length reply requirements. A short read must raise `BridgeProtocolError`.

- [ ] **Step 2: Write failing trainer/party tests**

Use deterministic fake RAM. Assert little-endian TID/SID parsing and that the captured window contains both the exact Slot 1 candidate bytes and surrounding evidence.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/connection/test_memory.py tests/pokemon_y -v`  
Expected: FAIL.

- [ ] **Step 4: Implement targeted memory and candidate readers**

Do not mark addresses verified. Returned diagnostic objects include source address and current profile verification status.

- [ ] **Step 5: Run focused tests**

Run: `python -m pytest tests/connection/test_memory.py tests/pokemon_y -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add connection/memory.py pokemon_y/trainer.py pokemon_y/party.py tests/connection/test_memory.py tests/pokemon_y
git commit -m "feat: add Pokemon Y candidate RAM readers"
```

### Task 5: Gen 6 PK6 decrypt, checksum, and structural classification

**Files:**
- Create: `gen6/__init__.py`
- Create: `gen6/crypto.py`
- Create: `gen6/pk6.py`
- Create: `gen6/shiny.py`
- Create: `tests/fixtures/pk6/README.md`
- Test: `tests/gen6/test_crypto.py`
- Test: `tests/gen6/test_pk6.py`
- Test: `tests/gen6/test_shiny.py`

**Interfaces:**
- Consumes: exactly 232 stored-PK6 bytes; PKHeX semantics as engineering authority.
- Produces:
  - `decrypt_stored_pk6(data: bytes) -> bytes`.
  - `calculate_pk6_checksum(decrypted: bytes) -> int`.
  - `classify_stored_pk6(data: bytes) -> PK6Classification`.
  - `PK6Classification(kind: Literal["empty","valid","invalid"], pokemon: PK6 | None, reason: str | None)`.
  - `PK6` with at minimum EC, checksum, species, TID, SID, PID, nature, ability, gender, IVs.
  - `is_shiny(pid: int, tid: int, sid: int) -> bool`.

- [ ] **Step 1: Create sanitized test vectors**

Derive small legal PK6 fixtures from user-controlled/generated test Pokémon and document provenance in `tests/fixtures/pk6/README.md`. Do not commit live save/RAM dumps.

- [ ] **Step 2: Write failing PK6 tests**

Assert:
- known valid fixture decrypts to expected fields,
- checksum matches expected,
- all-zero slot classifies `empty`,
- random/truncated/non-checksum data classifies `invalid`,
- invalid bytes never produce `is_shiny=True` through the classification API,
- shiny formula matches known shiny/non-shiny vectors.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/gen6 -v`  
Expected: FAIL.

- [ ] **Step 4: Implement PK6 crypto/parser against PKHeX reference behavior**

Keep crypto and semantic parsing separate. The classification API must require exact stored-size input and checksum validity before constructing a valid `PK6`.

- [ ] **Step 5: Cross-check vectors against PKHeX**

Record expected fields in fixture metadata and verify exact parity for the fields this plan decodes.

- [ ] **Step 6: Run tests**

Run: `python -m pytest tests/gen6 -v`  
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add gen6 tests/gen6 tests/fixtures/pk6
git commit -m "feat: add validated Gen 6 PK6 decoder"
```

### Task 6: Probe 01 and atomic support-bundle diagnostics

**Files:**
- Create: `diagnostics/support_bundle.py`
- Create: `diagnostics/json_io.py`
- Create: `probes/probe01_trainer_party.py`
- Test: `tests/diagnostics/test_support_bundle.py`
- Test: `tests/probes/test_probe01.py`

**Interfaces:**
- Consumes: Probe 00 identity result, candidate trainer reader, party window, PK6 classifier.
- Produces:
  - `SupportBundleWriter(root: Path)`.
  - `write_json(relative_path: str, payload: object) -> Path`.
  - `write_bytes(relative_path: str, data: bytes) -> Path`.
  - `finalize(zip_path: Path) -> Path`.
  - `run_probe01(...) -> ProbeResult`.

- [ ] **Step 1: Write failing support-bundle tests**

Assert:
- expected directory names are created,
- JSON is deterministic/UTF-8,
- final ZIP is written via a temporary path then atomically renamed,
- a forced ZIP failure leaves no completed-looking final ZIP,
- raw RAM/PK6 files are under ignored runtime output paths.

- [ ] **Step 2: Write failing Probe 01 tests**

Assert:
- wrong game prevents candidate reads,
- empty Slot 1 reports `EMPTY`,
- random Slot 1 reports `INVALID`, never shiny,
- a valid fixture reports decoded species but candidate address status remains `candidate`,
- result includes TID/SID, addresses used, read sizes, and timing.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/diagnostics/test_support_bundle.py tests/probes/test_probe01.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement Probe 01 and atomic bundle writer**

Bundle layout follows the spec:
`log.txt`, `result.json`, `game_info.json`, plus `ram/`, `pk6/`, `rng/`, `framebuffer/` directories. Empty directories may be represented by manifest entries rather than placeholder binary files.

- [ ] **Step 5: Run focused tests**

Run: `python -m pytest tests/diagnostics/test_support_bundle.py tests/probes/test_probe01.py -v`  
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add diagnostics probes/probe01_trainer_party.py tests/diagnostics tests/probes/test_probe01.py
git commit -m "feat: add trainer-party reference probe bundles"
```

### Task 7: Baseline state capture, CLI launcher, and hardware-safe package

**Files:**
- Create: `pokemon_y/rng_candidates.py`
- Create: `probes/probe02_baseline.py`
- Create: `probe_cli.py`
- Create: `RUN_PROBE.bat`
- Create: `docs/hardware/probe-00-02.md`
- Test: `tests/probes/test_probe02.py`
- Test: `tests/test_probe_cli.py`

**Interfaces:**
- Consumes: Probe 00/01 modules, candidate RNG addresses, support writer.
- Produces:
  - `capture_rng_candidates(...) -> RngCandidateSnapshot`.
  - `run_probe02(...) -> ProbeResult`.
  - CLI choices `00`, `01`, `02`; all are read-only.
  - Windows launcher that invokes the CLI using the active Python 3.13 environment.

- [ ] **Step 1: Write failing Probe 02 tests**

Use fake RAM to assert the baseline bundle includes trainer, party classification, candidate RNG words, exact address provenance, and no controller dependency.

- [ ] **Step 2: Write failing CLI tests**

Assert invalid probe numbers fail before connecting; Probe 00/01/02 dispatch correctly; host/port/timeouts are explicit CLI/config inputs; no default command can send controller input.

- [ ] **Step 3: Run tests and verify failure**

Run: `python -m pytest tests/probes/test_probe02.py tests/test_probe_cli.py -v`  
Expected: FAIL.

- [ ] **Step 4: Implement baseline snapshot and launcher**

Probe 02 captures the canonical pre-starter state evidence available from current candidate addresses. CRO/module and framebuffer hooks may be represented as `unsupported/not_captured` until their exact bridge protocol is mapped in Plan 2; do not fake those fields.

- [ ] **Step 5: Run the entire test suite with warnings promoted**

Run: `python -W error::ResourceWarning -m pytest -q`  
Expected: all tests PASS and no resource warnings.

- [ ] **Step 6: Run static repository safety check**

Run a script/test that fails if tracked paths match forbidden extensions/patterns from `.gitignore` (ROMs, firmware, RAM dumps, saves, support ZIPs).

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add pokemon_y/rng_candidates.py probes/probe02_baseline.py probe_cli.py RUN_PROBE.bat docs/hardware/probe-00-02.md tests
git commit -m "feat: package read-only Pokemon Y reference probes"
```

### Task 8: Hardware execution gate for current Pokémon Y save

**Files:**
- Modify: `docs/hardware/probe-00-02.md`
- Create after successful run: `docs/hardware/evidence/y-reference-probe-summary.md`
- Modify after proof only: `config/pokemon_y.json`

**Interfaces:**
- Consumes: the user's current N3DS, canonical pre-starter Pokémon Y save, existing bridge firmware, Probe 00/01/02 package.
- Produces: sanitized evidence summary and, only when proven, promotion of individual address metadata from `candidate` to `hardware_verified`.

- [ ] **Step 1: Run Probe 00 on hardware**

Expected:
- PING succeeds.
- Title ID exactly `0004000000055E00`.
- Process exactly `kujira-2`.
- No input sent.
- Support bundle produced.

If identity differs, stop this plan and debug before any RAM-address validation.

- [ ] **Step 2: Run Probe 01 on the untouched canonical save**

Expected:
- plausible TID/SID read,
- Party Slot 1 classifies empty/invalid as expected before starter receipt,
- surrounding party window captured,
- no input sent.

- [ ] **Step 3: Run Probe 02 on the same untouched baseline**

Expected:
- stable repeated candidate reads,
- baseline support bundle generated,
- no controller input.

- [ ] **Step 4: Compare at least two repeated baseline runs**

Only promote an address when repeated live evidence and an independently known in-game fact prove it. Record evidence without committing raw RAM.

- [ ] **Step 5: Commit sanitized hardware evidence and verified statuses**

```bash
git add docs/hardware/evidence/y-reference-probe-summary.md config/pokemon_y.json
git commit -m "test: record Pokemon Y reference probe hardware proof"
```

- [ ] **Step 6: Final verification**

Run: `python -W error::ResourceWarning -m pytest -q`  
Expected: all tests PASS.

Verify Git status contains no ROM/update/firmware/save/RAM/support binary artifacts.

