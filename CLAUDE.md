# Before You Sign — build brief for Claude Code

## What we are building

A public-records lookup for renters, for one county: Fairfax County, Virginia. For any
address it shows the owner of record, the owning company's state registration, and county
code-enforcement history, each row with its source and the date we read it.

It publishes records, never verdicts. Every design rule below exists to keep that true and
to keep the company out of court. The rules are requirements with tests, not copy.

The owner of this repo is a data scientist (10 years, MS CS/Cybersecurity). Write
production code they will maintain: typed, tested, documented. No demo shortcuts.

---

## Non-negotiables

Eleven rules. Each has a test named in `/docs/DISPLAY_RULES.md`. A rule without a passing
test is not enforced, and an unenforced rule blocks `pipeline publish`.

1. **Provenance or it doesn't ship.** Every displayed fact traces to a `provenance` row with
   a source URL, retrieval time, content hash, and stored raw copy. The publish step fails
   if any record lacks one.

2. **Never render a tenant's name anywhere.** A redaction guard runs before publish and
   blocks the row on any hit.

3. **Individual owners are natural persons and are treated as such.** Where
   `owners.kind = 'individual'`, the page shows ownership and deed facts only.
   Code-enforcement cases are hidden, exactly as they are for sub-threshold buildings, and
   the "Not shown here yet" section says so. This rule may only be relaxed by a written
   decision from counsel recorded in `/docs/DECISIONS.md`, and relaxing it requires
   changing the test, not a config flag.

4. **No judgment, in words or in color.** No scores, grades, rankings, percentiles,
   comparisons, predictions, or color used as judgment. Colors are allowed only for
   open/closed status.

5. **Owner→company links are exact-name or hand-confirmed.** No fuzzy matching. When
   unconfirmed, render the exact sentence: "We could not confirm the company behind this
   owner name."

6. **No implied common ownership.** Two parcels are never presented as sharing an owner
   except through a confirmed `owner_entity` row. `owners` rows are recorded name strings,
   not real-world people: `name_norm` collides across unrelated owners, and the site must
   never treat a collision as an identity. A company page lists a parcel only via a
   confirmed link.

7. **Small buildings show ownership only.** Buildings under `SMALL_BUILDING_THRESHOLD`
   (default 4 units) hide code cases. A NULL or unknown unit count counts as below the
   threshold.

8. **Every page has a "Not shown here yet" section** listing court records and renter
   outcomes. It cannot be removed by config.

9. **Nothing a visitor types is stored, with one named exception.** Search runs client-side
   over a static index. The only thing a visitor types that we keep is a phone number they
   explicitly submit to the watch feature. The only network events from the page are
   anonymous counters and that opt-in phone number. Watch records are deleted on STOP and
   auto-expire after `WATCH_RETENTION_DAYS` (default 180). The privacy page states plainly
   that a watch record links a phone number to an address and is subject to legal process.

10. **No LLM calls at runtime.** LLMs may be used offline in the pipeline only for OCR
    cleanup, and every such row is marked `extraction_method="ocr"` and `checked_by=null`
    until a person confirms it.

11. **The "Concept" banner stays on every page until `PUBLIC_LAUNCH=true`.** That flag also
    requires `LEGAL_REVIEW_DATE` to be non-empty; `PUBLIC_LAUNCH=true` with an empty
    `LEGAL_REVIEW_DATE` is a hard build failure, not a warning.

**Out of scope for v1, do not scaffold:** court records, tenant notes, AI entity matching,
grounded Q&A, email/scam check, Germany.

---

## Decisions that must be made before code is written

These are recorded in `/docs/DECISIONS.md` with a date and a rationale. Claude asks the
owner; Claude does not pick for them.

- **D1 — Coverage.** Fairfax has on the order of 350,000 parcels. One static page per parcel
  plus a client-side index over all of them collides with the ≤60 KB JS budget, with build
  time and memory, and with host per-deployment file caps (Cloudflare Pages has historically
  capped at 20,000 files — verify the current number as part of this decision, do not take
  it from memory). Two acceptable answers:
  (a) v1 covers only parcels with `units >= SMALL_BUILDING_THRESHOLD`, stated plainly on
      the site — this is also the legally narrower product, and it is the default
      recommendation; or
  (b) full county, with the search index sharded by address prefix and lazy-loaded, on a
      host whose file cap the decision record names.
  Nothing in `/web` is built until D1 is answered.

- **D2 — Individual owners.** Rule 3 above is the default. Counsel confirms or amends it.

- **D3 — Review capacity.** The redaction guard (below) is designed to over-block, which
  means a human confirms code cases by hand. Before milestone 7, the owner states who does
  that review and roughly how many cases per week. If the answer is "nobody yet," the code
  enforcement section ships empty behind the "Not shown here yet" copy, and that is a
  correct outcome, not a bug to work around.

- **D4 — Source access.** Settled by milestone 2 below.

---

## Repo layout

```
/pipeline   Python 3.12, uv, fully typed (mypy strict). Extractors, normalizer, matcher, diff, quality gate, publish.
/web        Astro (static output), TypeScript strict, no UI framework. Deploys to Cloudflare Pages.
/worker     Cloudflare Worker, TypeScript. Watch alerts via Twilio + KV; anonymous event counters.
/design     Reference HTML mockup (owner will drop it here). Match its layout, type, and tone exactly.
/docs       DISPLAY_RULES.md, DECISIONS.md, ADDING_A_COUNTY.md, DATA_DICTIONARY.md, INCIDENT_RESPONSE.md, SOURCES.md
/tests      Pytest for pipeline; Vitest for web + worker.
```

---

## Data sources

Discover current endpoints yourself and record them in `/docs/SOURCES.md` with: URL, access
method (API / bulk download / HTML), terms-of-use URL, robots.txt status, rate limit we
apply, and date verified. Do not hardcode URLs from memory without fetching them first.
Prefer official APIs and bulk downloads over HTML scraping. If a source is served by a
vendor portal (Tyler, Accela, etc.), read its terms before writing an extractor and note any
restriction in SOURCES.md; stop and ask the owner if automated access is prohibited.

1. **Fairfax County real estate / land records** (assessment + deed): owner of record as
   recorded, deed recorded date, mailing address on deed, assessed value, parcel ID, units,
   year built, property type, coordinates. Look for the county open-data portal and ArcGIS
   REST services first.

2. **Virginia State Corporation Commission** (Clerk's Information System): entity name, SCC
   ID, status, formation date, registered agent name and address, principal office. Expect
   terms that restrict automated access and a paid bulk-data product as the sanctioned path
   — read the terms before writing anything. If scraping is permitted, scrape at
   ≤1 request/2s with a persistent cache and a descriptive User-Agent that includes a
   contact email.

3. **Fairfax County Department of Code Compliance**: cases by parcel or address: opened
   date, case number, inspector description, status, closed date. This is the source most
   likely not to exist as a public feed. If it is FOIA-only, say so and stop — that outcome
   changes rules 3 and 7, the alerts feature, and half the page structure, and the owner
   decides what the product becomes. If case text lives only in scanned PDFs, add an OCR
   step (tesseract via `pytesseract`) and set `extraction_method="ocr"`.

Every extractor: fetch → store raw copy immutably (`./raw/<source>/<yyyy-mm-dd>/<id>.<ext>`
in dev, Cloudflare R2 in prod, keyed the same way, with SHA-256 in provenance) → parse →
normalize → upsert. Extractors are idempotent and resumable.

---

## Schema (Postgres, SQLAlchemy 2.x + Alembic)

```
parcels(parcel_id PK, address_norm, address_display, unit_field, units INT NULL,
        year_built INT, property_type, lat, lon, county='fairfax_va')

owners(owner_id PK, name_as_recorded, name_norm, mailing_address_as_recorded,
       mailing_address_norm, kind ENUM[company, individual, trust, government, unknown])

parcel_owner(parcel_id FK, owner_id FK, deed_recorded_at DATE, deed_instrument_no,
             is_current BOOL NOT NULL, source_ref FK,
             PRIMARY KEY(parcel_id, owner_id, deed_recorded_at, deed_instrument_no))

entities(entity_id PK, scc_id UNIQUE, name, name_norm, status, formed_at,
         registered_agent, registered_agent_address, principal_office, source_ref FK)

owner_entity(owner_id FK, entity_id FK, match_method ENUM[exact_name, manual],
             confirmed_by TEXT, confirmed_at TIMESTAMPTZ, PRIMARY KEY(owner_id, entity_id))

code_cases(case_id PK, parcel_id FK NULL, address_raw TEXT NOT NULL, opened_at DATE,
           case_number, description_raw, description_public, status ENUM[open, closed, unknown],
           closed_at DATE, source_ref FK, extraction_method ENUM[api, html, ocr],
           checked_by TEXT, redaction_flag BOOL)

provenance(source_ref PK, source_name, source_url, retrieved_at TIMESTAMPTZ, raw_path, content_hash)

snapshots(parcel_id FK, taken_at TIMESTAMPTZ, content_hash, PRIMARY KEY(parcel_id, taken_at))

disputes(dispute_id PK, parcel_id FK, record_table, record_id TEXT, reason, submitted_at,
         status ENUM[open, corrected, upheld, withdrawn], resolved_at, note)
```

Rules encoded in the DB layer, each with a test:

- `owner_entity.match_method='exact_name'` may only be inserted when
  `owners.name_norm == entities.name_norm`. Enforce in a check function and in a test.
- `owner_entity.match_method='manual'` requires `confirmed_by` and `confirmed_at` non-null.
- `code_cases.description_public` is derived from `description_raw` by the redaction guard;
  the publisher reads only `description_public`.
- `code_cases.parcel_id` is nullable because case addresses do not always resolve. Every
  extraction run writes `quality/unmatched_cases/<date>.csv` listing unresolved
  `address_raw` values. A silent drop is a bug.
- `parcel_owner.is_current` is set by the extractor, not derived at read time from
  `max(deed_recorded_at)` — corrective deeds and same-day recordings break that derivation.
  Exactly one current row per parcel; enforce with a partial unique index and a test.
- `disputes.record_table` has a check constraint enumerating the allowed table names.

---

## Normalization

- **Addresses:** USPS-style normalization (street-suffix abbreviations, directionals, unit
  separated into `unit_field`, ZIP+4 dropped). Use `usaddress` + `scourgify` or equivalent;
  both are lightly maintained, so pin versions and vendor stubs for mypy strict. Keep the
  original string. Add 50 fixture cases from real Fairfax address formats.

- **Owner/entity names:** uppercase, strip punctuation, collapse `L.L.C.`/`LLC`,
  `INC.`/`INC`, `CORP.`/`CORPORATION`, drop trailing `THE`. Used for exact matching only.
  Log every match decision to `logs/matching.jsonl`.

Normalization narrows nothing else. `ABC PROPERTIES OF VIRGINIA LLC` and
`ABC PROPERTIES LLC` are different names and stay different names.

---

## Redaction guard (`pipeline/redact.py`)

Runs over `code_cases.description_raw` before anything is published, and over every alert
summary and SMS body before it is sent. The publish path is not the only path.

- Patterns: capitalized two-token names not in an allowlist of street/company words;
  "tenant", "occupant", "resident", "complainant" followed by a name; phone numbers; email
  addresses; apartment/unit occupants' names in any form.
- On a hit: set `redaction_flag=true`, replace the span with `[redacted]` in
  `description_public`, and exclude the row from publish until `checked_by` is set.

**The guard is designed to over-block and it will.** "Front Yard", "Fire Escape",
"Building Code", inspector names, and most street names all trip the two-token rule. That is
the intended bias — a false positive costs a human review, a false negative publishes a
tenant's name. Do not tune the guard toward precision to make more pages render.

Tests:
- At least 30 adversarial fixtures (names that look like street names, street names that
  look like people, hyphenated names, names in quotes).
- A measured false-positive rate against a real sample of at least 200 case descriptions,
  reported by `pipeline quality report` and recorded per run. This number drives D3, so it
  has to be observed rather than assumed.

---

## Quality gate (`pipeline quality ...`)

- `pipeline quality sample --n 200 --source <name>` writes `quality/<source>/<date>.csv`
  with record ID, every displayed field, source URL, raw path.
- `pipeline quality record --id <id> --ok | --bad --note "..."` appends to
  `quality/verdicts.jsonl`.
- `pipeline quality report` prints accuracy per source for the latest sample, plus the
  redaction guard's false-positive rate and the owner→entity match rate.
- `pipeline publish` refuses to run unless every source has a sample ≥200 within 30 days
  with accuracy ≥99%, zero rows with `redaction_flag=true` and `checked_by=null`, zero
  displayed facts without provenance, and every non-negotiable's test passing. Print the
  blocking reason.

---

## Matching (`pipeline match ...`)

- `pipeline match exact` links owners to entities on `name_norm` equality only.
- `pipeline match review --owner <id>` shows the owner name, mailing address, candidate SCC
  entities, and their source URLs, and takes `--confirm <entity_id> --by <name>` to write a
  `manual` link. This CLI is a milestone 6 deliverable, not a later addition: exact matching
  alone will confirm a small minority of owners, and without a hand-confirmation path
  coverage can never rise.
- `pipeline match report` prints confirmed / unconfirmed counts. Expect "We could not
  confirm the company behind this owner name" to be the common answer at first. That is the
  rule working, not a defect.

---

## Diff and alerts

- `pipeline snapshot` hashes each parcel's public content weekly and writes `snapshots`.
- `pipeline diff` emits `alerts/<date>.jsonl` with one line per changed parcel:
  `{parcel_id, change_type ENUM[new_code_case, case_closed, owner_changed, entity_status_changed], summary}`.
  Summaries are generated from fixed templates, facts only, e.g. "A new code enforcement
  case was opened at 1420 Ridgeway Ave on 8 July 2026." No adjectives.
- Every summary passes through the redaction guard before it is written.
- The Worker consumes `alerts/*.jsonl` (uploaded to R2 by the pipeline) and texts
  subscribers.

---

## Web (`/web`)

Blocked until D1 is answered.

- Astro static generation: one page per parcel at `/va/fairfax/<parcel-slug>/`; one per
  entity at `/company/<scc-slug>/` listing its parcels in the county **via confirmed
  `owner_entity` links only**; a search page at `/`; `/how-we-read-records/`,
  `/add-a-county/`, `/privacy/`, `/terms/`, `/report/`, `/owners/`.
- Search: client-side FlexSearch over a prebuilt compact index (`address_norm`, owner
  `name_norm`, entity `name_norm`, parcel IDs) served as static JSON, sharded per D1.
  Nothing typed leaves the browser except a `POST /count {event:"search"}` beacon with no
  payload.
- Page structure, in order: address header (with "Text me if a record changes"); Who owns
  it; The company; Code enforcement (table; hidden below threshold and for individual
  owners); Not shown here yet; aside: Questions to ask (each tied to a specific record on
  the page, generated from templates in `/web/src/content/questions/*.md`); aside:
  letter/deposit-clock link.
- "What this means" blocks come from `/web/src/content/means/<record_type>.md`.
- **Content lint.** Fails the build on: slumlord, negligent, ignores, refuses, bad,
  dangerous, unsafe, shady, avoid, warning, beware, likely, probably, pattern, history of.
  The list lives in `web/lint/forbidden_words.json`. Matching is word-boundary only — `bad`
  must not match `badge` — and the lint runs over rendered page content (content
  collections and templates), not the whole repo, so that docs and the "Not shown here yet"
  copy do not trip it. Both behaviors have tests. There is no inline escape hatch.
- **Color lint.** A CSS check permitting exactly two semantic status tokens (open, closed)
  beyond the accent and the neutral ramp. This is how rule 4 is mechanically enforced;
  "no color used as judgment" is otherwise untestable.
- Footer, every page, verbatim placeholders the owner will replace: `{{FCRA_PROHIBITION}}`,
  `{{NOT_LEGAL_ADVICE}}`, `{{CONCEPT_BANNER}}` (shown while `PUBLIC_LAUNCH!=true`). Links:
  Report an error on this page · I own this building · How records are read · Privacy ·
  Terms.
- `/report/` and `/owners/` are forms posting to the Worker, which forwards to an email
  inbox and writes a `disputes` row via a signed webhook to the pipeline API. Confirmation
  copy: "We flag the record as disputed while we check, and reply within five business
  days."
- Disputed records render with an inline "This record is disputed and under review" line,
  never silently hidden or silently kept.
- Design: match `/design/*.html`. Atkinson Hyperlegible via Google Fonts, 18px base, one
  accent red used only for the logo, large type, single column on mobile, 3:2 grid ≥720px.
  No all-caps labels, no gradients, no card grids, no icons except the search glyph. Visible
  focus rings; `prefers-reduced-motion` respected; aria-live on search results; Lighthouse
  accessibility ≥95; total JS ≤60 KB gzipped including the search index loader (the index
  itself loads lazily).
- English first; structure all strings in `/web/src/i18n/en.ts` so Spanish can be added
  without touching templates.

---

## Worker (`/worker`)

- `POST /watch {phone, parcel_id}`: validate E.164, store `{phone, parcel_id, created}` in
  KV with a TTL of `WATCH_RETENTION_DAYS`. Send Twilio confirmation: "Certified: we'll text
  you if a public record changes for [address]. Reply STOP to cancel." One number per
  parcel; no other data.
- Scheduled: read new `alerts/*.jsonl` from R2, text each subscriber one message per change,
  from templates only.
- Inbound webhook: STOP deletes the KV entry; HELP replies with a fixed message.
- `POST /count {event}`: increment KV counters for `search`, `page_view`, `watch_set`,
  `report_submitted`, `owner_reply`, `letter_click`. No IPs, no user agents, no timestamps
  finer than day.
- `POST /report`, `POST /owner-reply`: rate-limited, forwarded to email + signed webhook.
  Never log message bodies.
- Wrangler config with secrets: `TWILIO_SID`, `TWILIO_TOKEN`, `TWILIO_FROM`, `REPORT_EMAIL`,
  `PIPELINE_WEBHOOK_SECRET`.

---

## Docs to write

- `DISPLAY_RULES.md`: the eleven non-negotiables plus the threshold, the redaction guard,
  the forbidden-words lint, the color lint, and the disputed-record behaviour, each with the
  test that enforces it.
- `DECISIONS.md`: D1–D4 and anything later that changes a rule. Date, decision, rationale,
  who decided.
- `ADDING_A_COUNTY.md`: extractor interface, normalization fixtures, the 200-record sample,
  the ≥99% bar, SOURCES.md entry, legal review checkbox.
- `DATA_DICTIONARY.md`: every column, its source, and whether it is displayed.
- `INCIDENT_RESPONSE.md`: what to do on a report-an-error, an owner reply, or an attorney
  letter: flag as disputed within 1 business day, pull the raw copy, compare, correct or
  uphold within 5 business days, log in `disputes`, keep the correspondence. Include a
  placeholder for counsel's standard response letter.
- `SOURCES.md`: as described above.

---

## Working style

- Commit after each milestone below with a clear message. Run the full test suite before
  each commit.
- When a source's terms, robots.txt, or structure block automated access, stop and write the
  finding to SOURCES.md and ask the owner. Do not work around access controls.
- Never invent records for fixtures; use obviously synthetic data (`TEST PARCEL 0001`,
  `EXAMPLE HOLDINGS LLC`).
- Ask before adding any dependency that makes network calls at runtime in the web layer.

---

## Milestones

1. Repo scaffold, uv + Astro + wrangler configs, CI (GitHub Actions: pytest, mypy, vitest,
   astro build, lint), pre-commit. Nothing source-specific is baked in yet.
2. **SOURCES.md with verified endpoints and terms for all three sources. Stop and report.**
   This comes before the schema because the schema's shape, and the product's shape, depend
   on what is actually obtainable. Settles D4; the owner answers D1 and D2 here.
3. Schema + Alembic migrations + DATA_DICTIONARY.md, shaped by what milestone 2 found.
4. Land-records extractor + raw store + provenance + tests.
5. Address and name normalization with fixtures.
6. SCC extractor + exact-match matcher + `pipeline match review` CLI + matching log + tests.
7. Code Compliance extractor + OCR path + redaction guard + adversarial tests + measured
   false-positive rate. Settles D3.
8. Quality gate CLI + publish blocker.
9. Snapshot + diff + alert templates, guard on the alert path.
10. Web: page templates matching the mockup, content lint, color lint, i18n structure,
    search index build per D1.
11. Worker: watch, count, report, owner-reply, STOP/HELP.
12. Docs complete; DISPLAY_RULES.md cross-references every test.
13. End-to-end: run pipeline on a 500-parcel subset, hand-check sample, build site, deploy
    to Cloudflare Pages preview with Concept banner on.

---

## Kickoff prompt (paste into Claude Code)

```
Read CLAUDE.md fully before doing anything. Then:

1. Restate the eleven non-negotiables in your own words and list the test you will write
   for each. Restate D1-D4 and say what you need from me to close them. Stop and show me.
2. Start milestone 1. Commit.
3. Do milestone 2: fetch and verify the real endpoints and terms for the three
   Fairfax/Virginia sources. Write SOURCES.md. Stop and tell me before writing any
   extractor. If any source's terms prohibit automated access, or if Code Compliance case
   data is not publicly available, say so plainly and do not design around it.
4. Continue milestone by milestone. After each, run the full suite, commit, and give me a
   three-line summary: what shipped, what's blocked, what you need from me.

Do not skip the quality gate or weaken any display rule to make a page render. If a rule
makes a page empty, the page renders the "Not shown here yet" section and nothing else.
An empty code-enforcement section is a correct outcome.
```

---

## What changed from the first draft

- Added non-negotiable 3 (individual owners) — the original ten protected tenants' names but
  let a natural person's name sit next to adverse code history.
- Added non-negotiable 6 (no implied common ownership) — the company page was an unstated
  inference without it.
- Rule 9 (was 7) reworded: "nothing a visitor types is stored" was contradicted by the watch
  feature; added retention and a plain statement of what a watch record is.
- Rule 11 (was 10) now fails the build rather than warning.
- Source verification moved from milestone 3 to milestone 2, ahead of the schema.
- D1–D4 added as explicit blocking decisions.
- Schema: `code_cases.parcel_id` nullable + `address_raw` + unmatched report;
  `parcel_owner.is_current` + instrument number in the PK; `mailing_address_norm` on owners;
  `disputes.record_table` check constraint.
- Redaction guard runs on the alert/SMS path too, and reports a measured false-positive rate.
- `pipeline match review` promoted to a milestone 6 deliverable.
- Content lint scoped and word-boundary matched; color lint added to make rule 4 testable.
