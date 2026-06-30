# Druid — Design Document

> **Thesis.** Druid is a verifiable, tamper-evident watchdog for public environmental data. It continuously observes a curated set of high-value government environmental pages and datasets, cryptographically attests every observation, detects *meaningful* change (definitional manipulation, threshold edits, silent dataset shifts — not byte noise), and emits a public, citable, self-verifying record. The v2 surface is a federated index over the broader rescue-archive ecosystem, using Druid's attested snapshots as the trust spine.
>
> **One-liner.** *An immutable, provable record of what the government's environmental data said — and how it is being changed.*
>
> **Status.** Foundational design. Pre-implementation. Public/open-source track.

---

## 1. Problem & context

Federal environmental information is structurally fragile, and that fragility is currently being exploited. Since January 2025 the documented pattern includes the removal of 8,000+ government web pages and ~3,000 datasets; the National Climate Assessments taken offline and the U.S. Global Change Research Program site (globalchange.gov) shut down; the climate.gov editorial team terminated; EJScreen pulled from the EPA. As important as deletion is **alteration**: definitions swapped, scope statements softened, terminology substituted ("climate change" → "resilience"), and a "Gold Standard Science" directive that critics argue is broad enough to discount inconvenient findings. The February 2026 glyphosate executive order — invoking the Defense Production Act, framed as national security, paired with a farm-bill provision preempting state/local pesticide-warning labels — is the kind of policy shift whose supporting and contradicting data is exactly what tends to move or vanish.

A volunteer rescue ecosystem already exists and does real work: **EDGI** (Environmental Data and Governance Initiative), **PEDP** (Public Environmental Data Partners), the **End of Term Web Archive**, the **Data Rescue Project**, Harvard's Library Innovation Lab (Perma.cc, Dataverse), and the Internet Archive's Wayback Machine. Druid is **complementary to these, not a competitor.** But the ecosystem has two structural weaknesses that define Druid's niche:

1. **It is volunteer-bottlenecked.** EDGI entered this term monitoring ~4,400 pages versus 25,000+ in 2017, with fewer people. Coverage is hand-driven and thin relative to the surface area.
2. **Crawlers preserve pages, not function — and they preserve *availability*, not *integrity*.** Snapshots prove a copy *exists*; they do not, on their own, let a third party *prove* "the EPA's page asserted exactly X on date Y, and this record has not been altered since — including by the archivist." Nobody in the ecosystem treats **provable observation integrity** and **manipulation detection** as first-class, primary features.

That second gap is Druid's reason to exist. Anyone can scrape. Druid's differentiator is that its record is *verifiable by a skeptical third party who trusts neither the government nor Druid's operators*, and that it surfaces the specific, meaningful changes that matter (a threshold moving from 10 ppb to 15 ppb; a definition narrowed; a dataset silently re-baselined) rather than drowning them in diff noise.

**Primary users:** investigative journalists, environmental litigators (this class of data is already cited in litigation), academic researchers, and watchdog organizations — anyone who needs to make a claim about what a government source said *and have it stand up to challenge*.

---

## 2. What Druid is — and is not

**Druid is** an observation pipeline + a verifiable append-only ledger + a semantic differ + a public record with self-verifying evidence bundles, over a curated, high-value target set, with a federated rescue-archive index layered on top in v2.

**Druid is not** a replacement for federal data *generation* (it watches; it does not sense — independent sensing is a different project), not a general-purpose web archive competing with Wayback (it is interoperable via WARC and differentiated by verifiability), not a truth oracle (it attests observations and *flags* meaningful changes; it does not adjudicate intent), and not a breadth play that tries to crawl all of .gov (depth + verifiability over coverage — explicitly avoiding the page-count bottleneck that the existing efforts already hit). Full scope discipline is in §10.

**Durability of framing.** Druid watches *institutional environmental-data fragility* as a general phenomenon, not one administration. This is both more accurate (the failure mode is structural and outlasts any presidency) and protects the project from going brittle the moment the political weather changes. No part of the system is keyed to a named administration.

---

## 3. The verifiability core

This is the heart of the system and the part that must be specified precisely. Everything else is plumbing around it.

### 3.1 Threat model

Druid does not trust the publisher, and — critically — it does not fully trust *itself*. Three adversaries:

- **Adversary A — the data publisher (agency).** May delete, alter, re-define, or back-date content, and may later assert "we never published that." Druid defends with an independent, externally-timestamped, tamper-evident record of what was observed. *Publisher is untrusted.*
- **Adversary B — the Druid operator / insider / future maintainer.** Could try to retroactively fabricate a snapshot to manufacture a fake "the government said X," or quietly suppress an inconvenient observation. **This is the subtle and most important adversary**: if a journalist or court is to trust Druid's record, that record must resist tampering *by the people who run Druid*. This is why the ledger is hash-chained and externally anchored, and (v3) multi-party witnessed.
- **Adversary C — network / MITM / cloaking.** Could feed a single crawler false content, or the publisher could serve different content to a known crawler IP. Mitigated by geographically distributed collectors, recording the served TLS certificate chain, and cross-checking against independent archives. *Mitigated, not eliminated* (see non-properties).

### 3.2 Security properties (stated precisely)

The discipline here is to claim exactly what is provable and label the rest. Druid maintains three property classes:

**Completeness — what Druid asserts.** For each recorded observation `O`, Druid asserts only: *a collector operating as configured received response `R` (content hash `h`) from URL `u` at wall-clock time `t` (upper-bounded by an external anchor), and `R` is faithfully preserved.* Nothing about ground truth at the source beyond what was observed.

**Soundness — what tampering is detectable.**
- Any post-hoc alteration of a stored snapshot → detectable (content-addressing; hash mismatch).
- Any post-hoc insertion, deletion, or reordering in the ledger → detectable (Merkle append-only/consistency proofs between two signed heads).
- Any back-dating of the ledger beyond the granularity of the external anchor → detectable (anchored roots fix "this state existed no later than `T_anchor`").
- With multi-party witnesses (v3), forging an observation requires colluding with a quorum of independent signers.

**Non-properties — explicitly NOT guaranteed (and why this matters).**
- Druid does **not** prove the source served the *same* content to other clients at the same time (a single vantage cannot exclude cloaking; multiple collectors reduce but do not eliminate this).
- Druid does **not** prove the source *originally* published content Druid never observed (coverage gaps are real; the curated target list is a human judgment call).
- Druid proves *"observed at `t`"* and *"existed no later than `T_anchor`"* — it does **not** establish a lower time bound earlier than first observation.
- The *interpretation* of a diff — whether a change is meaningful, deceptive, or benign — is **heuristic and human-reviewable, not a verified property.** (See §6; this separation is load-bearing.)

Overclaiming any of these would be the failure mode that destroys the project's credibility. The non-properties section is a feature, not a disclaimer.

### 3.3 Mechanism

- **Content-addressed storage.** Every artifact (snapshot bytes, rendered DOM, dataset blob) is keyed by a cryptographic digest (SHA-256, stored as multihash for agility). Identity = content. Tampering = a different key.
- **Append-only Merkle ledger.** Observation and diff *records* (metadata + artifact hashes + collector provenance) are leaves in a Merkle tree forming a verifiable, append-only log — the Certificate Transparency / Trillian model, or a purpose-built equivalent. The ledger supports two proofs a third party can check offline: an **inclusion proof** (this record is in the log at this position) and a **consistency proof** (this newer signed head is a strict, append-only extension of that older signed head — i.e., nothing was rewritten between them).
- **External anchoring.** The ledger head is periodically committed somewhere Druid's operators cannot silently rewrite, so even Adversary B's tampering is detectable. Candidate anchors, in increasing strength/cost: signed head published to multiple independent mirrors and to Wayback itself; **RFC 3161** timestamping authorities; **Sigstore Rekor**; **OpenTimestamps** (→ Bitcoin) for a maximally adversary-resistant "existed no later than" proof. Design intent: use *several* simultaneously so no single anchor is a trust or availability chokepoint. **(Open decision — see §14.)**
- **Multi-party witnesses (v3).** Independent collectors gossip and co-sign observations, so a single operator cannot forge the record. Optional but the strongest answer to Adversary B.

The clean line to hold everywhere downstream: **the snapshots and the ledger are *provable*; the differ's interpretation is *best-effort*.** The trust core must never be contaminated by ML heuristics.

---

## 4. Observation layer (collectors)

Targets are heterogeneous, so collection is a pluggable per-type abstraction. Each collector emits a normalized **Observation** record: `{url, collector_type, collector_version, fetched_at, http_status, response_headers, raw_bytes_hash, rendered_artifact_hash?, tls_cert_chain?, warc_record_hash?}`.

- **Static collector** — plain HTTP fetch of HTML/text/PDF. Output stored as **WARC** (the standard format used by Wayback and the End-of-Term crawl). *Interop, not reinvention* — this is what lets the v2 overlay and the existing ecosystem speak to Druid.
- **Render collector** — headless Chromium (Playwright) for JS-heavy pages and interactive tools. Captures both the rendered DOM *and* the underlying API/data calls the page makes, because for a dynamic tool the data feed is the thing that dies when the tool dies.
- **Dataset collector** — content-type-aware fetch for CSV/TSV/JSON, NetCDF/HDF, GeoTIFF, etc. Raw bytes stored content-addressed; lightweight schema/metadata extracted for the differ.
- **API collector** — captures request + full response for documented agency endpoints (data.gov, GHG Reporting Program APIs, etc.).

Crawling is polite by construction: robots-aware, rate-limited, backoff on error, identifiable user-agent. Note that U.S. federal government works are generally public domain, which keeps the archive itself clean from a rights standpoint; the constraint is courtesy and server load, not copyright.

---

## 5. Curated target set

Depth over breadth. v0 ships against ~12 hand-picked, high-value targets, expanded deliberately — never by trying to mirror all of .gov. Initial set:

- EPA climate change pages and the Climate Risk / Adaptation resources
- The National Climate Assessments (all available editions) and the USGCRP record
- **EJScreen** and related environmental-justice mapping/screening resources
- The EPA **Greenhouse Gas Reporting Program** datasets (congressionally mandated → high-signal if altered)
- NOAA climate datasets and any successor pages to climate.gov content
- USDA climate-risk / crop-resilience tools (the farmer-facing resources that were stripped)
- Selected pesticide/herbicide regulatory pages and label/threshold data (directly relevant given the glyphosate EO and label-preemption fight)

Target lists become community-curatable in v3; the curation criteria (mandate status, litigation relevance, prior alteration history, public traffic) are themselves published, mirroring how the existing rescue efforts publish their selection rationale.

---

## 6. Semantic diff engine — the differentiator

Byte-diffing is worthless: every page changes constantly (timestamps, session tokens, rotating banners). The product value is detecting *meaningful* change and classifying it. The engine is layered, and every layer feeds a **typed diff taxonomy** that is the alertable unit.

**Layer 0 — Structural normalization.** Strip nav/footer/scripts/analytics; readability-style main-content extraction; whitespace normalization. Byte-diff on normalized content now catches real text change.

**Layer 1 — Term watch (cheap, high-precision).** A curated dictionary of sensitive terms/phrases whose appearance, disappearance, or substitution is flagged. This catches the *exact* documented manipulations (terminology swaps, definition narrowing) with high precision and near-zero false positives.

**Layer 2 — Numeric / threshold extraction.** NER + units parsing (e.g., `pint`) to extract numbers-with-units in regulatory context and flag changes — a limit moving, a date shifting, a count dropping. High value, fully tractable, and the kind of change that is easy to slip past readers.

**Layer 3 — Embedding similarity (triage, not truth).** Sentence embeddings to score reworded passages; low-similarity edits get ranked up for human review. This is a *signal*, explicitly not a verified property.

**Layer 4 — Dataset diffing (largely novel — almost nobody does this).** For tabular data: schema diff (columns added/removed/renamed/retyped), row-level diff, and **distributional diff** (did the *values* shift — a column silently re-baselined, a series truncated). For NetCDF/geospatial: metadata diff, variable-presence diff, summary-statistic diff. Detecting *silent dataset manipulation* is a real, underserved capability.

**Layer 5 (v3) — LLM-assisted change summarization.** For human reviewers only: "summarize what changed and whether it plausibly alters meaning." Triage and drafting aid, never an authority.

### The diff taxonomy

Every detected change is classified into a typed, severity-scored category. This is what turns "the page changed" into "a regulatory threshold was altered." Initial taxonomy:

| Type | Example | Default severity |
|---|---|---|
| `Deletion` | Page or dataset removed / 404 / redirect-to-home | High |
| `TermSubstitution` | "climate change" → "resilience"; definition narrowed | High |
| `NumericThresholdChange` | Limit 10 ppb → 15 ppb; reporting cutoff moved | High |
| `SchemaChange` | Column removed/renamed/retyped in a dataset | Medium–High |
| `DistributionalShift` | Dataset values silently re-baselined/truncated | High |
| `MetadataChange` | NetCDF variable dropped; provenance fields altered | Medium |
| `ContentEdit` | Substantive prose change (embedding-flagged) | Medium |
| `Reappearance` | Previously removed resource returns (possibly altered) | Medium |
| `CosmeticOnly` | Boilerplate/markup change, no semantic delta | Info (suppressed) |

**The integrity/interpretation boundary, restated:** the *fact that content `A` was observed at `t₁` and content `B` at `t₂`* is provable. The *label* `NumericThresholdChange, High` is the differ's best-effort interpretation, is human-reviewable, and is stored alongside — not inside — the verifiable record. A reviewer can downgrade or re-classify; the underlying attested observations never change.

---

## 7. Public record, alerts, and the self-verifying proof bundle

**The public record** is a browsable, append-only timeline: per-target history, every observation, every classified diff, each with a permanent content-hash identifier and a resolvable URL.

**Alerts** are subscribable by target and by diff-type/severity, via RSS/Atom, webhook, and email. A journalist subscribes to "anything touching the NCAs" or "any `NumericThresholdChange` in EPA water standards." Watchdog orgs wire webhooks into their own systems.

**The standout feature — the self-verifying proof bundle.** For any observation or diff, a user can download a single bundle containing: the snapshot bytes, the Merkle **inclusion proof**, the **consistency proof** against a published head, and the **external anchor proof** (timestamp/Rekor/OTS). A third party can then verify *offline*, with an open-source verifier and **without trusting Druid's live service**, that "Druid observed exactly this content at this URL no later than this time, and the record has not been altered." This is the artifact a litigator attaches to a filing or a journalist links in a story. It is the entire point: trust is transferable and does not route through Druid.

---

## 8. V2 overlay — federated rescue-archive index (idea #4)

The v2 surface sits *on top* of the core and federates the broader ecosystem so the rescued corpus becomes usable. Today that corpus is scattered across Wayback/End-of-Term, PEDP, OSF, Dataverse, and Perma.cc, and even surviving resources (the NCAs) are buried behind agency search.

The overlay harvests metadata via the ecosystem's own interfaces (Wayback CDX API; OSF and Dataverse REST APIs; OAI-PMH where exposed; PEDP catalogs) into one queryable index. **What makes this more than another aggregator is the verification layer:** when the overlay surfaces a resource, it badges whether a **Druid-attested observation** exists (verifiable, with a downloadable proof bundle) versus an unverified third-party copy. The core's trust spine is what differentiates #4 from the dozen aggregators that could otherwise exist — and it is why #1 must come first and #4 is a surface, not a sibling.

Scope-limited "dead-tool resurrection": for one or two flagship killed tools (e.g., an EJScreen-style mapper), the overlay may host a *static* rebuild pointed at archived + attested data and link it from the index. Full, live, general-purpose tool resurrection is **out of scope** — that is its own project (§10).

---

## 9. Architecture & stack

**Shape.**

```
 Collectors ── Observations (WARC / raw bytes, content-addressed)
     │                 │
     │           Object store  (blobs: snapshots, datasets)
     ▼                 │
  Differ ──────────────┤
     │                 ▼
     │         Verifiable ledger  (Merkle append-only log; externally anchored)  ← trust core
     ▼                 │
  Alerts          API + Indexer  ──→  Web UI (public record; v2 overlay search)
                       │
                  Proof-bundle export
                       │
                  External anchors (RFC3161 / Rekor / OpenTimestamps / mirrors)
```

**Recommended stack** (leaning on tools already in your kit where they fit):

- **Collectors & differ:** Python — the ecosystem is unbeatable here (Playwright, `warcio`, `pandas`, `xarray` for NetCDF, `sentence-transformers`, `pint` for units). FastAPI for the API.
- **Metadata DB:** SQLite + **Litestream** (your stack) — the metadata/index workload is read-heavy and a perfect fit; blobs live separately.
- **Blob store:** **Cloudflare R2** / S3-compatible (your stack) for snapshots and datasets.
- **Public UI:** **Astro** (you've already evaluated it) — the public record is overwhelmingly read-heavy and static-leaning, which is exactly Astro's sweet spot; islands only where the timeline/search needs interactivity.

**The one genuine fork — the ledger core: Rust vs. Python.** This is the part where correctness is non-negotiable and a bug undermines the whole value proposition. A **Rust** Merkle-log + verifier core (with the open-source verifier compiled to a small, auditable binary, optionally WASM for in-browser verification) is the stronger choice on the merits *and* the stronger portfolio centerpiece — it puts a provably-correct, independently-auditable trust kernel at the center of the project, which is squarely your differentiator ("systems that stay correct under complexity, and I can prove it"). The cost is a second language and slower initial velocity. The pragmatic alternative is a Python ledger for v0 to ship fast, with a planned Rust rewrite of *only* the ledger/verifier once the design stabilizes. **Recommendation: Python everywhere for v0 to get to a public ship; carve out the ledger+verifier into Rust at v1, when its interface is settled.** Decision logged in §14.

**Why open-source is load-bearing, not incidental.** You cannot ask anyone to *trust* a closed-source integrity tool — the verifier especially must be open and independently auditable for the proof bundles to mean anything. OSS isn't a nice-to-have here; the security model *requires* it. That also makes this an unusually honest portfolio piece: the thing it claims to do, you can read the code and check.

---

## 10. Scope discipline (explicit non-goals)

Stated plainly, because what Druid refuses to do is as much a part of the design as what it does:

- **No claims beyond observation integrity.** Druid attests what it observed; it never claims to prove the source's *original* contents or that other clients saw the same thing. (§3.2 non-properties.)
- **No data generation.** Druid watches; it does not sense. Independent sensor networks are a different, harder, complementary project — out of scope.
- **No truth adjudication.** The differ flags and classifies; humans interpret. Severity labels are heuristic and reviewable, never authoritative.
- **No general-purpose web archiving.** Not competing with Wayback; interoperable (WARC) and differentiated (verifiability). Curated depth, not breadth.
- **No boil-the-ocean coverage.** ~12 curated targets to start, expanded deliberately. The page-count arms race is explicitly someone else's failure mode, not ours.
- **No live, general dead-tool resurrection in v2.** At most one or two static flagship rebuilds; full resurrection is a separate project.
- **No administration-specific coupling.** The system models data fragility as a structural, durable phenomenon.

---

## 11. Failure modes & honest limitations

- **Cloaking against a known crawler.** A publisher can serve clean content to Druid's IP and altered content elsewhere. Mitigated by distributed collectors and TLS-cert recording; not eliminated. Druid honestly attests *what it was served*.
- **Coverage gaps.** If Druid never observed a resource before it changed, it cannot attest the prior state. The curated list is a judgment call and will miss things.
- **Semantic-diff error.** False positives (noise that survives normalization) and false negatives (a meaningful change the heuristics miss). Mitigated by the term-watch/numeric layers being high-precision, embeddings being triage-only, and human review on high-severity classes. The verifiable core is unaffected by differ errors.
- **Anchor dependencies.** TSAs, Rekor, or OTS could be unavailable or deprecated; using several simultaneously avoids a single chokepoint.
- **Operational sustainability.** Continuous crawling, storage growth, and alert volume need funding/automation discipline. Storage of large datasets is the main cost driver (R2 egress economics matter here — a known constraint in your toolkit).
- **Legal/courtesy.** Crawling public-domain .gov content is permissible; the real constraints are politeness, rate-limiting, and not becoming a load problem. No CAPTCHA-bypassing or auth-walled access.

---

## 12. Roadmap

Phased to guarantee a public ship early — the launch is **v0 + early v1**, *not* gated on v2/v3.

- **v0 — Skeleton spike (~10k LOC; the public ship target).** Static collector; content-addressed store; simple hash-chain ledger; normalization + term-watch + numeric diff; RSS alerts; minimal browse UI. The ~12 curated targets. Ships and is immediately useful.
- **v1 — The credible product.** Render + dataset collectors; full typed diff taxonomy incl. schema/distributional; external anchoring (OpenTimestamps + RFC 3161); **downloadable self-verifying proof bundles**; webhook alerts; clean public archive UI; WARC export; **Rust ledger/verifier carve-out**. This is the "this is real" milestone and the portfolio anchor.
- **v2 — The overlay (#4).** Federated index over Wayback/OSF/Dataverse/Perma.cc/PEDP with **verification badging** and unified search; optional 1–2 static flagship dead-tool rebuilds.
- **v3 — Force multipliers.** Multi-party witness/gossip co-signing; embedding + LLM-assisted change triage at scale; geospatial/NetCDF diffing; community-curated target lists and published criteria.

Resist the 50k-line tarpit: ship the v0 public record, gather real diffs against live targets, and iterate in the open. The proof-bundle feature (early v1) is the point at which Druid becomes *citable* — prioritize reaching it.

---

## 13. Positioning

**Differentiator vs. the ecosystem.** EDGI/PEDP/End-of-Term optimize availability and are volunteer-bottlenecked. Druid adds the two things none of them treat as primary: **provable observation integrity** (resistant even to Druid's own operators) and **manipulation detection** (the specific, meaningful change, classified and alertable), delivered as **transferable, offline-verifiable proof bundles**.

**Why it matters.** It converts "a screenshot someone took" into "a record a court or a newsroom can rely on," and it turns "the page changed" into "this threshold was altered, here's the proof." That is a real capability gap in a domain where the data is actively contested.

**Portfolio fit.** Druid is a clean instance of the through-line — *automated systems that stay correct under complexity, and I can prove it* — applied to the correctness of a public record. The Rust trust kernel + open verifier is the kind of artifact that demonstrates rare, checkable depth: the claim it makes is one a reader can audit in the source.

---

## 14. Open decisions

1. **External anchoring mechanism(s).** Recommendation: run several simultaneously (mirrors + RFC 3161 + OpenTimestamps; add Rekor if convenient). Pick the v0 minimum — likely OpenTimestamps + multi-mirror — and expand. *Pending final choice.*
2. **Ledger implementation language/timing.** Recommendation: Python ledger for v0 to ship; Rust ledger + WASM verifier at v1 once the interface stabilizes. *Logged; revisit at v1 boundary.*
3. **Multi-party witnessing scope.** Is v3 gossip/co-signing worth the operational complexity, or is single-operator + strong external anchoring sufficient for the target users? *Defer; gather user feedback first.*
4. **Initial target-list governance.** Solo-curated through v2, community-curated in v3 — confirm the published selection criteria before launch.
5. **Funding/sustainability model for continuous operation** (storage + crawl). Out of scope for the design but a real gating concern for "matters in the real world."

---

*End of design document.*
