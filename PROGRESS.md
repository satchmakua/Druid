# PROGRESS — Druid

A build log of what shipped and the notable decisions behind it. **Keep it honest** —
this is the working memory between build sessions. The forward-looking plan and
acceptance tests live in [ROADMAP.md](ROADMAP.md); this is the backward-looking "what
got done and why" companion.

**Current phase:** Phase 0 done (**M0** walking skeleton). Next: **M1** — the Rust
tile-based Merkle log + offline verifier.

### State of the tree

| Component | File | Status |
|---|---|---|
| Content addressing | `src/druid/hashing.py` | ✅ sha2-256 multihash + verify |
| Blob store | `src/druid/store.py` | ✅ filesystem, content-addressed, sharded, dedups |
| Records / taxonomy | `src/druid/models.py` | ✅ `Observation`, `DiffRecord`, `DiffType` |
| Ledger (trust core) | `src/druid/ledger/log.py` | ✅ M0 signed hash-chain + `verify()` — **placeholder for M1 tile log** |
| Static collector | `src/druid/collectors/static.py` | ✅ httpx fetch, injectable `Fetcher`, WARC-less M0 |
| Differ L0 / L1 | `src/druid/differ/` | ✅ normalise + term-watch |
| Pipeline | `src/druid/pipeline.py` | ✅ collect → store → diff → append |
| CLI | `src/druid/cli.py` | ✅ `targets` / `observe` / `log` / `verify` |
| Curated data | `data/targets.toml`, `data/terms.toml` | ✅ 3 targets, 10 watched terms |

---

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
