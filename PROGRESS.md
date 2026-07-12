# PROGRESS — Druid

A build log of what shipped and the notable decisions behind it. **Keep it honest** —
this is the working memory between build sessions. The forward-looking plan and
acceptance tests live in [ROADMAP.md](ROADMAP.md); this is the backward-looking "what
got done and why" companion.

**Current phase:** **the core roadmap M0–M8 is complete and confirmed** — every *capability*
is proven (trust spine: Merkle log, signed checkpoints, RFC 3161 anchors, C2SP tile serving,
M8 witness cosignatures; five-layer detection + render collector + scientific datasets;
public product: record, RSS, WASM verify, alerts, search, triage, federated overlay).
**Phase 5–6 (M9–M14) is now the active arc — the "real tool" work**: making Druid actually
*operate* rather than demo. **M9 (polite collection) is built + confirmed** (robots.txt +
per-host rate-limiting/backoff + conditional GET; closes the half-met "polite collection"
hard constraint — proven live against real EPA robots.txt + a real 304). **Next up: M10**
scheduler (`druid run`, continuous re-observation + auto-firing alerts), then M11 WARC/archive
interop, M12 detection precision (pint cross-unit, structure/table-aware diff, rendered-DOM
noise), M13 consistency-proof gossip + OpenTimestamps, M14 R2 store + Cloudflare deploy +
independently-run witness + richer curated set + fuzz/scale tests. **No mocks on any
production path** — prove each milestone live, as M2b–M9 were.

### State of the tree

| Component | File | Status |
|---|---|---|
| Content addressing | `src/druid/hashing.py` | ✅ sha2-256 multihash + verify |
| Blob store | `src/druid/store.py` | ✅ filesystem, content-addressed, sharded, dedups |
| Records / taxonomy | `src/druid/models.py` | ✅ `Observation`, `DiffRecord`, `DiffType` |
| **Trust core (Rust)** | `rust/ledger-core/` | ✅ tlog Merkle log + signed checkpoints + inclusion/consistency proofs + `druid-verify` |
| Witness cosignatures | `rust/…/cosignature.rs`, `src/druid/witness.py` | ✅ C2SP tlog-cosignature (0x04) + quorum verification; `druid cosign` / `--witness --quorum` (M8) |
| Ledger front end | `src/druid/ledger/core.py` | ✅ shells out to `druid-ledger`/`druid-verify` (no FFI) |
| Proof bundle | `pipeline.bundle` + `verify_bundle` (Rust) | ✅ `druid.proofbundle/v1`, offline-verified (M2a) |
| Tile serving | `Ledger::write_tiles` + `druid-verify tiles` | ✅ C2SP tiles published on append; proofs reconstruct from tiles alone (M2c) |
| RFC 3161 anchoring | `rust/…/rfc3161.rs`, `src/druid/anchors.py` | ✅ offline verify (RSA/ECDSA P-256/384/521), real DigiCert+FreeTSA TSAs, pinned roots (M2b-1/2) |
| Static collector | `src/druid/collectors/static.py` | ✅ httpx fetch, injectable `Fetcher`, conditional-GET headers (M9) |
| Render collector | `src/druid/collectors/render.py` | ✅ Playwright headless DOM + captured API/data calls, injectable `RenderEngine` (M3b) |
| Polite collection | `src/druid/politeness.py` | ✅ robots.txt (Disallow + Crawl-delay) + per-host rate-limit + backoff/jitter + conditional GET (304), injectable clock/robots (M9) |
| Differ L0/L1/L2/L4 | `src/druid/differ/` | ✅ normalise + term-watch + numeric (M3a) + tabular (M4a) + NetCDF/HDF + zip/xlsx (M4b) |
| Reviewer aids L3/L5 | `differ/embedding.py`, `triage.py` | ✅ embedding triage + Claude summaries, injectable, outside the trust core (M6) |
| Federated overlay | `src/druid/overlay.py`, `web/…/overlay.astro` | ✅ third-party archives (Wayback CDX) cross-referenced + attested-badging with downloadable bundles (M7) |
| Pipeline | `src/druid/pipeline.py` | ✅ collect → store → diff → append |
| CLI | `src/druid/cli.py` | ✅ `targets`/`observe`/`log`/`verify`/`anchor`/`bundle`/`verify-bundle`/`export` |
| Public record + feeds | `src/druid/web/`, `web/` (Astro) | ✅ `record.json` + RSS + a browsable static site (M5a) |
| In-browser verifier | `rust/ledger-wasm/`, `web/…/verify.astro` | ✅ `ledger-core`→WASM; verifies a bundle in the browser (M5b) |
| Push alerts + search | `src/druid/notify.py`, `web/…/index.astro` | ✅ webhook + email by target/type/severity; client-side search (M5c) |
| Curated data | `data/targets.toml`, `data/terms.toml` | ✅ 3 targets, 10 watched terms |

---

## M9 — Polite collection layer · built + confirmed 2026-07-11

The first Phase-5 milestone: turn "polite by construction" from a *claim* into an enforced
layer. M0–M8 fetched courteously (identifiable UA, bounded timeout, no auth/CAPTCHA) but
only *half*-met the stated hard constraint — no robots.txt, no cross-run rate-limiting, no
conditional GET. M9 closes that gap. It is a **courtesy layer, not part of the trust core**:
it decides only *whether and when* to fetch, never what is attested; a 304/skip never
touches the ledger.

**What shipped.** `src/druid/politeness.py` — a single injectable `PolitenessPolicy` that
wraps a collector's network seam via `.fetcher(inner)` / `.engine(inner)` adapters (drop-in
`Fetcher` / `RenderEngine`):

- **robots.txt** — fetched once per host (cached with a TTL), parsed by the stdlib
  `urllib.robotparser` (no bespoke parsing, per the "no hand-rolled where a stdlib exists"
  rule), honoring `Disallow` **and** `Crawl-delay` for our UA token. A disallowed URL is
  never fetched (`CollectionSkipped`). Missing/unreachable robots is *fail-open* (allow) — a
  documented choice for a small, hand-vetted set of public-domain federal targets; a
  *present* policy is honored strictly. (`.modified()` is called explicitly after `parse()`
  so `can_fetch`/`crawl_delay` never hit RobotFileParser's conservative unread-file default.)
- **per-host rate-limiting** — a minimum interval between requests to a host, raised to the
  host's `Crawl-delay` when declared, enforced through an **injectable clock** (a no-op in
  tests, real sleeping in prod).
- **exponential backoff with jitter** — a transient status (429/5xx) or a network exception
  is retried with a capped, jittered backoff (`base·2^n`, capped, AWS full-jitter);
  persistent transient failure raises `TransientFetchError` rather than logging a bogus 5xx
  observation.
- **conditional GET** — the stored `ETag`/`Last-Modified` for a URL is sent as
  `If-None-Match`/`If-Modified-Since`; a `304` raises `NotModified`, and the pipeline logs
  **no new leaf**. Validators persist to `druid-data/politeness-state.json` so 304 works
  across restarts (`ObserveResult` gained a `status` — `observed`/`unchanged`/`skipped`).

Wired so the **default (production) collectors are polite by construction**: `Druid.__init__`
builds one shared policy across the static + render seams (injected collectors opt out, so
offline tests stay bare). `httpx_fetcher` gained an optional `headers` kwarg (conditional
validators); the `Fetcher` protocol the collector sees is unchanged.

**Proof it works.** 22 new offline tests (fake clock + canned robots/responses): a
`Disallow`-ed path is never fetched; two rapid same-host fetches are spaced ≥ the min
interval; `Crawl-delay: 7` overrides the 1 s floor; different hosts aren't spaced against
each other; a `304` sends the stored validator and yields no new leaf; a `503` retries with
exponential+capped backoff then succeeds; a persistent transient error gives up after
`max_retries` (not logged); validators survive a fresh policy instance. Full suite: **107
Python passed** (was 85), **24 Rust passed**; ruff + mypy + clippy + fmt clean.

**Live (no mocks on the production path).** `python -m druid --data-dir <tmp> observe
epa-ghgrp` fetched EPA's real `robots.txt` (allowed; no Crawl-delay), fetched
`www.epa.gov/ghgreporting` → `[200]` 61 797 bytes with real `ETag W/"1783830368"` +
`Last-Modified`, logged one leaf; an **immediate second observe returned a real `304 Not
Modified` → "no new observation logged"** (the ledger stayed at 1 entry, `verify` VALID),
with the min-interval spacing enforced (2.36 s between requests) and the real validators
persisted to `politeness-state.json`. This is the whole point: re-checking an unchanged page
now costs the server a tiny conditional request and the ledger nothing.

_(Also fixed a pre-existing ASCII violation in `druid log` — a `…` that corrupts to `�` on
piped Windows cp1252 stdout — to `...`, per the standing keep-CLI-ASCII constraint.)_

---

## M8 — Multi-party witness cosignatures · built + confirmed 2026-07-10

The last piece of the trust spine: stop trusting the log operator *alone*. Independent
witnesses co-sign each checkpoint, and a verifier can require a **quorum** — so a
split-view / equivocating log (an operator showing different histories to different people)
is caught. Heuristic-free, C2SP-conformant, no bespoke crypto.

**What shipped.** `rust/ledger-core/src/cosignature.rs` implements C2SP tlog-cosignature on
the same audited `ed25519-dalek` primitive as the log's signed note: the witness key ID
uses algorithm byte **0x04** (`SHA-256(name || 0x0A || 0x04 || pubkey)[:4]`), and a
cosignature is an Ed25519 signature over the exact message `cosignature/v1\ntime <T>\n<note
body>` with the 76-byte line payload `keyID(4) || timestamp(8, big-endian) || sig(64)`
(spec verified against c2sp.org/tlog-cosignature before coding). `verify_bundle` gained
`(witnesses, quorum)`: it counts *distinct* pinned witnesses that validly cosigned the
bundle's checkpoint and rejects if fewer than the quorum. Cosignatures ride the bundle as a
separate `cosignatures` array (ADR-0006) so they don't disturb the anchored checkpoint
bytes — anchoring (time) and cosigning (multi-party) compose cleanly. Tooling: `druid-ledger
cosign` (the kernel produces the cosignature line so the format lives in one place),
`src/druid/witness.py` (independent witness Ed25519 keys via `cryptography`),
`Druid.cosign(witness)` (stores a cosignature per checkpoint, keyed by digest, one per
witness), `druid cosign --name --key-file`, and `druid-verify bundle --witness name:hex
--quorum K`. The in-browser WASM verifier keeps quorum 0 (reports, doesn't require —
enforcement is a native/service policy).

**Verified.** `cargo test` → **24** (+4 cosignature: cosign→verify roundtrip, wrong key
rejected, tampered body rejected, wrong name rejected), clippy `-D warnings` + fmt clean,
wasm target compiles. Python: `ruff` + `mypy` clean, `pytest` → **85** (+5 witness: the
2-of-3 quorum — 0 and 1 cosignatures rejected, 2 validates with "2/2 verified"; an unpinned
witness's cosignature doesn't count; a tampered checkpoint is rejected; quorum 0 stays
backward-compatible; witness keys are distinct + persist stably). **Live** through the CLI
on the real ledger: two witnesses `druid cosign` the checkpoint, then `verify-bundle
--quorum 2` → `VALID … 2/2 witness cosignature(s) verified`, `--quorum 3` → `INVALID witness
quorum not met: 2 of 3`.

---

## M7 — Federated overlay index + verification badging · built + confirmed 2026-07-10

The rescued corpus becomes *queryable with verifiability as the differentiator* (DESIGN
§8): Druid's curated, attested record layered over the far larger volunteer archive
ecosystem, so a searcher sees at a glance which copies come with a proof.

**What shipped.** `overlay.py` — an injectable `ArchiveSource` port (so the harvest is
offline-testable) with a default `WaybackSource` that queries the Internet Archive **CDX
API** (polite: identifiable UA, bounded timeout, read-only, `collapse=digest`); OSF /
Dataverse / Perma.cc / PEDP adapters are the same port over their metadata APIs.
`build_overlay` cross-references third-party captures with Druid's attested observations
(matched by a lenient URL identity key — scheme/`www`/trailing-slash insensitive) into a
`druid.overlay/v1` index: each resource is badged **druid-attested** — carrying a
downloadable proof-bundle reference — when Druid observed it, or left **unverified** when
only a third party archived it. `write_overlay` / `druid overlay` emit `overlay.json` +
`bundles/<target>.json` for every attested resource. A `web/src/pages/overlay.astro` page
renders the badged, client-side-searchable list with bundle-download + verify links; a
clean reproducible sample (deterministic source, real bundles) ships committed like
`record.json`.

**Scope/decisions.** Badging is the point (DESIGN §1): a third-party copy is a *real,
valuable* archive, but Druid can prove nothing about its bytes, so it gets no badge — the
overlay never overclaims. The harvest stays polite and read-only (no crawling; CDX is a
single GET per URL). The overlay build needs network, so it's a separate `druid overlay`
step, not folded into the offline `export`.

**Adversarial review (workflow) caught three real bugs — two of them overclaims — all
fixed.** (1) Two curated targets whose URLs collapse under the lenient key, and (2) a
single target observed at two URLs, both produced overlay rows that **advertised one
observation's hash but referenced a bundle proving a different one** — a direct violation
of "don't overclaim verifiability" (DESIGN §4.2). Root cause: the advertised hash was
keyed per-URL-latest while the bundle proved the *target's* globally-latest observation.
Fixed by keying attestation on the **specific observation leaf** (its ledger index):
identity/hash/index now move together to the latest leaf for each URL, the bundle
reference is `bundles/<leaf-index>.json`, and `write_overlay` generates
`druid.bundle(target, index)` — so the shipped proof attests *exactly* the advertised
hash + URL, and an attested row shows the URL Druid observed (not a third party's
equivalent capture form). (3) A ragged/truncated CDX row raised `IndexError` and aborted
the whole harvest — now short rows are skipped. Regression tests pin the invariant
(every attested bundle proves its advertised hash+URL, incl. a URL-that-moved case) and
the ragged-row skip.

**Verified.** `ruff` + `mypy` clean; `pytest` → **80** (+7 overlay: a resource in both
Wayback and Druid → attested badge + a leaf-keyed bundle; a third-party-only sibling →
no badge; an attested-only resource appears with an empty external list; `WaybackSource`
parses real CDX rows and handles an empty result; `write_overlay` emits a valid
`druid.proofbundle/v1`). **Live** (`druid overlay --sources wayback` on the real ledger):
12 resources, 6 attested — `www.epa.gov/ghgreporting` badged attested with **7 real
Wayback captures back to 2012** + a downloadable bundle, while `ejscreen.epa.gov/mapper`
(Wayback-only) got no badge. The `/overlay` page was driven in a real browser: the badge
list renders, the bundle link resolves to a valid `druid.proofbundle/v1`, search filters
3→2 (attested) / →1 (unverified) / →0 (no match, message shown), console clean. `astro
build` → **13 pages**. Rust untouched (20 tests still green).

---

## M6 — Embedding triage + LLM change summaries · built + confirmed 2026-07-10

The reviewer-aid layers (DESIGN §6.2, L3 + L5): everything here is **triage, not truth** —
best-effort signals stored alongside the attested record, never inside a ledger leaf, so
the trust core is untouched.

**What shipped.** **L3** (`differ/embedding.py`) — an injectable `Embedder` port (default
`sentence_transformer_embedder`, `all-MiniLM-L6-v2`, lazily loaded) drives
`embedding_triage`: it segments a normalised text change into sentences, embeds each
*changed* passage, and scores it against its closest prior passage by cosine — banding into
`ContentEdit [reworded]` / `[new_passage]` (surfaced for review, Medium) when semantically
distant, `CosmeticOnly` for a minor edit, and silence when near-identical. The pipeline
takes an optional `embedder`; when present it **owns** the interpretation of a change L1/L2
didn't explain (replacing the coarse L0 `ContentEdit` fallback), reachable via `druid
observe --embed`. **L5** (`triage.py`) — an injectable `Summarizer` port (default
`claude_summarizer`, Claude Messages API, `claude-opus-4-8`) drafts a plain-language
"what changed and does it alter meaning" summary of a reworded passage; `druid triage
<target>` writes it to a **review sidecar** (`druid-data/review/<hash>.json`) with a
`druid.review/v1` schema and an explicit "not attested, not in the ledger" disclaimer.
Both live behind an optional `triage` extra (sentence-transformers / anthropic).

**Scope/decisions.** L3 is a *signal* — it fires only when the high-precision layers found
nothing, keeping it from muddying attested-looking output; the score's sub-classification
(reworded vs new) is best-effort and human-reviewed. L5 is **reviewers-only and never
attested**: the summary is a sidecar, so an LLM hallucination can never enter the immutable
log. A live Claude call is **billable + outward-facing**, so it's not fired automatically;
`claude_summarizer(client=...)` is injectable and the request shape (model / system / single
user turn / `max_tokens`) is verified against a fake client per the claude-api skill.

**Adversarial review (workflow) caught one real bug — fixed.** With an embedder
configured, `embedding_triage` inspects only *added/reworded* passages, so a pure sentence
**deletion** (or a change confined to sub-4-word sentences the segmenter drops) produced no
L3 finding — and the coarse `ContentEdit` floor only ran in the no-embedder branch, so the
diff annotation was **lost entirely** (the observation leaf is still attested; only the
interpretation leaf vanished, and `druid log`/RSS/`notify` would surface nothing). Enabling
M6's embedder was thus strictly worse for deletions. Fixed: the coarse floor now fires
whenever no layer (L1/L2/L3) itemised a real text change — L3 augments, never removes, the
guarantee. Regression test added (embedder + a pure deletion still yields a `ContentEdit`).

**Verified.** `ruff` + `mypy` clean; `pytest` → **73** (+9 M6: reworded passage surfaced
as `ContentEdit [L3-embedding]`; near-duplicate not a review finding; unchanged text quiet;
pipeline uses L3 when an embedder is present and keeps the coarse fallback when absent; the
L5 summary lands in the sidecar with the **ledger entry count unchanged** and `verify`
still VALID; `summarize_event` returns None when there's nothing to explain; and
`claude_summarizer` builds a correct Messages request via an injected fake client — no SDK,
no network, no billable call). A deterministic bag-of-words fake `Embedder` exercises the
banding offline. **Live semantic confirmation** (installed the `triage` extra,
`all-MiniLM-L6-v2`): a policy-weakening rewrite with *different vocabulary* — "firmly
committed to aggressively reducing greenhouse gas emissions" → "will weigh voluntary,
market-based approaches" — scored cosine **0.443** → `ContentEdit [reworded]` (exactly the
low-lexical-overlap case bag-of-words would miss), while a cosmetic reorder scored **0.964**
→ `CosmeticOnly`, no review finding. Rust untouched (20 tests still green).

---

## M4b — Scientific/geospatial + packed dataset diffing · built + confirmed 2026-07-10

Extends L4 to the formats agencies actually publish scientific data in — NetCDF/HDF —
plus the containers they ship it in (`.zip`, `.xlsx`), catching the same silent
manipulations one layer down.

**What shipped.** `differ/dataset.py` became a **magic-byte format router**:
`detect_format` sniffs `CDF`→netcdf, the HDF5 signature→hdf5, `PK`→xlsx-or-zip (an xlsx
is a zip carrying `[Content_Types].xml` + `xl/`), `{`/`[`→json, else csv; `_route`
dispatches and fails soft to a `MetadataChange` on unparseable/absent-backend payloads.
New `differ/netcdf.py` (`netcdf_diff`, xarray) flags **variable presence** (removed=High,
added=Medium), **dimension-size** shifts, **global + per-variable attribute** changes
(`MetadataChange` — units/fill/provenance), and **per-variable summary-stat** shifts
(mean/min/max, numpy finite-filtered, reusing `distribution_changed`). `.xlsx` →
per-sheet tabular diff (sheet presence + each sheet through the existing column/
distributional logic, scoped `sheet:<n>`); `.zip` → member presence + **recursion** into
each *changed* member via `_route` (so a CSV or NetCDF inside a zip is fully diffed,
scoped `member:<n>`). The tabular CSV/JSON path is factored out unchanged (M4a tests pass
byte-identically). Backends (`xarray`/`scipy`/`h5netcdf`/`h5py`/`openpyxl`) are an
optional **`science` extra**, lazily imported — the tabular + zip paths need none. This
also resolves the M4a note: the seeded `epa-ghgrp-data` `.zip` target now unpacks instead
of reporting "could not parse".

**Adversarial review (workflow) caught seven real bugs — all fixed.** A two-reviewer +
per-finding-skeptic workflow over the diff confirmed and I fixed: (1) json↔csv treated as
*incompatible* formats, so a serialization switch masked a real column-drop — routing now
keys on **handler family** (json/csv both tabular); (2) a **2-3 byte magic prefix
collided** with text (a CSV whose first column is `PK…`/`CDF…` misrouted to zip/NetCDF and
lost its diff) — detection now requires the **full signature** (`PK\x03\x04`, `CDF\x0{1,2,5}`);
(3) nested-zip evidence mis-scoped (inner member `**`-overrode outer) — scope now
**accumulates a path** (`a.zip/data.csv`); (4) `prev` xarray handle **leaked** if opening
`curr` raised — now an `ExitStack`; (5) a NaN attribute spuriously read as changed
(`nan != nan`) *and* wrote a non-standard `NaN` token into the leaf — `_scalar` stringifies
non-finite floats and `Ledger.canonical()` is now `allow_nan=False` (fail-loud, never
commit unparseable leaf JSON); (6) a **silent all-NaN data wipe** of a variable went
undetected — `_var_stats` distinguishes non-numeric (`None`) from all-NaN (`{}`) and a
finite→all-NaN transition is now a `DistributionalShift [High]`; (7) a complex/`datetime64`
attribute crashed the ledger append — `_scalar` coerces any non-JSON-native value to a
stable string. Two touched the trust boundary (5 hardens `canonical`; the rest are the
detection layer).

**Verified.** `ruff` + `mypy` clean; `pytest` → **64** (+17 M4b + 7 review-regression:
handler-family routing on a json→csv column drop; magic-prefix non-collision; nested-zip
member path; all-NaN wipe flagged; NaN attr not spuriously changed; `canonical` rejects
non-finite; `_scalar` JSON-safety; corrupt-curr fails soft). Installed the `science` extra
and ran the live path: two NetCDF versions through the dataset pipeline → `SchemaChange
[High]` (ch4 dropped), `DistributionalShift [High]` (co2 mean 414.2→314.2),
`MetadataChange` (units ppm→ppb), ledger `VALID`. Both NetCDF3 (scipy) and NetCDF4/HDF5
(h5netcdf) backends exercised. Rust untouched (20 tests still green).

---

## M3b — Render collector · built + confirmed 2026-07-10

Detection reaches **JavaScript tools** (EJScreen-class maps/dashboards): the static shell
is nearly empty, so a headless browser captures both the post-JS DOM *and* the page's own
API/data calls — the layer where the real regulatory content actually lives.

**What shipped.** `collectors/render.py` — a `RenderCollector` behind an injectable
`RenderEngine` port (mirrors `static`'s `Fetcher`), so tests exercise it with a fake
engine and **need no browser**. The default `playwright_engine` drives headless Chromium
(polite: identifiable UA, bounded timeout, no auth/CAPTCHA), lazily importing Playwright —
an **optional `render` extra**, so the core install and CI stay light. The collector
attests the **rendered DOM** as the primary artifact (`raw_bytes_hash == rendered_dom_hash`,
so the existing diff/bundle/tile machinery works unchanged and term/numeric watch run on
what a *reader* sees) and captures the page's **XHR/fetch data calls**: each response body
is stored content-addressed and referenced by hash from a canonical
`druid.captured_requests/v1` manifest (`captured_requests_hash`). The pipeline gained a
**collector registry** dispatching on `target.collector` (`_collector_for`), and
`collect()` now returns a `Collected` (observation + primary body + `side_artifacts`) so a
collector can emit extra blobs the pipeline stores. New model field
`Observation.captured_requests_hash` (DESIGN §3 had reserved it). Curated target
`epa-ejscreen` (collector=`render`) added.

**Scope/decisions.** Rendered DOM is the attested body (not the raw shell) — the honest
"what the page showed" for a JS tool. Data calls = XHR/fetch only (not images/CSS/fonts):
the *data*, not the decoration. Response bodies are stored (retrievable/verifiable), the
manifest stays small by referencing them by hash. Robots-awareness + cross-run
rate-limiting still ride with the scheduler (as for `static`). Rendered-DOM
non-determinism (nonces/timestamps in the DOM) can cause cosmetic diffs — an L0/render
normalisation refinement for later. Routing captured API JSON into the L4 dataset differ
is a natural M3b×M4 crossover, deferred.

**Adversarial review (workflow) caught two real bugs — both fixed.** A two-reviewer +
per-finding-skeptic workflow over the diff confirmed: (1) `page.goto(wait_until=
"networkidle")` **throws** on the polling/SSE dashboards this collector targets (idle
never fires → TimeoutError → *no* observation captured — failing hardest on exactly the
JS-heavy pages it exists for; Playwright's own docs discourage networkidle). Fixed to
wait for `domcontentloaded` then a **bounded** networkidle settle that captures whatever
rendered if the window elapses. (2) The "canonical" request manifest recorded calls in
**network-arrival order**, so identical call sets hashed differently (`sort_keys` sorts
dict keys, not list elements). Fixed by sorting `manifest_calls` by a stable key —
`captured_requests_hash` is now genuinely content-determined. (Both medium; neither
touched the trust core.)

**Verified.** `ruff` + `mypy` clean; `pytest` → **45** (+7 render: collect captures
DOM+calls; manifest hash is call-order-independent; pipeline routes render targets &
stores side artifacts; detection fires on the rendered DOM; a render observation is
citable offline; unregistered-collector rejected; **a live real-Playwright test**
rendering a localhost JS page). Installed Playwright 1.61 + Chromium and ran the live
path (with the fixed load strategy): a page whose JS `fetch()`es `/api/scores.json` and
mutates the DOM → the collector captured `benzene=12` in the DOM and the `/api/scores.json`
response body. Live through the CLI pipeline into the real ledger: render observation
`[200]`, injected data in the attested DOM, the API response retrievable by its manifest
hash, and the leaf `VALID … via tiles alone`. Rust untouched (20 tests still green).

---

## M2c — Tile serving · built + confirmed 2026-07-10

The last piece of the M2 citable-proof arc: the log itself becomes **fetchable static
files**, so proofs no longer depend on Druid handing them out — a verifier reconstructs
them from tiles it fetched, trusting only the checkpoint signature.

**What shipped.** `Ledger::append` now publishes the C2SP tile files
(`tile/<h>/<l>/<n>[.p/<w>]`, height 8) beside the ledger via `Tile::new_tiles` +
`tile.read_data` from the `tlog_tiles` crate (API verified against the vendored 0.2
source; the crate is the Go `sumdb/tlog` port, so `TileHashReader` gives authenticated
tile fetching for free). Per the spec's MAY-delete: a wider partial prunes narrower
ones, a completed full tile removes its `.p/` dir. `Ledger::write_tiles(0, size)` (CLI:
`druid tiles`, binary: `druid-ledger tiles`) regenerates everything — the migration for
pre-tile ledgers. Verification: `DirTileReader` (exact partial → full tile → wider
partial fallback, always sliced to the requested width) + `verify_inclusion_from_tiles`
→ `druid-verify tiles --tiles <dir>` takes the inclusion-JSON *without* a proof and
reconstructs it from tile files alone — `TileHashReader` authenticates every tile
against the signed root before use, so a substituted tile is caught, and trust still
reduces to the checkpoint signature. Python: `Ledger.emit_tiles()` /
`offline_verify_from_tiles()`; `druid export` ships `checkpoint` + a mirrored `tile/`
into the site (committed as sample data like `record.json`), so the **static Astro site
is now literally a C2SP-layout tile server** (`/checkpoint`, `/tile/8/0/…`) — the dev
form of "R2/CDN-served".

**Scope call.** Hash tiles only — no entry bundles yet: nothing consumes them (the
record bytes travel inside the proof bundle), and C2SP entry-bundle framing can land
when a consumer exists (e.g. M7 mirroring).

**Verified.** `cargo test` → **20** (+5 tiles: proofs-from-tiles-alone with `hashes` +
`entries.b64` deleted; a 600-leaf log exercising two full level-0 tiles + partials at
two levels, incl. boundary records 255/256, with full tiles pruning their partials; a
flipped tile byte rejected; a wrong record rejected; pre-tile regeneration), clippy/fmt
clean, wasm target still compiles. Python: `ruff` + `mypy` clean, `pytest` → **38** (+4:
e2e pipeline appends publish tiles + verify-from-tiles with the hash file deleted;
tampered tile → INVALID; `emit_tiles` migration; export ships checkpoint + tiles). Live:
`druid tiles` on the real 11-entry ledger → `tile/8/0/000.p/11`; a scratch dir holding
**only** the fetched `tile/` tree → `VALID record 0 included in tree size 11 via tiles
alone, root a67e1f…` (the confirmed checkpoint root); one flipped byte → `INVALID
downloaded inconsistent tile`; `astro build` serves `dist/checkpoint` +
`dist/tile/8/0/000.p/11`.

---

## Confirmation pass — M2a → M5c all confirmed; anchor aggregation fixed (ADR-0005) · 2026-07-10

Ran every unconfirmed milestone's ROADMAP **Test** steps end-to-end on a fresh
`druid-data`; all eight (M2a, M2b-1/2, M3a, M4a, M5a/b/c) pass and are ticked.

**Evidence.** Suites: `ruff` + `mypy` clean, `pytest` **34/34**, `cargo test` **15/15**,
clippy/fmt clean. Live: real `observe` of epa-ghgrp; L1/L2/L4 scenarios replayed through
the real pipeline (injected fetcher, clearly-synthetic `druid.invalid` fixture URLs) →
`TermSubstitution [High]`, `NumericThresholdChange [High]` (10→15 ppb), `SchemaChange
[High]` (dropped column), `DistributionalShift [High]` (re-baseline + row_count); ledger
`VALID 11 entries`. Anchored via **real DigiCert + FreeTSA** (`anchor --tsa
digicert,freetsa`); `bundle` → `verify-bundle` (no `--root`) → `VALID … 2 anchor(s)
verified - existed no later than 2026-07-10T16:40:18Z`; one flipped artifact byte →
`INVALID`; dev-TSA anchor + `--root` → 3 anchors verified. Site: `druid export` → valid
`record.json` (4 targets / 6 events) + 5 well-formed RSS feeds; `astro build` → **12
pages**; `/verify` exercised **through the page's own file-input + button** → verdicts
byte-for-byte identical to native `druid-verify` (VALID + time bound; tampered →
INVALID); home-page search filters 6→1 ("numeric"), →0 + no-results message, reset→6;
console clean. `notify --dry-run` → 13 pending deliveries across 3 subscriptions,
correctly filtered. Committed sample `record.json`/`sample-proof.json` refreshed to the
fresh ledger (same pubkey across record + sample bundle).

**One trust-core fix (surfaced, per ADR-0005).** M2b-2's embedded roots made
`verify_bundle` reject a whole bundle over any anchor it couldn't attribute — a dev-TSA
anchor without `--root` returned `INVALID` even though the inclusion proof and both real
anchors were fine, contradicting M2b-1's documented "UNCHECKED" intent and making bundles
fragile against verifier root-set skew. Fixed to the **C2SP witness model**: an
internally-consistent token whose TSA root isn't pinned (`ERR_UNTRUSTED_ROOT`, the only
distinguishable non-tamper failure) is reported "present but not verified", claims no
time bound, and leaves the verdict to the inclusion proof + verified anchors; **every
other token failure stays fatal** (corruption anywhere rejects the bundle). Unsupported
anchor types are reported the same way instead of silently skipped — no silent drops in
either direction. ROADMAP M2b-1's "pin the wrong root → INVALID" line updated
accordingly; unit-level strictness (`untrusted_tsa_root_is_rejected`) unchanged; pytest
`test_unpinned_anchor_is_reported_not_fatal` covers both halves. M8's cosignatures
inherit these semantics.

---

## M5c — Push alerts + client-side search · built 2026-07-03 (✓ confirmed 2026-07-10)

Closes out the public ship: the record now *pushes* to subscribers and is searchable.

**What shipped.** `src/druid/notify.py` — data-driven **subscriptions**
(`data/subscriptions.toml`: `channel` webhook|email, `dest`, `min_severity`, optional
`targets`/`diff_types`), `matches()` filtering, and `dispatch()` that sends each new
matching diff event over an injectable **webhook** (`HttpWebhookNotifier`, POSTs a
`druid.alert/v1` payload) or **email** (`SmtpEmailNotifier`, builds an `EmailMessage`).
Delivery is **idempotent**: a per-`{subscription}:{event}` key is persisted in
`druid-data/notify-state.json`, so re-runs never re-send, a *new* subscription still gets
its historical events, and only successful sends are marked (failures retry). CLI
`druid notify [--dry-run] [--smtp-host/-port/--email-from]`. The Astro home page gains a
**client-side search** island filtering the change list (by target/type/severity/value).

**Verified.** `ruff` + `mypy` clean; `pytest` → **34** (+7 notify: severity/target/type
matching; dispatch is idempotent; a failed delivery isn't marked + retries; webhook posts
the alert payload; email message built + sent via an injected sender; repo subscriptions
load). Live: `notify --dry-run` on a High 10→15 ppb change lists 2 matching webhook subs
(the email sub is correctly filtered out by its target list). Site: `npm run build` → 8
pages; a headless eval confirmed search filters "numeric"→1, "epa-ghgrp"→3, no-match→0 +
a "no results" message, reset→all; no console errors.

---

## M5b — In-browser WASM verifier · built 2026-07-02 (✓ confirmed 2026-07-10)

The headline demo of Druid's whole thesis: **anyone verifies the record offline, in a
browser, trusting no one.** The proof is transferable and doesn't route through Druid.

**What shipped.** A new `rust/ledger-wasm` crate (cdylib, wasm-bindgen) exposes
`verify_bundle(json) -> String` and compiles `ledger-core` to `wasm32-unknown-unknown`
(the whole crypto stack — cms/x509/rsa/ecdsa/p256-384-521/ed25519/tlog_tiles — is pure
Rust, so it compiles cleanly; `getrandom` gets its `js` backend via a wasm-target dep). It
ships the same pinned DigiCert/FreeTSA roots (`include_str!`), so real-TSA-anchored bundles
verify with nothing extra. `npm run build:wasm` runs `cargo build --target wasm32` +
`wasm-bindgen --target web` into `web/public/wasm/`. A `/verify` Astro page (inline module
island) loads the WASM and verifies a chosen `proof.json` — green check / red cross,
nothing uploaded. Linked from the home hero and every event permalink.

**Verified.** In a real browser (astro dev + preview): the page shows "Verifier ready
(runs offline in your browser)"; a headless eval loaded the WASM, fetched the committed
`sample-proof.json` (a real DigiCert-anchored EPA bundle) and got `VALID … included offline`
(+ time bound), and a tampered-artifact variant got `INVALID artifact bytes do not hash to
…` — **byte-for-byte the same verdicts as the native `druid-verify`**. Adding the wasm
crate to the workspace left the native build/tests green: `cargo build --release` OK,
`cargo test` 15, clippy/fmt clean; Python `pytest` 27 unchanged.

**Toolchain/notes.** Needs `rustup target add wasm32-unknown-unknown` + `wasm-bindgen-cli`
(pinned to the crate's 0.2.126). The generated `web/public/wasm/` (a ~1.2 MB unoptimised
wasm) is **gitignored** and rebuilt via `build:wasm`; a production deploy would `wasm-opt`
it. `sample-proof.json` is committed (public, ~92 KB) so `/verify` has something to check.
Client-side search + webhook/email push are M5c.

---

## M5a — Public record (Astro) + RSS feeds · built 2026-07-02 (✓ confirmed 2026-07-10)

The public ship, first slice: make the record **browsable and subscribable**. Trust spine
solid + detection across four layers, so now it becomes *citable*.

**What shipped.** `src/druid/web/` builds the public record from the ledger:
`build_record` → a `druid.record/v1` JSON (per-target timelines of attested observations +
classified diff events, each keyed by its permanent leaf hash); `feed.py` renders RSS 2.0
(stdlib ElementTree — well-formed, no dep) globally and per target; `druid export --out
<dir>` writes `record.json` + `feed.xml` + `feeds/<id>.xml`. A minimal **Astro 6** site
(`web/`, static) renders it: a home page (recent changes with severity badges + a targets
grid), per-target timelines (observations interleaved with diffs + evidence), and
per-event permalink pages — each carrying the integrity/interpretation boundary in copy
("severity labels are best-effort, human-reviewable, never a verified fact"). This
realises DESIGN §3's Astro choice (read-heavy/static). Data flow: `druid export --out
web/public` + copy `record.json` to `web/src/data/` (the `web` npm `export` script), then
`astro build`.

**Verified.** `ruff` + `mypy` clean; `pytest` → **27** (+3 web: record has targets/
observations/events with 64-hex leaf-hash ids; RSS is well-formed with one item per event;
export writes record.json + feeds). `cd web && npm install && npm run build` → **7 static
pages** (index + 2 targets + 4 events). Preview (astro dev): home + target pages render the
classified changes (TermSubstitution, NumericThresholdChange 10→15 ppb) with badges,
evidence, and permalinks; **no console errors**.

**Notes.** `web/node_modules`, `web/dist`, `web/.astro`, `site-data/` are gitignored;
`web/src/data/record.json` + `web/public/feed.xml` are committed as **sample** data so the
site builds standalone (production regenerates them via `druid export`). Arrows/em-dashes
in the *site* HTML are fine (UTF-8) — the ASCII rule is only for CLI stdout. WASM in-browser
verify + search (M5b) and webhook/email push (M5c) are next.

---

## M4a — L4 tabular dataset diffing · built 2026-07-02 (✓ confirmed 2026-07-10)

The other largely-novel detection capability (DESIGN §6.2): catch *silent dataset
manipulation* — a column quietly dropped, a series re-baselined, a record set truncated.

**What shipped.** A `dataset`-kind target (new `Target.kind`, from `targets.toml`) routes
the pipeline differ to `differ/dataset.py` (pandas) instead of the text layers.
`dataset_diff` parses CSV/TSV/JSON and emits: `SchemaChange` for a column added
(Medium) / removed (High) / retyped (Medium); `DistributionalShift [High]` when a numeric
column's mean/min/max moves (re-baselining/scaling) or the row count changes (truncation),
with before/after stats as evidence. Distributional checks run only on numeric columns —
string columns get schema checks only — which keeps it high-precision. Added `pandas>=2.2`
(installed 3.0.3 + numpy 2.4.6); the through-line to `xarray` for M4b (NetCDF).

**Verified.** `ruff` + `mypy` clean; `pytest` → **24** (+6 dataset: column-removed →
SchemaChange High; re-baselined series → DistributionalShift; truncation → row_count
shift; identical → no diffs; JSON-records parse; pipeline routes dataset targets to L4).
Live: a NOAA-style CO2 CSV with methane dropped + CO2 re-baselined -3 ppm + a row truncated
→ `SchemaChange [High] (methane_ppb removed)` + `DistributionalShift [High]` on co2_ppm
(mean 414.2 → 410.4) + `row_count 3 → 2`.

**Notes for next time.** Truncation shifts an index-like column's stats too (e.g. `year`),
so it also fires a per-column DistributionalShift — correct, not a false positive, but a
future refinement could tag index columns. `.zip`/`.xlsx`/NetCDF aren't parsed yet (they
yield a `MetadataChange "could not parse"`); that's M4b. The seeded `epa-ghgrp-data` target
points at a `.zip`, so a live `observe` on it will note "could not parse" until M4b.

---

## M3a — L2 numeric / threshold extraction · built 2026-07-02 (✓ confirmed 2026-07-10)

With the trust spine hardened (M0–M2b-2), this deepens what Druid *catches* — the
canonical manipulation: a regulatory number quietly edited. (Pulled ahead of M2b-3, whose
offline time bound needs hours of Bitcoin confirmation, and M2c.)

**What shipped.** `differ/numeric.py` extracts numbers-with-units that sit next to a
regulatory keyword (threshold / limit / standard / reporting / MCL / …) with a plausible
unit (an env-units allowlist, or any unit carrying `% / µ °`), keyed on the keyword phrase
+ unit. `numeric_watch` flags a `NumericThresholdChange [High]` when the value for the same
context changes, with evidence `{context, unit, from, to}`. Wired into the pipeline differ
after L1 term-watch. High-precision by design: a number with no regulatory keyword nearby,
or a bare count/year, is ignored — no false positives on prose.

**Decisions.** Regex + keyword-context + unit allowlist rather than an ML NER (dependency-
light, high-precision, deterministic — fits the differ's "high-precision layers" intent).
Cross-unit normalisation (`pint`: 10 ppb == 0.010 ppm) is deliberately deferred — until
then differing units don't match, which only *misses* a re-expression, never false-fires.

**Verified.** `ruff` + `mypy` clean; `pytest` → **18** (+4 numeric: extracts only near a
keyword; flags 10→15 ppb High with from/to evidence; no flag when unchanged; ignores
years/counts). Live: two versions of an EPA page ("reporting threshold … 10 ppb" →
"15 ppb") → `NumericThresholdChange [High] {context: "threshold for benzene is", unit:
"ppb", from: "10 ppb", to: "15 ppb"}`.

---

## M2b-2 — Independent third-party TSAs · built 2026-07-01 (✓ confirmed 2026-07-10)

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

## M2b-1 — RFC 3161 anchor + offline verifier · built 2026-07-01 (✓ confirmed 2026-07-10)

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

## M2a — Self-verifying proof bundle · built 2026-06-30 (✓ confirmed 2026-07-10)

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
