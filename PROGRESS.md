# PROGRESS — Druid

A build log of what shipped and the notable decisions behind it. **Keep it honest** —
this is the working memory between build sessions. The forward-looking plan and
acceptance tests live in [ROADMAP.md](ROADMAP.md); this is the backward-looking "what
got done and why" companion.

**Current phase:** Phase 1 — **M1** confirmed; **M2a** (proof bundle), **M2b-1**
(RFC 3161 offline verifier), **M2b-2** (independent third-party TSAs) built and
self-verified, awaiting confirmation. Next: **M2b-3** (OpenTimestamps), **M2c**
(tile serving).

### State of the tree

| Component | File | Status |
|---|---|---|
| Content addressing | `src/druid/hashing.py` | ✅ sha2-256 multihash + verify |
| Blob store | `src/druid/store.py` | ✅ filesystem, content-addressed, sharded, dedups |
| Records / taxonomy | `src/druid/models.py` | ✅ `Observation`, `DiffRecord`, `DiffType` |
| **Trust core (Rust)** | `rust/ledger-core/` | ✅ tlog Merkle log + signed checkpoints + inclusion/consistency proofs + `druid-verify` |
| Ledger front end | `src/druid/ledger/core.py` | ✅ shells out to `druid-ledger`/`druid-verify` (no FFI) |
| Proof bundle | `pipeline.bundle` + `verify_bundle` (Rust) | ✅ `druid.proofbundle/v1`, offline-verified (M2a) |
| RFC 3161 anchoring | `rust/…/rfc3161.rs`, `src/druid/anchors.py` | ✅ offline verify (RSA/ECDSA P-256/384/521), real DigiCert+FreeTSA TSAs, pinned roots (M2b-1/2) |
| Static collector | `src/druid/collectors/static.py` | ✅ httpx fetch, injectable `Fetcher` |
| Differ L0 / L1 | `src/druid/differ/` | ✅ normalise + term-watch |
| Pipeline | `src/druid/pipeline.py` | ✅ collect → store → diff → append |
| CLI | `src/druid/cli.py` | ✅ `targets`/`observe`/`log`/`verify`/`bundle`/`verify-bundle` |
| Curated data | `data/targets.toml`, `data/terms.toml` | ✅ 3 targets, 10 watched terms |

---

## M2b-2 — Independent third-party TSAs · built 2026-07-01 (awaiting test)

The step that makes the time bound *real*: anchor against independent operators Druid
doesn't control, and ship their roots pinned so anchors verify offline by default.

**What shipped.** `HttpTsaAnchorer` submits an RFC 3161 query over HTTP (polite UA,
bounded timeout) to real TSAs; `druid anchor --tsa digicert,freetsa` anchors the current
checkpoint against both (best-effort, per-TSA failures reported). The Rust verifier now
**embeds the DigiCert + FreeTSA roots** (`rust/ledger-core/roots/*.crt`, `include_str!`),
so `druid verify-bundle` checks those anchors with no `--root`; `--root` still adds extras
(e.g. the self-hosted dev TSA). `verify_bundle` reports the tightest bound — the earliest
genTime across all verified anchors.

**Real-world hardening (the crux).** Verifying *real* production tokens surfaced two bugs
the self-signed fixture never would:
- **Digest ≠ curve.** FreeTSA signs with an **ECDSA P-384 key and SHA-512** (`ecdsa-with-
  SHA512`). I had derived the curve from the signature digest (SHA-512 ⇒ P-521) — wrong.
  Now the curve is read from the key's SPKI and verification uses a **prehash** (ECDSA
  reduces the SHA-512 hash to the P-384 field). Added `p521` for genuine P-521 too.
- **Signer digest ≠ SHA-256.** FreeTSA's SignerInfo digest is SHA-512; the messageDigest
  cross-check now uses `SignerInfo.digestAlgorithm`, not a hardcoded SHA-256.
DigiCert (RSA-4096, 3-cert chain, cross-signed root) worked first try and exercises the
chain walk.

**Verified.** `cargo test` → 15 (rfc3161 now 8, incl. **committed real DigiCert + FreeTSA
tokens** that verify against their pinned roots and are each rejected under the other's
root — the independence property). `clippy -D warnings` + `fmt` clean. Python: `ruff` +
`mypy` clean, `pytest` → **14** (+1 live DigiCert test, network-gated, skips offline).
Live: `anchor --tsa digicert,freetsa` → 2 tokens; `verify-bundle` (no `--root`) → `VALID
… 2 anchor(s) verified - existed no later than 2026-07-02T10:18:14Z`.

**Fixtures/roots are public-only** (tokens + root **.crt** certs, no keys) so the
`/commit` secret-scan stays quiet; the dev-TSA keys remain in gitignored `druid-data/`.

---

## M2b-1 — RFC 3161 anchor + offline verifier · built 2026-07-01 (awaiting test)

Anchors bind a checkpoint to a *time*, so the bundle can (eventually) claim "existed no
later than T". This slice delivers the genuinely-hard, correctness-critical half: a
from-scratch **offline RFC 3161 verifier in Rust**, proven against real openssl-minted
tokens. Preceded by a research workflow (fan-out + adversarial verification) that pinned
the exact crate stack and caught a would-be compile bug before any code.

**What shipped.** `rust/ledger-core/src/rfc3161.rs` — `verify_rfc3161_token(token, hash,
roots)` parses the CMS `SignedData`, extracts the `TSTInfo` (via the `x509-tsp` crate),
and offline-verifies: (1) the token's messageImprint == the anchored hash; (2) the
signed-attribute cross-checks (messageDigest == hash(eContent), contentType ==
id-ct-TSTInfo); (3) the TSA signature over the DER `SET OF` signed attributes (RSA
PKCS#1v1.5 / ECDSA, dispatched by alg OID, handling the `rsaEncryption`-with-separate-
digest form openssl emits); (4) the signer's `id-kp-timeStamping` EKU; (5) a signature
chain to a **pinned** root; (6) genTime within the signer's validity window. Wired into
`verify_bundle(json, roots)` — anchors are checked against `sha256(checkpoint)`; a pinned
anchor that fails is a hard error, anchors with no pinned root are reported UNCHECKED (the
inclusion proof stands alone). Python: an `anchors.py` port with `OpensslTsaAnchorer`
(a self-hosted TSA), `Druid.anchor()`, bundle embedding, and CLI `druid anchor` /
`druid verify-bundle --root`.

**Key decisions.** (1) **der-0.7 RustCrypto generation** (cms 0.2.3, x509-cert 0.2.5,
x509-tsp 0.1, rsa 0.9.10, ecdsa 0.16.9) — cms/x509-cert have no stable 0.3 yet; a
`TODO(der-0.8)` marks the future migration. No dedicated end-to-end RFC 3161 verify crate
exists, so ~230 lines assembled on audited primitives (no bespoke crypto). (2) **Pinned-
root** verification, not full RFC 5280 path validation (no revocation/name-constraints) —
honest limit for a small pinned TSA set (ADR-0004). (3) **Self-hosted dev TSA** ships so
anchoring works offline now, but it is **not independent** — Druid holds its key, so it's
no defence against Adversary B. Independent third-party TSAs are M2b-2; the code and copy
say so and don't overclaim a real time bound yet. Keys live in gitignored `druid-data/`
(no committed secret — the `/commit` secret-scan would flag one).

**Verified.** `cargo test` → 11 (incl. 4 new `rfc3161_test` against real openssl tokens:
valid verifies + reports genTime, wrong-hash/tamper/untrusted-root each rejected for its
specific reason), `clippy -D warnings` + `fmt` clean. Python: `ruff` + `mypy` clean,
`pytest` → **13** (+2 anchoring: anchor→bundle→verify offline, and wrong-root rejection —
both fully offline via the openssl dev TSA, skipped if openssl/kernel absent). Live:
`observe → anchor → bundle → verify-bundle --root` → `VALID … anchored no later than
2026-07-02T04:12:31Z`; without `--root` → core VALID + "anchor UNCHECKED"; tamper → INVALID.

**Fixtures.** `tests/fixtures/rfc3161/` holds a committed self-issued TSA **root cert +
tokens** (public only, no keys) minted with `openssl ts`. The Python tests generate an
ephemeral dev TSA at runtime (openssl), so nothing secret is committed.

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
