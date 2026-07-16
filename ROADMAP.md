# ROADMAP — Verderer

The milestone checklist. Build the next unchecked milestone in order.

**Rules of the road:**
- Each milestone is an **independently runnable** slice — something actually
  testable end-to-end, not an internal-only refactor.
- Every milestone ends with explicit **Test** steps — the acceptance criteria.
- Build **top-down**: M0–M2 build and harden the trust spine; M3–M4 deepen
  detection; **M5 is the public ship**; M6–M8 are force multipliers.
- Check a box **only after its Test passes**, then add a
  `PROGRESS.md` entry.

See [DESIGN.md §8](DESIGN.md) for the full rationale behind this arc.

---

## Phase 0 — Walking skeleton

- [x] **M0 — Skeleton & it runs.** Python pipeline end-to-end on one target: static
  collector → content-addressed store → a signed hash-chain ledger (the honest M0
  stand-in for M1's Merkle log) → L0 normalisation + L1 term-watch → a CLI
  (`observe` / `log` / `verify` / `targets`). One passing test, lint + typecheck wired.
  **Test:** `pip install -e ".[dev]"`; `python -m verderer observe epa-ghgrp` prints a
  `[200]` observation with a content hash; `python -m verderer verify` prints `VALID`;
  `pytest` → green. _(Verified at scaffold: 7/7 tests, live observe + verify OK.)_

## Phase 1 — The trust spine

- [x] **M1 — Real trust core (Rust `ledger-core` + offline verifier).** Replaced
  `SignedLog` with a Merkle log on the `tlog_tiles` crate (C2SP tlog algorithms): a
  Rust kernel does append-leaf, signed checkpoint (C2SP signed note, Ed25519),
  inclusion proof, and consistency proof, backed by the canonical stored-hash file; a
  standalone `verderer-verify` validates a leaf against a signed checkpoint **offline**.
  Python shells out over stdio (no FFI). _(Confirmed; committed 9570122. HTTP
  tile-file serialization moved to M2c.)_

- **M2 — The citable proof (split into runnable slices).**
  - [x] **M2a — Self-verifying proof bundle.** _Confirmed 2026-07-10._
    `verderer bundle <target> [-o file]` exports a single self-contained
    `verderer.proofbundle/v1` (observation + raw artifact bytes + Merkle inclusion proof
    + signed checkpoint + pinned key); `verderer-verify bundle <file>` validates it fully
    **offline** — artifact bytes hash to the observation, the leaf is included under
    the signed root — trusting neither the source nor Verderer. Built on M1's
    `offline_verify`.
    **Test:** `verderer observe epa-ghgrp`; `verderer bundle epa-ghgrp -o proof.json`;
    `verderer verify-bundle proof.json` → `VALID`; edit any byte of `proof.json` →
    `INVALID`. `pytest` + `cargo test` green.
  - **M2b — External anchoring (split; RFC 3161 offline verification is the spine).**
    - [x] **M2b-1 — RFC 3161 anchor + offline verifier.** _Confirmed 2026-07-10._
      A Rust `rfc3161` verifier (on cms/x509-cert/x509-tsp/rsa/ecdsa) validates a
      timestamp token offline: it binds the token's messageImprint to the checkpoint,
      verifies the TSA signature over the signed attributes, checks the timestamping EKU,
      and chains the signer to a **pinned** root. `verderer anchor` timestamps the current
      checkpoint (a **self-hosted dev TSA** for now — proves the mechanism, not
      independence); `bundle` embeds the token in `anchors`; `verderer verify-bundle
      --root <pem>` reports "anchored no later than T" offline.
      **Test:** `observe` → `anchor` → `bundle` → `verify-bundle --root` → `VALID …
      anchored no later than <T>`; tamper the token → `INVALID`; an anchor whose TSA
      root isn't pinned is reported `not verified` and claims no time bound — the bundle
      stands on its inclusion proof (the C2SP witness model, ADR-0005).
      `cargo test` (incl. openssl-minted token fixtures) + `pytest` green.
    - [x] **M2b-2 — Independent third-party TSAs.** _Confirmed 2026-07-10._
      `verderer anchor --tsa digicert,freetsa` submits over HTTP to real, independent TSAs;
      the verifier **ships their roots pinned**, so those anchors verify with no `--root`.
      The verifier now handles real production tokens (DigiCert RSA-4096; FreeTSA ECDSA
      **P-384 + SHA-512** — curve taken from the key, not the digest). This is the step
      that gives a *real* time bound (self-hosted does not).
      **Test:** `verderer anchor --tsa digicert,freetsa` → `bundle` → `verify-bundle` (no
      `--root`) → `VALID … N anchor(s) verified - existed no later than <T>`; `cargo test`
      verifies committed real DigiCert + FreeTSA tokens and rejects each under the other's
      root. (Live submission is network-gated + skips offline.)
    - [ ] **M2b-3 — OpenTimestamps.** Add an OTS proof + the Bitcoin block header needed
      to bound time offline (a distinct `anchors` entry type). **Test:** an OTS anchor
      validates offline against the carried header; a forged one is rejected.
  - [x] **M2c — Tile serving.** _Confirmed 2026-07-10._ Every append publishes the C2SP
    tile files (`tile/<h>/<l>/<n>[.p/<w>]`, height 8) beside the ledger; `verderer tiles`
    regenerates them for pre-tile ledgers; `verderer export` ships `checkpoint` + `tile/`
    so the static site doubles as a tile server. `verderer-verify tiles` reconstructs an
    inclusion proof from the tile files alone, authenticating every tile against the
    signed root. _(Hash tiles only — entry bundles wait for a consumer; the record
    bytes travel in the proof bundle.)_
    **Test:** the verifier reconstructs an inclusion proof from fetched tiles alone.
    _(Passes: a dir holding only `tile/` → `VALID … via tiles alone`; a flipped tile
    byte → `INVALID downloaded inconsistent tile`; hash-file deletion doesn't matter.)_

## Phase 2 — Detection depth

- **M3 — Numeric extraction + render collector (split).**
  - [x] **M3a — L2 numeric / threshold extraction.** _Confirmed 2026-07-10._
    Extract numbers-with-units in a regulatory context (a limit, standard, threshold,
    reporting cutoff) and flag when the value tied to the same context changes →
    `NumericThresholdChange [High]`. High-precision: a number counts only next to a
    regulatory keyword with a plausible unit, so prose numbers (years, counts) are
    ignored. Wired into the differ after L1. _(Cross-unit normalisation via `pint` —
    10 ppb == 0.010 ppm — is the L2 refinement.)_
    **Test:** a page whose "reporting threshold is 10 ppb" becomes "15 ppb" →
    `NumericThresholdChange [High] {from: 10 ppb, to: 15 ppb}`; a page changing years/
    counts emits nothing. `pytest` green.
  - [x] **M3b — Render collector.** _Confirmed 2026-07-10._ A Playwright headless
    collector (`collectors/render.py`, behind an injectable `RenderEngine` port so tests
    need no browser) captures the post-JS **rendered DOM** as the attested artifact and
    the page's own **API/data calls** (XHR/fetch) as content-addressed side artifacts
    referenced by a canonical request manifest (`captured_requests_hash`). The pipeline
    dispatches on `target.collector`; detection runs on the rendered content. Playwright
    is an optional `render` extra.
    **Test:** a JS-rendered fixture is observed with its data calls captured. _(Passes:
    a fake engine offline + a live localhost JS page rendered by real headless Chromium →
    the fetched `/api/scores.json` body is captured and retrievable by hash, the mutated
    DOM is attested and tile-verifiable.)_

- **M4 — Dataset diffing (split).**
  - [x] **M4a — Tabular (CSV/JSON) schema + distributional diff.** _Confirmed
    2026-07-10._ A `dataset`-kind target routes to the L4 differ (`differ/dataset.py`,
    pandas): column add/remove/retype → `SchemaChange`; a numeric column re-baselined/
    scaled or the series truncated → `DistributionalShift`. High-precision (distributional
    checks only numeric columns).
    **Test:** two dataset versions — a dropped column → `SchemaChange [High]`; a
    re-baselined series → `DistributionalShift [High]`; truncation → a `row_count`
    `DistributionalShift`. `pytest` green.
  - [x] **M4b — Scientific/geospatial datasets.** _Confirmed 2026-07-10._ `dataset_diff`
    became a magic-byte format router (`detect_format` + `_route`): NetCDF/HDF via
    `xarray` (`differ/netcdf.py` — variable presence, dimension sizes, global/per-variable
    attributes, per-variable summary stats), `.xlsx` per-sheet tabular diff, and `.zip`
    per-member diff recursing into each changed member. The scientific backends
    (xarray/scipy/h5netcdf/openpyxl) are an optional `science` extra; the tabular + zip
    paths need none.
    **Test:** a NetCDF with a dropped variable / changed summary stat is flagged.
    _(Passes offline + live: NetCDF3 and NetCDF4/HDF5 with a dropped `ch4` →
    `SchemaChange [High]`, a re-baselined `co2` → `DistributionalShift [High]`, a changed
    `units` attr → `MetadataChange`; zip member-removal + recursion; xlsx column drop.)_

## Phase 3 — The public product

- **M5 — Public record + alerts (split). ★ public ship.**
  - [x] **M5a — Public record (Astro) + RSS feeds.** _Confirmed 2026-07-10._
    `verderer export` builds `record.json` + a global and per-target RSS feed from the
    ledger. An Astro static site (`web/`) renders a home page (recent classified changes
    + targets), per-target timelines (attested observations interleaved with diffs), and
    per-event permalinks — each with the integrity/interpretation boundary stated in copy.
    **Test:** `verderer export` writes valid `record.json` + `feed.xml`; `cd web && npm run
    build` renders the pages; the home + target pages show the classified changes (incl.
    10→15 ppb). `pytest` green.
  - [x] **M5b — In-browser WASM verifier.** _Confirmed 2026-07-10._ `ledger-core`
    compiles to WASM (`rust/ledger-wasm`, wasm-bindgen), shipping the pinned DigiCert/FreeTSA
    roots; the `/verify` page verifies a downloaded `verderer.proofbundle/v1` **entirely in the
    browser** — a green check / red cross, nothing uploaded, trusting neither the source nor
    Verderer. _(Client-side search is a small remaining add.)_
    **Test:** in the browser, a real anchored bundle → `VALID … anchored no later than <T>`;
    a tampered artifact → `INVALID`; matches the native `verderer-verify`. `cargo build
    --target wasm32` + native `cargo test` green.
  - [x] **M5c — Push alerts + search.** _Confirmed 2026-07-10._ `verderer notify`
    delivers each new diff event to matching subscriptions (`data/subscriptions.toml`) over
    **webhook** (POST `verderer.alert/v1`) and **email** (SMTP), filtered by target, diff-type,
    and minimum severity; delivery is idempotent (a per-(sub, event) key persisted in
    `notify-state.json`, so re-runs never re-send and failures retry). Senders are
    injectable → fully offline-testable. The record site gains **client-side search** over
    the changes. _(A FastAPI/SQLite live-query surface is deferred — the static export +
    search covers the read need.)_
    **Test:** `notify --dry-run` lists pending deliveries; dispatch sends matching events
    once (idempotent, failures retry); the site's search filters the change list. `pytest`
    green.

## Phase 4 — Force multipliers

- [x] **M6 — Embedding triage + LLM summaries (reviewer aid).** _Confirmed 2026-07-10._
  **L3** (`differ/embedding.py`, injectable `Embedder`) segments a text change L1/L2
  couldn't explain, embeds each changed passage, and ranks it against its closest prior
  passage: a semantically distant rewrite → `ContentEdit` for review, a near-duplicate
  stays quiet. **L5** (`triage.py`, injectable `Summarizer`, `verderer triage`) drafts a
  plain-language Claude summary of a reworded passage into a **review sidecar**
  (`verderer-data/review/`), clearly labelled best-effort and **never** in a ledger leaf.
  Both are an optional `triage` extra (sentence-transformers / anthropic).
  **Test:** a reworded-but-not-term-flagged edit is surfaced for review with a
  plain-language summary; the trust core is untouched. _(Passes: L3 flags the reworded
  passage as `ContentEdit [L3-embedding]`; the summary lands in the sidecar with the
  ledger entry count unchanged and `verify` still VALID.)_

- [x] **M7 — Federated overlay index + verification badging.** _Confirmed 2026-07-10._
  `overlay.py` harvests third-party archive metadata behind an injectable `ArchiveSource`
  port (default `WaybackSource` — Internet Archive CDX, polite/read-only; OSF/Dataverse/
  Perma.cc/PEDP are the same port) and `build_overlay` cross-references it with Verderer'
  attested observations into a `verderer.overlay/v1` index: a resource in both is badged
  **verderer-attested** with a downloadable proof bundle, a third-party-only copy shows no
  badge. `verderer overlay` writes `overlay.json` + `bundles/`; a `/overlay` Astro page
  renders the badged, searchable list.
  **Test:** search a resource that exists in both Wayback and Verderer → it shows the
  attested badge with a downloadable bundle; an unverified-only resource shows no badge.
  _(Passes offline + live: a real Wayback CDX harvest on the ledger badged
  `www.epa.gov/ghgreporting` attested with 7 real captures + a bundle, while
  `ejscreen.epa.gov/mapper` (Wayback-only) got no badge; the `/overlay` page renders both
  with the bundle link resolving to a valid `verderer.proofbundle/v1`.)_

- [x] **M8 — Multi-party witnesses.** _Confirmed 2026-07-10._ C2SP `tlog-cosignature`
  (`rust/…/cosignature.rs`, algorithm 0x04, `keyID || timestamp || sig` over
  `cosignature/v1\ntime T\n<note body>`) implemented on the same audited Ed25519 primitive
  as the log's signed note — no bespoke crypto. Independent witnesses (`witness.py`, own
  keys) co-sign the checkpoint via `verderer cosign`; bundles carry a `cosignatures` array
  (ADR-0006 — separate from the checkpoint so anchoring stays intact); `verderer-verify
  bundle --witness name:hex --quorum K` requires K distinct pinned witnesses.
  **Test:** with a 2-of-3 witness set, a bundle missing a quorum of cosignatures is
  rejected; a complete one validates. _(Passes offline + live: `--quorum 2` on a
  2-cosignature bundle → `VALID … 2/2 witness cosignature(s) verified`; `--quorum 3` →
  `INVALID witness quorum not met`; an unpinned witness's cosignature does not count.)_

## Phase 5 — From capability to a running, faithful watchdog

M0–M8 proved every *capability* in isolation. Phase 5 makes Verderer **actually operate** —
politely, on its own, and interoperably — so it protects real data instead of demoing.
**No mocks on any production path.** Injected fakes are a *test* device only; every
milestone here is proven against the real thing (real robots.txt, real WARC, real
schedule, real network) as well as offline.

- [x] **M9 — Polite collection layer.** _Confirmed 2026-07-11._ Close the stated hard
  constraint ("robots-aware, rate-limited") that M0–M8 only half-met. A `politeness.py` layer, injected into the
  `static` and `render` collectors behind the existing seams (with an injectable clock +
  robots fetcher so it is fully offline-testable): **robots.txt** fetch/cache/respect per
  host (honor `Disallow` and `Crawl-delay`), **per-host rate-limiting** with a minimum
  interval + **exponential backoff with jitter** on transient errors, and **conditional
  GET** (`ETag`/`If-Modified-Since` → a `304` means *no new observation is logged*). The
  identifiable UA stays; never fetch an auth-walled or CAPTCHA'd resource.
  **Test:** a `Disallow`-ed path is not fetched; two rapid fetches to one host are spaced
  ≥ the min interval (fake clock); a `Crawl-delay` is honored; a `304` yields no new leaf;
  a transient `503` retries with backoff then succeeds — all offline via injected
  clock/robots/fetch, plus one live polite fetch of a real target. `pytest` green.

- [x] **M10 — The scheduler (continuous operation).** _Confirmed 2026-07-12._ The piece
  that turns a manual demo into a watchdog: `verderer run` re-observes the curated set on a **per-target cadence**
  (interval + jitter), persists per-target schedule state (last run, next due, stored
  `ETag`), observes only what is *due* through the M9 polite layer, appends any diffs, and
  **fires the M5c notify pipeline** on each new diff — idempotently, surviving restarts. A
  `--once` mode for cron/systemd and a long-lived loop for a service; a deployment doc
  (systemd unit / cron / Docker). This is what makes M5c alerts fire on their own.
  **Test:** with a fake clock, a due target is re-observed and a changed page produces a
  diff **and** a notify delivery; a not-yet-due target is skipped; schedule state persists
  across a restart; `--once` processes exactly the due set; a failed target is retried,
  not lost. `pytest` green.

- [x] **M11 — Faithful WARC capture + archive interop.** _Confirmed 2026-07-12._ Make
  Verderer genuinely interoperable with the rescue ecosystem (Wayback / End-of-Term / EDGI), not
  self-referential. Each observation writes a standards **WARC** record (request +
  response) via `warcio`; `warc_record_hash` is populated and attested in the observation
  leaf; the raw artifact is recoverable from the WARC. `verderer export` ships the WARCs; the
  record/overlay link them. Populates the reserved field DESIGN §2/§7 promised.
  **Test:** an observation produces a valid WARC whose response payload hashes to
  `raw_bytes_hash`; a warcio-independent reader replays the captured bytes; `warc_record_hash`
  verifies against the stored WARC; the export includes the WARC. `pytest` green.

## Phase 6 — Depth, precision, and production

- [x] **M12 — Detection precision.** _Confirmed 2026-07-12._ Cut the misses and false
  positives found while building the earlier layers. (a) **L2 `pint` cross-unit normalization** — `10 ppb` vs
  `0.010 ppm` is *not* a change, but a real threshold move across units *is* caught. (b)
  **Structure-aware diffing** — preserve block structure (headings, lists, and **tables**)
  through L0 so a single table-cell edit is localized and attributed, not smeared into
  flat-text noise. (c) **Rendered-DOM noise suppression** — strip nonces / CSRF tokens /
  timestamps / session ids before diffing a rendered DOM so re-renders don't false-fire.
  (d) Fix the L4 **index-column** false positive on truncation.
  **Test:** cross-unit-equal values emit no `NumericThresholdChange`, a real cross-unit
  move is flagged; a one-cell table change yields a localized diff naming the cell; a page
  with a rotating nonce yields no diff; a truncated dataset emits no spurious index-column
  `DistributionalShift`. `pytest` green.

- **M13 — Consistency gossip + OpenTimestamps (completes the trust core).** Close the
  last "trust the operator" gaps. (Split like M2b: the consistency half is done; OTS is the
  deferred M2b-3 piece it was folded into.)
  - [x] **M13a — Consistency-proof gossip.** _Confirmed 2026-07-12._ Consistency proofs
    surfaced to verifiers: prove that checkpoint A is a prefix of a later checkpoint B — the
    log never forked, shrank, or rewrote history — via `verderer-verify consistency`, with the
    export/site carrying a rolling consistency chain a client can gossip. The client-side
    verifier binds to a **pinned** public key (a bundle verified under its own key proves
    only internal consistency, not that it is Verderer' log).
    **Test:** a consistency proof between two real checkpoints validates; a forged history (a
    changed leaf, a shorter tree claiming to extend a longer one, or an equivocation — two
    roots at one size) is rejected. `cargo test` + `pytest` green.
  - [ ] **M13b — OpenTimestamps (M2b-3).** An OTS proof + the Bitcoin block header needed to
    bound time **offline** (a distinct `anchors` type), the maximally adversary-resistant
    "existed no later than" anchor. **Deferred, honestly:** unlike the RFC 3161 TSAs (M2b),
    which respond instantly so real tokens can be committed live, a faithful OTS anchor needs
    a *Bitcoin-confirmed* `.ots` proof — hours of confirmation latency — so it can't be
    live-proven in a session without a synthetic fixture (which this arc's "no mocks on a
    production path" rule forbids). Real time bounds already exist via M2b's independent TSAs;
    OTS is the incremental, maximally-adversary-resistant addition, to be done as a focused
    slice with a real confirmed fixture. **Test:** an OTS anchor validates offline against its
    carried header; a forged one is rejected.

- **M14 — Production deployment & scale.** Make it deployable, multi-party for real, and
  proven at scale. (Split into runnable slices; the robustness/scale slice is done first,
  since it needs no credentials and directly serves the "provably correct" thesis.)
  - [x] **M14d-1 — Property/fuzz + scale hardening.** _Confirmed 2026-07-13._ **Property-based
    / fuzz tests** (Hypothesis) prove the differ + the WARC reader are *total* on untrusted
    bytes — they never crash, hang, or mis-behave on adversarial/malformed input, only ever
    return a result or a controlled `ValueError`; plus round-trip and idempotence invariants.
    A **scale test** proves the Merkle log holds its invariants at size — inclusion +
    consistency proofs stay O(log n) (bounded + asserted) and every spot-checked leaf verifies
    against the signed checkpoint — with a 100k-leaf run gated behind `--ignored`. (The scale
    signal is *structural*, not wall-clock: timings are printed, never asserted, so a suspended
    or loaded box can't flake it.)
    **Test:** the differ/verifier survive a fuzz corpus (no crash on arbitrary bytes); the
    100k-leaf log's proofs verify and stay logarithmic. `cargo test` + `pytest` green.
  - [x] **M14a — R2/S3 store adapter.** _Confirmed 2026-07-16._ `store_s3.S3Store` implements
    the `BlobStore` port (dev = filesystem, prod = any **S3-compatible** bucket — R2, Backblaze
    B2, AWS S3, MinIO), selected by environment (`VERDERER_STORE=s3`), so the vendor is a config
    line rather than a code change. One contract suite runs against **every** backend — and CI
    runs a real MinIO service with `VERDERER_REQUIRE_S3=1`, so a missing server **fails** rather
    than skipping (a silent skip in CI is indistinguishable from a pass, which is precisely how
    an S3 regression would otherwise merge green while the docs claimed full coverage).
    **Test:** the pipeline runs unchanged against the S3 adapter — proven **live against a real
    S3 server** (a local MinIO, not a mock): observe → diff → bundle → WARC all round-trip, the
    ledger verifies, and the bundle's artifact fetched from S3 hashes to the attested leaf.
    (Any hosted bucket is the same code path plus `VERDERER_S3_*` credentials.)
  - [ ] **M14b — Read API + Cloudflare deploy** (FastAPI over the record/overlay) + a
    **Cloudflare Pages/Workers deploy** that publishes the site, record, feeds, tiles,
    bundles, WARCs, and **submits each checkpoint to ≥2 independent mirrors + the Wayback
    Machine** (DESIGN §6.3). **Test:** the deploy publishes a live site + a mirrored checkpoint.
  - [x] **M14c — Independently-run `verderer-witness`.** _Confirmed 2026-07-16._ A third party
    actually runs it (fetches the published checkpoint, verifies consistency against its own
    memory of the log, cosigns) — turning M8 from an in-process demo into real multi-party
    gossip. It holds its own key, pins the log key out-of-band, and needs **no operator
    ledger**; it **refuses** to cosign a fork/equivocation/unpinned-key log, and the operator
    files the returned C2SP line (`ingest_cosignature`) without ever holding the witness's key.
    **Test:** a separately-run witness cosigns and a bundle meets quorum (and `--quorum 2`
    fails); an equivocating log is refused. `pytest` green.
  - [ ] **M14d-2 — Richer curated set** — ≥12 justified targets + an expanded term dictionary
    with published criteria. **Test:** the expanded set observes live.

---

**North star:** A skeptical third party can verify, offline and trusting neither the
government nor Verderer, exactly what a source said and when — and Verderer flags the
specific meaningful change, classified and alertable — over a curated set that Verderer
**observes continuously, politely, and interoperably**, deployed for real.

**Status:** the **core roadmap M0–M8 is complete and confirmed** (2026-07-10) — every
capability is proven. **Phase 5–6 (M9–M14) is the "real tool" arc**: it turns those
capabilities into a self-running, polite, interoperable, precise, deeply-verifiable, and
deployed watchdog, filling the gaps M0–M8 deliberately left. **M9–M12, M13a
(consistency-proof gossip), M14c (independently-run witness), and M14d-1 (property/fuzz + scale) are built and
confirmed**; the rest of **M14** (R2 adapter, Cloudflare deploy + mirrors, an independently-run
witness, a richer curated set) is next, with **M13b (OpenTimestamps) deferred** pending a real
Bitcoin-confirmed fixture. Guiding rule for this arc — *nothing mocked on a production path;
prove every milestone against the real thing* (which is exactly why M13b waits rather than
ships a synthetic OTS).
