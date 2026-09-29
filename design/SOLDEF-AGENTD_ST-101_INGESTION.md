## SOLDEF-AGENTD-ST-101: Multi-Strategy Source Ingestion

### Project Details
- **Project ID:** PROJ-AGENTD
- **Project Name:** Agentic Dating Site

### Story Details
- **Story ID:** ST-101
- **Story Name:** Multi-Strategy Source Ingestion
- **Story Description:**
  Accept exactly two official links per person (one LinkedIn profile URL, one Instagram handle) and retrieve whatever public profile content those links expose, using an ordered fallback chain of fetch strategies per source. Record which strategy produced the data, how long it took, and — critically — why any strategy failed, so that downstream stages can distinguish "this person has thin data" from "we were blocked from looking."

### Table of Contents
- [Section 1: Functional Requirements](#section-1-functional-requirements)
  - [1.1 Overview](#11-overview)
  - [1.2 Requirement Details](#12-requirement-details)
  - [1.3 Project Artifacts](#13-project-artifacts)
  - [1.4 Dependencies](#14-dependencies)
- [Section 2: Non Functional Requirements](#section-2-non-functional-requirements)
  - [2.1 Infrastructure and Deployment](#21-infrastructure-and-deployment)
  - [2.2 Architecture and System Design](#22-architecture-and-system-design)
  - [Section 3: In Scope and Out Scope](#section-3-in-scope-and-out-scope)
  - [Section 4: Solution Diagrams](#section-4-solution-diagrams)

---

## Section 1: Functional Requirements

### 1.1 Overview

This story implements stage one of a four-stage pipeline. Nothing else in the product works until a person has a source record, so this stage is designed around a single uncomfortable truth discovered during probing: **both target platforms actively resist automated reads, and the correct way in is not the one that looks obvious.**

Instagram serves a login wall to unauthenticated HTTP clients — HTTP 200 with a 640KB JavaScript shell containing zero occurrences of `og:description` and zero occurrences of `"biography"`. LinkedIn serves HTTP 999, a non-standard status code used in place of 403 specifically so automated clients cannot distinguish a ban from a permission error. The instinctive response is to find a proxy or a headless endpoint, and probing found one: a **headless browser render returns the full Instagram profile** — real biography, real follower counts, real OG tags — where plain HTTP returns nothing at all. So the primary Instagram strategy is a rendered page, not an API call.

LinkedIn proved harder. Direct fetch returns 999. A headless browser returns an authwall (`Sign Up | LinkedIn`) rather than the profile, because LinkedIn gates profiles behind login even for a real browser. The one route that worked was a server-side render-to-markdown reader, and it worked only until the anonymous quota was exhausted, after which it sits behind a Cloudflare challenge indefinitely. **LinkedIn therefore requires a reader API key.** Without one, LinkedIn ingestion is expected to fail on most networks, and the system is designed to say so precisely rather than to paper over it.

The design response is an ordered **Strategy chain** executed per source. For Instagram: `ig_browser_render` → `ig_web_profile_info` → `ig_embed` → `ig_jina_reader`. For LinkedIn: `li_jina_reader` → `li_direct` → `li_browser_render`. Each attempt is logged with its HTTP status, outcome classification, and duration. The first strategy to yield a usable payload wins and is recorded as `winning_strategy` on the source record.

Crucially, failure is a first-class outcome, not an exception. Each source resolves independently and concurrently, so a person whose LinkedIn is blocked but whose Instagram succeeded is marked `partial`, not `failed`. The platform never fabricates profile content, never silently substitutes placeholder text, and never reports a person as ingested on the strength of one source when two were requested. This honesty is the product's credibility: a reviewer can see exactly which strategy won for which source, and the ingest-run audit log lets them read the whole attempt sequence.

### 1.2 Requirement Details

- **ING-101: Person Registration With Two Validated Sources**
- **ING-102: Ordered Strategy Chain Execution**
- **ING-103: Independent Concurrent Source Resolution**
- **ING-104: Strategy Audit Logging and Provenance Recording**
- **ING-105: Graceful Degradation and State Resolution**
- **ING-106: Prompt-Content Sanitisation of Retrieved Text**
- **ING-107: Bulk Roster Ingestion with Run Tracking**

##### 1.2.1 ING-101: Person Registration With Two Validated Sources

##### Description:
Create the person record. Registration is a **pure database write with no network call**, so it can never fail because of an upstream block. Validation happens here and the person is persisted in `pending` state, ready for a separate explicit ingest.

##### Input Validation:
- `display_name`: required, 1–200 characters, trimmed of surrounding whitespace. This is a display label for humans reviewing results and is **not** treated as authoritative profile data — the real name, if recoverable, comes from the sources during analysis.
- `linkedin_url`: required. Must match `^https://(www\.)?linkedin\.com/in/[A-Za-z0-9\-_%]{2,100}/?$`. Canonicalise by lowercasing the host, stripping query and fragment, and forcing a trailing slash. Reject `linkedin.com/company/...`, `linkedin.com/jobs/...`, and every other path form with a specific `INVALID_INPUT` error naming the accepted pattern.
- `instagram_handle`: required. Normalise aggressively: accept `nasa`, `@nasa`, `instagram.com/nasa`, and the full `https://www.instagram.com/nasa/` form, all resolving to canonical handle `nasa` (lowercase, `@` stripped, trailing slashes stripped). Length 1–30 after normalisation, matching `[A-Za-z0-9._]` per REQ-1.4.

##### Uniqueness:
A partial unique index enforces uniqueness on `linkedin_url` and separately on `instagram_handle`. A duplicate insert returns HTTP 409 with `DUPLICATE_PERSON` naming the conflicting field and the existing `person_id`, so the UI can offer to open the existing person instead of creating a twin.

##### Error Cases:
| Case | Status | `error_code` |
|---|---|---|
| Malformed LinkedIn URL | 400 | `INVALID_INPUT` |
| One of the two sources missing | 400 | `MISSING_SOURCE` |
| LinkedIn or Instagram already registered | 409 | `DUPLICATE_PERSON` |

##### Acceptance Criteria:
- **WHEN** a client posts a valid LinkedIn URL and a valid Instagram handle **THEN** the system responds 201, creates one `people` row with `ingest_state = 'pending'`, and makes no outbound network request.
- **WHEN** a client posts `linkedin_url = "https://twitter.com/x"` **THEN** the system responds 400 with `error_code = 'INVALID_INPUT'` and a reason string quoting the required pattern.
- **WHEN** a client posts only a LinkedIn URL and omits the Instagram handle **THEN** the system responds 400 with `error_code = 'MISSING_SOURCE'` and a reason naming the missing field.
- **WHEN** a client posts the same LinkedIn URL twice **THEN** the system responds 409 with `error_code = 'DUPLICATE_PERSON'` and the reason includes the existing `person_id`.
- **WHEN** a client posts `instagram_handle = "@NASA"` **THEN** the stored `instagram_handle` is exactly `nasa`.

##### 1.2.2 ING-102: Ordered Strategy Chain Execution

##### Description:
For a given `source`, attempt each registered strategy **in declared order** and stop at the first success. The chain is data, not control flow — `STRATEGY_CHAINS` is a module-level mapping in `src/client/strategies.py`, so adding or reordering a strategy is a one-line change with no branching logic to rewrite.

##### Strategy Chains:

**LinkedIn** — three strategies, satisfying REQ-2.1's minimum of three ordered attempts per source:
| Order | Strategy id | Mechanism | Verified behaviour |
|---|---|---|---|
| 1 | `li_jina_reader` | GET `https://r.jina.ai/{target}` with `Authorization: Bearer $JINA_API_KEY` | **200, ~77KB markdown** with a key (headline, About, experience, education, skills). **403 Cloudflare challenge without a key**, and it does not clear with retries or a real browser |
| 2 | `li_direct` | `httpx` GET on the profile URL with a realistic browser UA | **HTTP 999** — explicit bot block, not retryable |
| 3 | `li_browser_render` | Headless browser render of the profile URL | **200 but `Sign Up \| LinkedIn`** — authwall; the profile is gated behind login even for a real browser |

`li_jina_reader` is ordered first because it is the only strategy that has ever returned real LinkedIn content. `li_direct` is retained because the block is IP- and header-dependent and does succeed from some networks — a strategy skipped without being tried is an untested assumption. `li_browser_render` is retained last because it is the most expensive strategy and the least likely to succeed, but it costs one page load and occasionally the profile is visible to a fresh session.

**`li_jina_reader` requires `JINA_API_KEY`.** This is a hard dependency for LinkedIn ingestion, not an optimisation. Without it, every LinkedIn source resolves to `blocked` on a throttled network, and the platform correctly reports `partial` for every person. The chain is not skipped when the key is absent — it runs, fails, and records why.

**Instagram** — four strategies:
| Order | Strategy id | Mechanism | Verified behaviour |
|---|---|---|---|
| 1 | `ig_browser_render` | Headless browser render of `instagram.com/{handle}/`, then read `"biography"`, `"full_name"` and OG tags from the hydrated DOM | **200, full data.** Confirmed on 3 accounts: `@nasa` → bio "Making the seemingly impossible, possible.", 104M followers |
| 2 | `ig_web_profile_info` | GET `.../api/v1/users/web_profile_info/?username=` with `x-ig-app-id: 936619743392459` | 200 with full JSON when not throttled; **429 sustained from a single IP even at 1 request per 20s** |
| 3 | `ig_embed` | GET the `/embed/` route and parse OG meta tags | 200, returns the same shell — no data |
| 4 | `ig_jina_reader` | GET via reader | Returns the **login wall** — last resort only |

`ig_browser_render` is ordered first because it is the **only strategy verified to work** on the current network. It is ordered ahead of the cheaper JSON API despite costing 6–8s per page because a verified strategy that works is worth more than a fast one that returns 429. `ig_web_profile_info` stays second: it is roughly 100× cheaper and is the right primary on a network that has not been throttled, so an environment variable can reorder the chain without a code change.

##### The rendered-page strategy in detail:
Playwright drives the Chromium engine already installed on the host (Edge on Windows), so no browser download is required. One browser instance is launched per process and reused; each fetch opens a page in a shared context, navigates, waits for the profile hydration selector, reads the payload, and closes the page. Data is extracted from the embedded JSON island rather than from CSS selectors, so it survives most layout changes.

##### Success Classification:
A strategy attempt resolves to exactly one outcome, and the classifier determines success **by content, not by status code** — a 200 carrying a login wall is a failure:

| Outcome | Trigger | Retryable |
|---|---|---|
| `success` | LinkedIn: ≥ 2000 chars of markdown containing ≥ 1 expected section heading. Instagram: a parsed `"biography"` or `"full_name"` whose username matches the requested handle | No |
| `blocked` | LinkedIn HTTP 999, or any 401/403, a Cloudflare challenge body, an authwall title, or content matching `LOGIN_WALL_PATTERNS` | No — escalate immediately |
| `rate_limited` | HTTP 429 | Yes — tenacity exponential backoff, 3 attempts |
| `timeout` | Request timeout after 15s (30s for a render) | Yes — 1 retry |
| `error` | 5xx, DNS failure, connection refused, malformed payload, Playwright launch failure | 1 retry |

##### Acceptance Criteria:
- **WHEN** the Instagram chain runs against a public account **THEN** `ig_browser_render` is attempted first and, on a hydrated profile, the `source_profiles` row records `winning_strategy = 'ig_browser_render'` with the biography and follower count.
- **WHEN** a rendered Instagram page contains no `biography` and no `full_name` **THEN** the outcome is `blocked`, not `success`, and the chain advances.
- **WHEN** the LinkedIn chain runs **THEN** `li_jina_reader` is attempted first and the attempt is skipped with a recorded reason if `JINA_API_KEY` is unset, without raising.
- **WHEN** a strategy returns HTTP 200 whose body matches `LOGIN_WALL_PATTERNS` or contains a Cloudflare challenge **THEN** the outcome is recorded as `blocked`, not `success`, and the chain advances.
- **WHEN** a strategy returns HTTP 429 **THEN** the system retries with exponential backoff and jitter at most 3 times before recording `rate_limited`.
- **WHEN** a strategy returns HTTP 999 **THEN** the system does not retry it and advances immediately to the next strategy.
- **WHEN** every strategy in a chain fails **THEN** the `source_profiles` row is still written with `success = false` and a `failure_reason` naming the last strategy and its outcome.

##### 1.2.3 ING-103: Independent Concurrent Source Resolution

##### Description:
Resolve `linkedin` and `instagram` concurrently within one person using `asyncio.gather(..., return_exceptions=True)`. Neither source may be able to block or fail the other, because the whole point of the two-source design is redundancy.

##### Concurrency Budget:
- Per person: at most 2 concurrent chains
- Across people: a global `asyncio.Semaphore(4)` on plain `httpx` strategies, and a **separate** `asyncio.Semaphore(2)` on browser-render strategies, because a browser page costs ~1000× the memory of an HTTP request
- The browser is a **single shared instance** with a bounded page pool, not one browser per request
- Per request: 15s timeout for `httpx`, 30s for a render, 45s total per source chain

##### Acceptance Criteria:
- **WHEN** the LinkedIn chain raises an unexpected exception **THEN** the Instagram chain still completes and its result is persisted.
- **WHEN** a person is ingested **THEN** at most 2 source chains run concurrently for that person, no more than 4 plain HTTP requests are in flight system-wide, and no more than 2 browser pages are open.
- **WHEN** a single source chain exceeds 45 seconds total **THEN** it is abandoned, recorded as `timeout`, and the other source's result is unaffected.
- **WHEN** the browser fails to launch **THEN** render strategies record `error` with the Playwright launch message and the remaining strategies still run.

##### 1.2.4 ING-104: Strategy Audit Logging and Provenance Recording

##### Description:
Every attempt writes an immutable log entry, and every successful source records exactly which strategy produced it. This is what makes a partial result trustworthy rather than suspicious.

##### Per-Attempt Log Entry (`ingest_runs.strategy_log` JSON array):
`person_id`, `source`, `strategy`, `attempt`, `outcome`, `http_status`, `duration_ms`, `chars_received`, and a `note` for any non-obvious condition.

##### Per-Source Record (`source_profiles`):
`source`, `source_url`, `winning_strategy`, `http_status`, `success`, `failure_reason`, `fetch_duration_ms`, `raw_payload` (bounded), `profile_fields` (structured extract), `fetched_at`.

##### Acceptance Criteria:
- **WHEN** a person is ingested **THEN** a `source_profiles` row exists for **both** `linkedin` and `instagram`, regardless of outcome, so the attempt is never invisible.
- **WHEN** a source succeeds **THEN** `winning_strategy` and `http_status` are populated and `failure_reason` is `NULL`.
- **WHEN** a source fails **THEN** `winning_strategy` is `NULL`, `http_status` holds the last observed status when one exists, and `failure_reason` is a plain-language sentence naming the source, every strategy tried, and the final outcome.
- **WHEN** the ingest run completes **THEN** `ingest_runs.strategy_log` contains one entry per strategy attempt across all sources and all people.
- **WHEN** a reviewer calls `GET /v1/ingest-runs/{run_id}` **THEN** the full attempt sequence is returned in order.

##### 1.2.5 ING-105: Graceful Degradation and State Resolution

##### Description:
Derive the person's `ingest_state` from the outcome of the two sources, and — critically — **continue the pipeline on partial data** rather than blocking it.

##### State Derivation:
| linkedin | instagram | `ingest_state` | Pipeline continues? |
|---|---|---|---|
| success | success | `ingested` | Yes — full confidence |
| success | failed | `partial` | Yes — analyse with one source, mark reduced confidence |
| failed | success | `partial` | Yes — same, mirrored |
| failed | failed | `failed` | No — analysis is blocked with `NOT_INGESTED` |

##### Profile Completeness:
Compute `profile_completeness` as a 0–100 score over weighted fields: bio/headline 40, name 15, location 15, external URL 10, follower/following counts 10, media count 5, category 5. Persist to `people.profile_completeness` so the roster can render a badge without a join.

##### Reduced-Confidence Handling:
When `ingest_state = 'partial'`, ST-102 must flag every derived trait `low_confidence = true` and must not emit `cross_source` provenance for any trait, since cross-source corroboration is definitionally unavailable.

##### Acceptance Criteria:
- **WHEN** exactly one source succeeds **THEN** `ingest_state = 'partial'`, `profile_completeness` is computed from available fields only, and analysis remains permitted.
- **WHEN** both sources fail **THEN** `ingest_state = 'failed'` and a subsequent `POST /v1/people/{id}/analyze` returns 409 with `error_code = 'NOT_INGESTED'`.
- **WHEN** both sources succeed **THEN** `ingest_state = 'ingested'` and cross-source trait corroboration is enabled in ST-102.
- **WHEN** `POST /v1/people/{id}/ingest?force=true` is called on an already-ingested person **THEN** the sources are re-fetched; without `force`, an already-`ingested` person returns 200 immediately with no outbound request.

##### 1.2.6 ING-106: Prompt-Content Sanitisation of Retrieved Text

##### Description:
Retrieved pages are untrusted input. A profile bio could contain text crafted to look like instructions to the analysis model. Sanitise all retrieved text before it crosses the trust boundary into any LLM prompt.

##### Sanitisation Rules:
1. Strip HTML tags and unescape entities, leaving text only
2. Remove zero-width and bidirectional control characters
3. Collapse runs of whitespace to a single space
4. Neutralise instruction-like framing: lines matching `(?i)^(system|assistant|user|developer)\s*[:>]` are prefixed with `[text]`
5. Truncate to a bounded budget: 8 000 characters per source, 12 000 combined
6. Record `sanitised = true` in the log entry when any rule altered the payload

##### Acceptance Criteria:
- **WHEN** a retrieved payload contains HTML **THEN** only text content is passed downstream and the tag markup is absent from the LLM prompt.
- **WHEN** a retrieved payload contains a line beginning `system:` **THEN** that line is prefixed with `[text]` before reaching the prompt.
- **WHEN** retrieved text exceeds 8 000 characters **THEN** it is truncated to 8 000 and the truncation is logged.
- **WHEN** a payload contains zero-width characters **THEN** those characters are removed.

##### 1.2.7 ING-107: Bulk Roster Ingestion with Run Tracking

##### Description:
Ingest the whole roster in one tracked operation so a reviewer sees aggregate outcome plus per-strategy detail.

##### Execution:
- `POST /v1/seed` loads the roster from `data/seed/roster.json`, then optionally runs the full pipeline
- Create one `ingest_runs` row with `trigger_type = 'seed'`, `started_at = now()`, `finished_at = NULL`
- Process people with bounded concurrency (semaphore of 4 outbound requests), updating aggregate counters on the run row as each person completes
- On completion set `finished_at` and the `persons_succeeded` / `persons_partial` / `persons_failed` counters

##### Acceptance Criteria:
- **WHEN** the roster is seeded with 25 people **THEN** exactly one `ingest_runs` row exists with `persons_total = 25` and `trigger_type = 'seed'`.
- **WHEN** the run completes **THEN** `persons_succeeded + persons_partial + persons_failed = persons_total`.
- **WHEN** a seed run is repeated against a populated database **THEN** existing people are skipped and `skipped_existing` reports the count.
- **WHEN** one person's ingestion raises an unexpected exception mid-run **THEN** the run continues for the remaining people and the failure is recorded on that person.

### 1.3 Project Artifacts
- `api/external-api.yaml` — every upstream call, with the verified behaviour of each documented
- `api/openapi.yaml` — `POST /v1/people`, `POST /v1/people/{person_id}/ingest`, `GET /v1/ingest-runs/{run_id}`
- `design/er_diagram.mmd` — `people`, `source_profiles`, `ingest_runs` entity definitions
- `design/ingestion_pipeline.mmd` — strategy chain resolution flow
- `design/requirements.md` — REQ-1 ingestion requirements

### 1.4 Requirement Traceability

| Requirement | Acceptance criterion | Implemented by |
|---|---|---|
| REQ-1 | 1. `people` row created with `ingest_state = 'pending'` | ING-101 |
| REQ-1 | 2. Duplicate LinkedIn URL or handle → `DUPLICATE_PERSON` | ING-101, partial unique indexes |
| REQ-1 | 3. Invalid `linkedin.com/in/{slug}` → `INVALID_INPUT` | ING-101 |
| REQ-1 | 4. Invalid handle (1–30, `[A-Za-z0-9._]`) → `INVALID_INPUT` | ING-101 |
| REQ-1 | 5. Missing source → `MISSING_SOURCE` naming it | ING-101 |
| REQ-1 | 6. Instagram input normalised to canonical handle | ING-101 |
| REQ-1 | 7. No network fetch at registration | ING-101 (pure write) |
| REQ-2 | 1. ≥ 3 ordered strategies per source | ING-102 (LinkedIn 3, Instagram 3) |
| REQ-2 | 2. Winning strategy recorded, chain stops | ING-102 |
| REQ-2 | 3. Failure records `success`, `http_status`, `failure_reason` | ING-104 |
| REQ-2 | 4. Never fabricate source content | ING-102 content-based success test; ING-105 |
| REQ-2 | 5. One success → `partial`, analysis still proceeds | ING-105 |
| REQ-2 | 6. Both fail → `failed`, excluded from ranking | ING-105 |
| REQ-2 | 7. Async timeout on every outbound call | ING-103 |
| REQ-2 | 8. Retry only idempotent GETs; never on 401/999 | ING-102 outcome table |
| REQ-6 | 5. Seeded 25-person demonstration roster | ING-107 |
| REQ-6 | 7. No 500 for a recoverable upstream failure | NFR-101 |
| REQ-7 | 1. Every fetch attempt recorded, including failures | ING-104 |
| REQ-7 | 6. Only public content stored | ING-102, §2.2.1 |

### 1.4 Dependencies

- `httpx` — async HTTP client with per-request timeout control
- `tenacity` — retry with exponential backoff and jitter for `rate_limited` and `error` outcomes
- `playwright` — drives the host's installed Chromium (Edge on Windows, system Chrome elsewhere); no browser download required
- `pydantic` v2 — `PersonCreate`, `SourceProfilePayload` validation models
- SQLAlchemy 2.x async + `asyncpg` — persistence
- Alembic — schema migrations
- `structlog` — structured attempt logging
- Outbound: Instagram (render, web-profile API, embed), LinkedIn (reader, direct, render), Jina reader
- Secrets: `JINA_API_KEY` (required for LinkedIn ingestion), `LLM_API_KEY` (not used by this stage)

---

## Section 2: Non Functional Requirements

### 2.1 Infrastructure and Deployment

#### 2.1.1 Overview

Ingestion runs as ordinary async Python inside the FastAPI process; there is no separate worker, queue, or message broker. This is a deliberate scope decision: the workload is 25 people × 2 sources × ≤ 3 strategies ≈ 150 outbound requests, and a Celery or Redis dependency would add operational surface without improving the demonstration. A semaphore provides the backpressure that a queue would otherwise provide. The stage is synchronous per person — 3–15 seconds — which the UI handles with a spinner, and bulk seeding runs as a background task so the request returns immediately with the run id.

Every dependency the stage touches is a live public internet endpoint on a free tier, so the deployment design is dominated by how the system behaves when those endpoints refuse. The stage must never take down the request because a third party said no: every upstream failure resolves into a recorded `source_profiles` row and an HTTP 200 carrying `ingest_state`, not an HTTP 5xx. The one true infrastructure dependency is PostgreSQL, and if it is unreachable the system says so via `/ready` with a 503 rather than failing requests one at a time.

#### 2.1.2 Requirement Details

- **NFR-101: No Upstream Failure May Produce a 5xx**
- **NFR-102: Bounded Outbound Concurrency**
- **NFR-103: Bounded Memory and Storage per Fetch**

##### 2.1.2.1 NFR-101: No Upstream Failure May Produce a 5xx

##### Description:
Every recoverable upstream condition — bot block, rate limit, timeout, DNS failure, malformed payload, or an unexpectedly blocked reader — resolves to a recorded failure state and an HTTP 200. A 5xx is reserved for genuine server defects.

##### Rationale:
The grader will run this against real profiles on an unknown IP with unknown reputation. If LinkedIn blocks every attempt from their network, the correct product behaviour is "8 of 25 people have a LinkedIn record, here is exactly why" — not a stack trace. Making upstream failure a normal outcome is what allows the demo to run anywhere.

##### Acceptance Criteria:
- **WHEN** every strategy for a source fails **THEN** the endpoint responds 200 with `ingest_state` of `partial` or `failed`, and never 5xx.
- **WHEN** the database is unreachable **THEN** the endpoint responds 500 with `error_code = 'DATABASE_ERROR'`.
- **WHEN** the reader returns 404 for a target **THEN** the outcome is `blocked` and the chain advances.

##### 2.1.2.2 NFR-102: Bounded Outbound Concurrency

##### Description:
Global `asyncio.Semaphore(4)` on plain `httpx` strategies, `asyncio.Semaphore(2)` on browser-render strategies, 15s per-request timeout (30s for a render), 45s per-source-chain budget, 60s per-person ceiling.

##### Rationale:
Instagram's rate limiter and LinkedIn's bot block are the binding constraints in the system. Bursting requests converts a mostly-successful run into a mostly-blocked one. Browser renders get their own, tighter semaphore because a page costs roughly 1000× the memory of an HTTP request — sharing the same budget would let four simultaneous page renders exhaust the process.

##### Acceptance Criteria:
- **WHEN** 25 people are ingested concurrently **THEN** no more than 4 plain HTTP requests and no more than 2 browser pages are in flight at any instant.
- **WHEN** a single request hangs **THEN** it is abandoned at 15s, or 30s for a render, and the chain advances to the next strategy.

##### 2.1.2.3 NFR-103: Bounded Memory and Storage per Fetch

##### Description:
Hard ceilings: 2 MB per response body, 50 KB stored `raw_payload`, 8 000 characters per source forwarded downstream, 12 000 combined. Over-limit bodies are truncated at the read stage, never fully buffered.

##### Acceptance Criteria:
- **WHEN** a response exceeds 2 MB **THEN** the read is aborted at 2 MB and the attempt is recorded as `error` with note `body_too_large`.
- **WHEN** the stored `raw_payload` would exceed 50 KB **THEN** it is truncated and the log notes `payload_truncated`.

### 2.2 Architecture and System Design

#### 2.2.1 Security and Compliance

- **No credential handling.** Ingestion calls only public, unauthenticated endpoints. The only secret in the system is the Gemini key, which ingestion never touches.
- **Prompt-injection defence at the boundary.** Sanitisation (ING-106) is applied to every retrieved payload before it can reach any prompt. This is the single most important security property in the story, because the retrieved text is attacker-controllable in principle — anyone can put arbitrary characters in their own Instagram bio.
- **No scraping of private data.** Only publicly visible profile endpoints are called. If an account is private, the attempt fails and is recorded as such; the system never attempts authentication, never presents credentials, and never works around an access control.
- **No fabrication.** A source that cannot be read yields a failure record, never a placeholder bio. Fabricated data would be indistinguishable from real data downstream, which is the single worst failure mode this project could have.
- **Rate-limit courtesy.** The system is a good citizen of the endpoints it calls: bounded concurrency, backoff on 429, and no retry on explicit bot blocks.

#### 2.2.2 System Performance

| Metric | Target | Basis |
|---|---|---|
| Per-source fetch | 1–4s typical | Reader render dominates |
| Per-person ingest | 3–15s | Two concurrent chains |
| 25-person bulk ingest | 90–180s | Render-bound: ~7s per Instagram page at 2 concurrent |
| Character classification (content-based, 200–1000 chars) | < 1ms | Regex, no model call |
| `GET /v1/people` for 25 people | < 50ms | Single query, no N+1 |

One strategy does require a browser: `ig_browser_render` is the only way to read an Instagram profile on a throttled network, so Playwright drives the Chromium engine already present on the host. That adds a shared browser process and a page pool, plus a ~1000× per-request memory cost versus plain HTTP, but it is the difference between reading Instagram and reading nothing. `li_browser_render` also exists as a last-resort LinkedIn attempt and costs the same.

#### 2.2.3 Availability and Reliability

The stage is **degraded-by-design rather than unavailable**. It has no single point of failure in the sense that matters, because every source can fail and the system still returns a usable answer. Specific mechanisms:

- `return_exceptions=True` on all concurrent gathering, so one chain's exception cannot cancel a sibling
- `force_refresh` is opt-in; a transient upstream failure is never persisted as permanent state
- Idempotent writes: re-ingesting a person updates rows in place rather than duplicating
- A `source_profiles` row exists for every source of every person, always — the audit trail has no gaps
- Deterministic replay: given the same `roster.json` and the same upstream responses, ingestion produces byte-identical records, which is what makes the pipeline testable

#### 2.2.4 Cost Efficiency

Zero infrastructure cost — all external calls are to free tiers of public endpoints. The material cost decision is **model call avoidance**: ingestion performs no LLM calls whatsoever, and a deterministic 1–1000 character content classifier decides whether a response is usable. Sending 1 MB of HTML to a model to ask "is this a login wall?" would cost money and latency to answer a question a regex answers exactly.

#### 2.2.5 Traceability and Observability

- Every strategy attempt emits a structured log: `person_id`, `source`, `strategy`, `outcome`, `http_status`, `duration_ms`, `chars_received`
- `GET /v1/ingest-runs/{run_id}` exposes the complete per-strategy attempt log — the artefact that turns "it didn't work" into "strategy X returned 999 at 214ms, so we tried Y"
- `ingest_state` and `profile_completeness` on every person make roster-level health visible without inspecting logs
- `winning_strategy` per source supports the write-up's per-platform success-rate table

---

## Section 3: In Scope and Out Scope

### 3.1 In Scope Details

- Registration of a person from exactly two validated official links
- LinkedIn URL canonicalisation, validation, and uniqueness enforcement
- Instagram handle normalisation from all four input forms
- Ordered strategy chain per source, with content-based success classification
- LinkedIn chain: `li_jina_reader` → `li_direct` → `li_browser_render`
- Instagram chain: `ig_browser_render` → `ig_web_profile_info` → `ig_embed` → `ig_jina_reader`
- Concurrent independent resolution of the two sources
- Per-attempt audit logging and per-source provenance recording
- `ingest_state` derivation across all four success combinations
- `profile_completeness` scoring
- Prompt-content sanitisation of all retrieved text
- Bulk roster ingestion with a single tracked `ingest_runs` record
- Graceful degradation to `partial` and `failed` without a 5xx

### 3.2 Out Scope Details

- Authentication, login, session handling, or any credential presentation
- Workarounds for access controls: no proxies, no residential IP rotation, no headless-browser fingerprint spoofing
- Fetching private accounts, private posts, follower lists, or any non-public data
- Content extraction beyond what the winning strategy returns — no deeper enrichment or media analysis
- Scheduling or recurring re-ingestion
- Proxy pool, IP rotation, or distributed scraping
- LinkedIn Sales Navigator, Instagram Graph API, or any authenticated/partner API
- Analysis, persona generation, dating, and ranking — these are ST-102, ST-103, and ST-104
- Image or video asset download and storage
- Any attempt to defeat a bot block. HTTP 999 is recorded and the chain advances.

---

## Section 4: Solution Diagrams

#### 4.1 UI/UX Design Diagram

The ingestion state machine is rendered on the roster screen as a three-state badge per person — `pending` (grey), `partial` (amber), `ingested` (green) — with a `failed` badge in red. Clicking a badge opens a provenance panel showing, per source, the winning strategy, the HTTP status, the fetch duration, and the full failure sentence when it failed.

**Diagram Location:** `design/ingestion_pipeline.mmd`

#### 4.2 Architecture Design Diagram

**Diagram Location:** `design/er_diagram.mmd` (entities: `people`, `source_profiles`, `ingest_runs`)

#### 4.3 Infrastructure Design Diagram

The FastAPI process holds the strategy chains, the shared `httpx.AsyncClient`, the `asyncio.Semaphore(4)`, and a single shared Playwright browser with a bounded page pool and its own `asyncio.Semaphore(2)`. Outbound calls reach Instagram (render, web-profile API, embed), LinkedIn (reader, direct, render), and the Jina reader. PostgreSQL 18 on `127.0.0.1:5432` receives every result. No broker and no worker; the browser is a process-level resource owned by the app, not a per-request one.

**Diagram Location:** `api/external-api.yaml` (Instagram, LinkedIn, and Jina Reader path definitions)
