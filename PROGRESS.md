# PROGRESS — Druid

A build log of what shipped and the notable decisions behind it. **Keep it honest** —
this is the working memory between build sessions. The forward-looking plan and
acceptance tests live in [ROADMAP.md](ROADMAP.md); this is the backward-looking "what
got done and why" companion.

**Current phase:** Phase 1 — **M1** confirmed; **M2a** (the self-verifying proof
bundle) built and self-verified, awaiting confirmation. Next: **M2b** (external
anchoring), then **M2c** (tile serving).

### State of the tree

| Component | File | Status |
|---|---|---|
| Content addressing | `src/druid/hashing.py` | ✅ sha2-256 multihash + verify |
| Blob store | `src/druid/store.py` | ✅ filesystem, content-addressed, sharded, dedups |
| Records / taxonomy | `src/druid/models.py` | ✅ `Observation`, `DiffRecord`, `DiffType` |
| **Trust core (Rust)** | `rust/ledger-core/` | ✅ tlog Merkle log + signed checkpoints + inclusion/consistency proofs + `druid-verify` |
| Ledger front end | `src/druid/ledger/core.py` | ✅ shells out to `druid-ledger`/`druid-verify` (no FFI) |
| Proof bundle | `pipeline.bundle` + `verify_bundle` (Rust) | ✅ `druid.proofbundle/v1`, offline-verified (M2a) |
| Static collector | `src/druid/collectors/static.py` | ✅ httpx fetch, injectable `Fetcher` |
| Differ L0 / L1 | `src/druid/differ/` | ✅ normalise + term-watch |
| Pipeline | `src/druid/pipeline.py` | ✅ collect → store → diff → append |
| CLI | `src/druid/cli.py` | ✅ `targets`/`observe`/`log`/`verify`/`bundle`/`verify-bundle` |
| Curated data | `data/targets.toml`, `data/terms.toml` | ✅ 3 targets, 10 watched terms |

---

## M2a — Self-verifying proof bundle · built 2026-06-30 (awaiting test)

The "citable" artifact: a single file anyone can verify offline, trusting neither the
government nor Druid. Builds directly on M1's `offline_verify`.

**What shipped.** `druid bundle <target> [--index N] [-o file]` assembles a
`druid.proofbundle/v1` — a self-contained JSON holding the observation record, the raw
response bytes (base64-inlined as an artifact), the Merkle inclusion proof, the signed
checkpoint, and the pinned public key. `druid-verify bundle <file>` (new Rust subcommand
on `ledger_core::verify_bundle`) validates it **fully offline**: each artifact's bytes
hash to the observation's `raw_bytes_hash`, the leaf bytes hash to the claimed leaf hash,
and the leaf is included under a validly-signed checkpoint (reusing `verify_inclusion`).
`druid verify-bundle <file>` is the Python convenience wrapper. The bundle is a single
file by design (artifacts inlined) so it stays dependency-light — no zip/sidecar handling
in the verifier yet.

**Key decisions.** (1) Single self-contained JSON with inlined artifact bytes (not a zip)
— keeps the open verifier tiny (only `serde_json`), fine for HTML/text observations;
datasets will want sidecar/zip later. (2) The `anchors` array is present but empty —
external-anchor verification is **M2b**, so the bundle proves *inclusion under a signed
checkpoint*, not yet *existed-no-later-than* (don't overclaim). (3) Tile-file serving is
**M2c**.

**Verified.** `cargo test` 7/7 (+ the existing suite, `verify_bundle` exercised via
Python), `clippy -D warnings` + `fmt` clean. Python: `ruff` + `mypy` clean, `pytest`
**11/11** (+2: bundle verifies offline; tampered bundle rejected). Live: `bundle
epa-ghgrp` → an 84 KB proof.json; `verify-bundle` → `VALID … included offline`; flip a
byte → `INVALID artifact bytes do not hash to …`.

**Gotcha fixed.** The CLI success line used a `→` (U+2192), which **crashes** on Windows
when stdout is piped (cp1252 can't encode it) — caught in the live run, replaced with
ASCII `->`. Reinforces the standing rule: keep CLI output ASCII (the em dash/ellipsis in
`observe`/`verify` are cp1252-safe; arrows are not).

---

## M1 — Real trust core (Rust ledger-core + offline verifier) · built 2026-06-30 (✓ confirmed)

The M0 hash chain is gone; the ledger is now a genuine Merkle transparency log with an
independent offline verifier — the project's engineering pillar #1.

**What shipped.** A Rust workspace at `rust/` with the `ledger-core` crate (built on the
Cloudflare `tlog_tiles` v0.2 crate — the C2SP tlog algorithms ported from Go's
`sumdb/tlog`) and two binaries. `druid-ledger` appends a leaf (`record_hash` =
SHA-256(0x00‖bytes), RFC 6962), maintains the canonical stored-hash file, and writes a
**C2SP signed checkpoint** (a signed note: body + blank line + `— name base64(keyID‖sig)`,
Ed25519, key ID = SHA-256(name‖0x0A‖0x01‖pubkey)[:4] — implemented from the C2SP
signed-note spec and unit-tested). It also produces **inclusion** and **consistency**
proofs. `druid-verify` has two modes: `log` (recompute the whole tree and check it
against the signed checkpoint — catches tampering of a stored leaf *or* a stored hash)
and `inclusion` (verify a record against a checkpoint **fully offline**, no directory,
no live service — the seed of the M2 proof bundle). Python's `ledger/core.py` owns
canonicalisation and shells out over stdio.

**Key decisions.** (1) Built on `tlog_tiles` rather than hand-rolling Merkle crypto
(no bespoke crypto — CLAUDE.md). (2) Backed by the canonical flat **stored-hash file**;
the literal HTTP **tile-file serialization** (R2/CDN) is deferred to M2, where fetching
tiles from the blob store is the actual need — the trust *properties* (tamper-evidence,
offline inclusion + consistency) are fully delivered now. See ADR-0003. (3) Python↔Rust
over stdio (no FFI) keeps the auditable kernel a small standalone binary.

**Verified.** `cargo test` → **7/7** (signed-note roundtrip/wrong-key/tamper; append +
whole-log verify; offline inclusion; consistency proof; entry-tamper detection); `cargo
clippy -D warnings` clean; `cargo fmt` clean. Python: `ruff` + `mypy` clean, `pytest` →
**9/9** (the 4 ledger-backed tests shell out to the real binaries, skipping if unbuilt).
Live demo: observe an EPA page twice → `TermSubstitution` "climate change"→absent
flagged High; `verify` → `VALID 4 entries`; `offline_verify(0)` → `VALID record 0
included in tree size 4`; corrupt a stored leaf → `INVALID entry 0 hash mismatch`.

**Gotcha for the next session.** The Rust binaries emit UTF-8 (the checkpoint's
signature line carries a U+2014 em dash). Decode subprocess output as UTF-8 explicitly —
`subprocess(text=True)` uses the Windows locale (cp1252) and silently corrupts the note,
which a unit test caught. The trust kernel must be built before the pipeline runs:
`cargo build --release --manifest-path rust/Cargo.toml`. `Cargo.lock` is committed
(reproducible binaries); `rust/target/` is ignored.

## M0 — Walking skeleton · built 2026-06-30 (✓ verified at scaffold)

The thinnest end-to-end slice runs: **observe → content-address → diff → tamper-evident
log → verify**, as a Python CLI over one of three curated EPA/USGCRP targets.

**What shipped.** A ports-and-adapters skeleton (DESIGN §5): a content-addressed blob
store (sha2-256 multihash, sharded filesystem, dedups); an `Observation`/`DiffRecord`
data model with the full `DiffType` taxonomy enum; a static collector built on `httpx`
behind an injectable `Fetcher` port (so collection is testable with no network); an L0
structural-normalisation + L1 term-watch differ that emits typed, severity-scored
`TermSubstitution`/`ContentEdit` diffs; and a `druid` CLI (`targets`, `observe`, `log`,
`verify`).

**The trust core, honestly scoped.** M0's ledger is a **signed append-only hash chain**
(`SignedLog`): every entry hash-links to the previous, and the head `(size, head_hash)`
is Ed25519-signed, so altering/inserting/reordering any line is detectable and a third
party can pin the state with the public key. It is explicitly **not** the real thing —
no Merkle proofs, no external anchoring, single key. **M1 replaces it wholesale** with
the Rust `ledger-core` (C2SP tlog-tiles + checkpoints + offline verifier). This keeps
the M0 promise truthful while standing up the end-to-end shape.

**Key decisions.** Python-only for M0 to ship the skeleton fast; Rust enters at M1 where
correctness is load-bearing (see `docs/adr/0002`). Differ interpretation is stored as
its own leaves *alongside* observations, never inside them — the integrity/interpretation
boundary is structural, not a convention. Multihash and canonical-JSON hashing are
hand-rolled (tiny, dependency-free) rather than pulling a library.

**Verified.** `ruff check .` clean; `mypy src` clean (16 files); `pytest` → **7/7**
including an end-to-end test (observe twice → a `TermSubstitution` is detected) and a
**tamper test** (corrupting a stored leaf makes `verify()` fail). Live run:
`druid observe epa-ghgrp` → `[200]`, content `1220525c97bc…`, stored; `druid verify` →
`VALID, 2 entries`. Installs clean on Python 3.11.9 with httpx 0.28.1, cryptography
49.0.0, beautifulsoup4 4.15.0 (dev: pytest 9.1.1, ruff 0.15.20, mypy 2.1.0).

**Gotchas for the next session.** `data/` is resolved from the repo root (`parents[2]`
of `cli.py`) — fine for the editable install, revisit if the package is ever installed
standalone. The CLI prints a `…`/`—`; Windows consoles handle it, but keep new
machine-readable output ASCII. DESIGN targets Python 3.12+; the dev machine has 3.11.9,
so code stays 3.11-compatible (no `type` alias statements) and `requires-python` is `>=3.11`.
