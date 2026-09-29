## SOLDEF-AGENTD-ST-102: Profile Analysis and Agent Persona Generation

### Project Details
- **Project ID:** PROJ-AGENTD
- **Project Name:** Agentic Dating Site

### Story Details
- **Story ID:** ST-102
- **Story Name:** Profile Analysis and Agent Persona Generation
- **Story Description:**
  Turn the two raw source payloads into a structured, evidence-backed trait set and a distinct speaking agent for each person. Every trait must carry the quote that supports it and the source it came from, and every agent must be prevented from claiming anything the underlying sources do not support.

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

Ingestion produces prose. This stage turns that prose into something a matching algorithm can compute over, and gives each person a voice.

Two decisions define the stage. First, **traits are typed, not free-text.** Rather than storing a paragraph summary and hoping a ranking function can reason about it, the stage decomposes a person into eight typed traits. Five are **extracted** by the LLM from the source text; three are **derived** in code from those five, because each maps directly onto something a later stage needs and recomputing it there would be wasteful:

| Type | Origin | Definition | Consumed by |
|---|---|---|---|
| `hobby` | LLM | An activity the person actually does | ST-104 dimension 2 |
| `interest` | LLM | A subject of sustained interest | ST-104 dimension 1 |
| `need` | LLM | Something wanted from a partner | ST-104 dimension 4 |
| `value` | LLM | A principle the person holds | ST-104 dimension 3 |
| `dealbreaker` | LLM | A hard constraint on a partner | ST-104 dimension 8, ST-103 steering |
| `conversation_opener` | derived | A trait that reliably opens a line of talk | ST-103 opening messages |
| `date_idea` | derived | An activity concrete enough to propose | ST-103 `proposed_date` |
| `dealmaker` | derived | A trait stated positively, as its `dealbreaker` converse | ST-104 dimension 3 |

Deriving `date_idea` and `conversation_opener` here rather than in ST-103 is the reason ST-103 can build a real proposal plan: the concrete, proposable activities already exist as first-class rows with provenance, so the dating prompt has something grounded to work with instead of having to invent a plan from raw hobbies. This is what makes typed traits more than a matching convenience — they are the shared vocabulary the whole pipeline speaks.

Each item carries a label, a confidence, a provenance source, and an evidence quote. This is what makes ST-104 a matter of arithmetic over overlapping typed sets rather than semantic similarity, which is the difference between a ranking that is explainable and one that is a black box.

Second, **every claim must be traceable to a quote.** The system prompt demands an `evidence_quote` per trait, and a post-processing validator enforces that the quote is a genuine substring of the sanitised source text. A trait whose quote cannot be found is demoted to `inferred` with reduced confidence, or dropped. This is the mechanism that stops the model from inventing interests — a model asked about a sparse profile will otherwise happily produce plausible-sounding hobbies, and those fabrications would propagate into rankings and into agents' mouths where they would be indistinguishable from real facts.

The stage is also where **the free-tier LLM constraint is absorbed**. `gemini-3.1-flash-lite` is the only model verified working on the assessment key; `gemini-2.5-*` is retired, `gemini-3.8-flash` returns 503 under demand, and Pro models return 429. So the LLM Engine holds an ordered fallback chain, uses Gemini's `responseSchema` to constrain output to the trait schema at the provider level, and — critically — falls back to a **deterministic Taxonomy Engine** that performs keyword and pattern extraction when the LLM is unavailable. The pipeline therefore always completes, and every record states whether it was `generated_by: llm` or `generated_by: taxonomy`.

### 1.2 Requirement Details

- **ANA-201: Typed Trait Extraction with Evidence Attribution**
- **ANA-202: Provenance and Confidence Assignment**
- **ANA-203: LLM Engine with Ordered Model Fallback Chain**
- **ANA-204: Schema-Constrained Structured Output**
- **ANA-205: Deterministic Taxonomy Fallback Engine**
- **ANA-206: Agent Persona Generation with Guardrails**
- **ANA-207: Analysis Idempotency and Re-analysis**

##### 1.2.1 ANA-201: Typed Trait Extraction with Evidence Attribution

##### Description:
Produce a typed, de-duplicated trait set from the sanitised source text. The five LLM-extracted types plus the three code-derived types.

##### Extracted Types (LLM):
| Type | Definition | Example | Used by |
|---|---|---|---|
| `hobby` | An activity the person actually does | "trail running" | ST-104 hobby complement |
| `interest` | A subject of sustained interest | "climate tech" | ST-104 interest overlap |
| `need` | Something the person wants from a partner | "an ambitious partner" | ST-104 need satisfaction |
| `value` | A principle the person holds | "sustainability" | ST-104 value alignment |
| `dealbreaker` | A hard constraint on a partner | "long-distance only" | ST-104 penalty, ST-103 steering |

##### Derived Types (code, no model call):
| Type | Derivation rule | Example |
|---|---|---|
| `conversation_opener` | Highest-confidence `interest` or `hobby`, capped at 2 | "trail running" |
| `date_idea` | A `hobby` concrete enough to schedule (physical activity, venue, class) | "trail running" → "ridge trail walk" |
| `dealmaker` | A `value` whose canonical label is the positive converse of a common `dealbreaker` | `sustainability` is the dealmaker for `anti-environmental` |

Derived traits inherit the confidence and provenance of the trait they derive from, and record `provenance_strategy = 'derived'` so it is always clear they were computed rather than extracted. A derived trait is never invented — `date_idea` is only produced when a qualifying `hobby` already exists, so a sparse profile yields fewer date ideas rather than fabricated ones.

##### Extraction Rules:
- Emit **5–15 extracted traits** per person. This is the expected range, not a validity gate
- **Low-confidence threshold is 3 total traits** across both sources, per REQ-3.7. Below that the profile is marked low-confidence rather than padded
- De-duplicate case-insensitively on `label`, keeping the highest-confidence instance
- **Never infer a trait that contradicts the source.** If a profile says "vegan" and the model outputs "loves steak", the quote validation in ANA-202 rejects the trait
- A person with `ingest_state = 'partial'` may still be analysed; all traits are marked `low_confidence = true` and no trait may carry `cross_source` provenance

##### Acceptance Criteria:
- **WHEN** a profile contains both "trail running" and "marathon training" **THEN** both may appear as separate hobbies, since they are distinct activities, but two spellings of the same label collapse to one row.
- **WHEN** a profile yields fewer than 3 total traits **THEN** analysis still succeeds, the person is marked `low_confidence = true`, and no traits are invented to reach a target count.
- **WHEN** a person has at least one concrete `hobby` **THEN** a `date_idea` trait is derived from it with `provenance_strategy = 'derived'`.
- **WHEN** a person has no qualifying `hobby` **THEN** no `date_idea` is derived, and the dating engine proposes no plan.
- **WHEN** a person has `ingest_state = 'partial'` **THEN** every trait has `low_confidence = true` and no trait has `provenance_source = 'cross_source'`.
- **WHEN** the LLM returns a trait type outside the five extracted types **THEN** it is discarded, since the three derived types are computed in code and must not be model-authored.

##### 1.2.2 ANA-202: Provenance and Confidence Assignment

##### Description:
Every trait must be attributable. This is the requirement that separates this system from a plausible-sounding demo.

##### Provenance Values:
| Value | Meaning |
|---|---|
| `linkedin` | The label is present in the LinkedIn payload |
| `instagram` | The label is present in the Instagram payload |
| `cross_source` | The label appears in **both** payloads — the strongest signal, and the basis for genuine corroboration |
| `inferred` | No supporting quote was found; the trait is a reasoned inference and must be visibly weaker |

##### Confidence Rules:
- Base confidence per type: `dealbreaker` 0.9, `need` 0.8, `value` 0.75, `hobby` 0.7, `interest` 0.65 — dealbreakers and needs are the most explicitly stated
- `cross_source` adds +0.15; `inferred` multiplies by 0.5
- Capped to [0.0, 1.0]
- Traits below 0.4 confidence are not persisted

##### Evidence Validation (the fabrication guard):
Post-process every returned trait:
1. Normalise the `evidence_quote` (lowercase, collapse whitespace)
2. Search the sanitised source text for it
3. **Found** → keep confidence, set provenance to the source it was found in
4. **Not found** → set `provenance_source = 'inferred'`, `confidence *= 0.5`, and set `evidence_quote = ''`
5. **After demotion, confidence < 0.4** → drop the trait entirely

This is deliberately unforgiving. A fabricated quote must be able to fail.

##### Acceptance Criteria:
- **WHEN** a trait's `evidence_quote` is found verbatim in the Instagram payload **THEN** `provenance_source = 'instagram'` and `provenance_strategy` is recorded.
- **WHEN** a label appears in both payloads **THEN** `provenance_source = 'cross_source'` and confidence increases by 0.15.
- **WHEN** a trait's `evidence_quote` cannot be found in any payload **THEN** the trait is marked `inferred` with `evidence_quote = ''` and halved confidence.
- **WHEN** a demoted trait's confidence falls below 0.4 **THEN** the trait is not persisted.
- **WHEN** analysis completes **THEN** `100 * (traits with a non-empty evidence_quote) / (total traits)` is retrievable as the evidence coverage rate.

##### 1.2.3 ANA-203: LLM Engine with Ordered Model Fallback Chain

##### Description:
Wrap Gemini behind a client that resolves a working model once and reuses it, so the rest of the codebase never hardcodes a model name.

##### Verified Model Availability (probed on the assessment key):
| Model | Status | Action |
|---|---|---|
| `gemini-3.8-flash` | 503, high demand | Retry with backoff, then fall through |
| `gemini-3.1-flash-lite` | **200 — works** | Pin as the workhorse |
| `gemini-flash-latest` | 503, high demand | Fall through |
| `gemini-3.1-pro-preview` | 429, free-tier quota | Fall through |
| `gemini-2.5-flash` | 404, retired for new keys | Never attempted |

##### Fallback Chain:
`gemini-3.8-flash` → `gemini-3.1-flash-lite` → `gemini-flash-latest`. On first successful call the winning model is cached in the client instance and reused for the rest of the process lifetime. Resolution is re-attempted only after a later call fails on the cached model.

##### Failure Classification:
| Condition | Handling |
|---|---|
| 429 quota | Non-retryable within the request; fall through the chain, then to Taxonomy |
| 503 high demand | Retry 2× with exponential backoff, then fall through |
| Timeout | Retry once, then fall through |
| Malformed or schema-invalid output | Retry once with a stricter instruction, then fall through |
| Missing API key | Skip the LLM entirely; go straight to Taxonomy |

##### Acceptance Criteria:
- **WHEN** the first model in the chain returns 503 twice **THEN** the client advances to the next model and the successfully resolved model is reused for subsequent calls.
- **WHEN** every model in the chain fails **THEN** the LLM Engine raises `LLMUnavailable` and ST-102 falls back to the Taxonomy Engine.
- **WHEN** the API key is absent or invalid **THEN** no network call is attempted and the stage completes via Taxonomy.
- **WHEN** a call succeeds **THEN** `provenance_strategy` on each resulting trait records the model that produced it, and `model_usage` records token counts.

##### 2.2.4 ANA-204: Schema-Constrained Structured Output

##### Description:
Constrain the model to emit exactly the trait schema rather than parsing free text.

##### Configuration:
`generationConfig.responseMimeType = "application/json"` together with a `responseSchema` derived from the Pydantic `TraitExtraction` model, which Gemini supports natively. `thinkingConfig.thinkingBudget = 0` is required on thinking-capable models — without it the output budget is consumed by reasoning tokens and an empty response is returned.

##### Belt-and-braces validation:
Provider-level schema constraint is not trusted on its own. The response is additionally parsed into the Pydantic model and validated; on `ValidationError` the call is retried once with the validation error appended to the prompt as corrective feedback. A second failure falls through to the next model, then to Taxonomy.

##### Acceptance Criteria:
- **WHEN** a call is made **THEN** `responseMimeType` is `application/json` and `responseSchema` matches the `TraitExtraction` model.
- **WHEN** the model returns output failing Pydantic validation **THEN** the system retries once with the error as feedback, and falls through to Taxonomy if that also fails.
- **WHEN** `thinkingConfig.thinkingBudget` is not sent to a thinking-capable model **THEN** the request is not considered conformant.

##### 1.2.5 ANA-205: Deterministic Taxonomy Fallback Engine

##### Description:
Guarantee the stage always produces a usable result, with no network and no key, so the demo is never blocked on quota.

##### Mechanism:
1. **Keyword taxonomies** per type — a curated dictionary mapping phrases to canonical labels, e.g. `"trail running" | "trail runner" | "ultrarunning" → trail running`
2. **Section-aware extraction** — the reader markdown's `### About`, `### Experience`, `### Skills` headings delimit regions, and a trait found in `Skills` gets higher confidence than one found in a job title
3. **Pattern rules** for `needs` and `dealbreakers` — `"looking for"`, `"I want"`, `"must have"`, `"no ... "`, `"dealbreaker"` are high-precision signal phrases
4. **Graceful empty result** — if nothing matches, return an empty trait list and `low_confidence = true`. Never fabricate to avoid an empty result

##### Acceptance Criteria:
- **WHEN** the LLM is unavailable and the profile contains "trail runner" **THEN** Taxonomy produces a `hobby` trait with label `trail running`, `provenance_source` set to the source it was found in, and `generated_by = 'taxonomy'`.
- **WHEN** the LLM is unavailable and the profile yields no taxonomy matches **THEN** an empty trait list is returned with `low_confidence = true`, and the system does not invent traits.
- **WHEN** Taxonomy runs **THEN** the call completes in under 50ms and makes zero network requests.
- **WHEN** Taxonomy produces a result **THEN** the same provenance and confidence rules in ANA-202 apply unchanged.

##### 1.2.6 ANA-206: Agent Persona Generation with Guardrails

##### Description:
Give each person an agent with a stable identity, a distinct voice, and — most importantly — hard behavioural limits.

##### Persona Fields:
| Field | Description |
|---|---|
| `agent_name` | Short, memorable, gender-neutral where possible — "Scout", "Anchor", "Ember" |
| `tone` | One of `warm`, `dry`, `playful`, `direct`, `poetic` — drawn deterministically from `person_id` so the same person always gets the same voice |
| `summary` | One sentence describing the person, grounded in their traits |
| `guardrails` | The hard behavioural limits, stored and enforced |

##### Guardrails (persisted, and injected into every ST-103 prompt):
- "Never claim to have met this person in person."
- "Never state a fact that is not in their two sources or derived traits."
- "Never invent a biography, job, or location detail."
- "Never propose anything that violates a stated dealbreaker."
- "Reference real shared interests or a real trait, or say you are still figuring that out."

##### Voice Differentiation:
Two people with overlapping traits must not sound identical. `tone` is assigned deterministically from `person_id` and is included in the dating prompt, and each persona is given a distinct verbal tic derived from its dominant trait type — e.g. a `poetic` agent opens with an image, a `dry` agent opens with an observation. The dating prompt is explicitly told that the two agents must not mirror each other's register.

##### Acceptance Criteria:
- **WHEN** a persona is generated **THEN** `agent_name` is unique across all people in the system; on collision, a numeric suffix is appended deterministically.
- **WHEN** the same person is analysed twice **THEN** `agent_name` and `tone` are identical, because both derive from `person_id`.
- **WHEN** a persona is created **THEN** `guardrails` is populated and non-empty.
- **WHEN** `tone` is assigned **THEN** it is one of the five valid values and is stable for that person.
- **WHEN** two people share three or more traits **THEN** their `tone` values are allowed to coincide, but their `agent_name` values differ.

##### 1.2.7 ANA-207: Analysis Idempotency and Re-analysis

##### Description:
Re-running analysis must be safe and must not duplicate rows.

##### Behaviour:
- `profile_traits` and `agent_personas` are keyed to the person; re-analysis replaces rows in a single transaction
- `agent_persona.name` is preserved across re-analysis so that date transcripts referencing an agent remain valid
- `model_usage` and `error_log` are append-only — usage history is never overwritten
- Analysis is permitted for `ingest_state` of `ingested` or `partial`, and refused with 409 `NOT_INGESTED` for `pending` or `failed`

##### Acceptance Criteria:
- **WHEN** a person is analysed twice **THEN** `count(profile_traits)` equals the count after the first analysis, with no duplicates.
- **WHEN** a person is re-analysed **THEN** the `agent_persona.name` is unchanged from the first analysis.
- **WHEN** analysis is attempted on a person with `ingest_state = 'pending'` **THEN** the system responds 409 with `error_code = 'NOT_INGESTED'`.
- **WHEN** analysis completes **THEN** one `model_usage` row is written per LLM call made, including failed calls.

### 1.3 Project Artifacts

- `api/openapi.yaml` — `POST /v1/people/{person_id}/analyze`, `GET /v1/people/{person_id}/profile`
- `api/external-api.yaml` — Gemini `generateContent` with `responseSchema` and the verified model table
- `design/er_diagram.mmd` — `profile_traits`, `agent_personas`, `model_usage`, `error_log`
- `design/analysis_flow.mmd` — LLM path, Taxonomy fallback, and evidence validation
- `design/requirements.md` — REQ-2 analysis requirements

### 1.4 Requirement Traceability

| Requirement | Acceptance criterion | Implemented by |
|---|---|---|
| REQ-3 | 1. Profile with `hobbies`, `interests`, `needs` (plus 5 more types) | ANA-201 |
| REQ-3 | 2. Every trait carries `confidence`, `provenance_source`, `evidence_quote` | ANA-201, ANA-202 |
| REQ-3 | 3. Trait in both sources → `cross_source` | ANA-202 |
| REQ-3 | 4. LLM unavailable → Taxonomy fallback, still a valid profile | ANA-205 |
| REQ-3 | 5. Schema validation failure → retry once → Taxonomy | ANA-204 |
| REQ-3 | 6. Exactly one `agent_personas` row with `agent_name` and `tone` | ANA-206, ANA-207 |
| REQ-3 | 7. Fewer than 3 total traits → low-confidence, nothing invented | ANA-201 |
| REQ-3 | 8. Strip instruction-like text before the LLM | ING-106 (ST-101), applied here |
| REQ-6 | 3. Profile page shows traits, confidence, and source | `GET /v1/people/{id}/profile` |
| REQ-6 | 6. Fully operational without an LLM key | ANA-205 |
| REQ-6 | 7. Typed error, not a 500, on recoverable failure | ANA-203, `NOT_INGESTED` 409 |
| REQ-7 | 2. Every LLM call recorded in `model_usage` | ANA-203, §2.2.5 |
| REQ-7 | 3. Verbatim `evidence_quote` stored | ANA-202 |
| REQ-7 | 4. Errors stored in `error_log` with file, function, message | §2.2.5 |
| REQ-7 | 5. API key never logged or persisted | NFR-203 |

### 1.4 Dependencies

- `google-genai` or direct `httpx` calls to the `generateContent` REST endpoint
- `pydantic` v2 — `TraitExtraction`, `AgentPersona`, `SourceProfilePayload`; schemas derived to JSON Schema for `responseSchema`
- `tenacity` — retry on 503
- SQLAlchemy 2.x async + `asyncpg`
- `structlog` — model-resolution and validation logging

---

## Section 2: Non Functional Requirements

### 2.1 Infrastructure and Deployment

#### 2.1.1 Overview

The stage depends on exactly one external service — Gemini — and its entire purpose in the deployment design is to make that dependency optional. The assessment key's free-tier quota and demand conditions make a hard dependency on any single model unsafe: `gemini-3.8-flash` was under sustained 503, the Pro tier returned 429, and `gemini-2.5-flash` is retired outright. If analysis were written against one model name, the demo would break in front of an audience. So the Taxonomy Engine exists not as a nice-to-have but as the thing that makes the demo reliable: with the LLM entirely unavailable, all 25 people still receive traits, agents, rankings, and dates, just with visibly plainer language.

The API key is read from the `LLM_API_KEY` environment variable at startup. It is never logged, never echoed in an error response, never persisted, and the `.env` file holding it is gitignored. The key supplied for this build was pasted into chat and should be rotated after the assessment.

#### 2.1.2 Requirement Details

- **NFR-201: Model Resolution Must Be Dynamic**
- **NFR-202: No Network Dependency for the Fallback Path**
- **NFR-203: Key Confidentiality**

##### 2.1.2.1 NFR-201: Model Resolution Must Be Dynamic

##### Description:
No model name is hardcoded at a call site. The client resolves the chain once, caches the winner, and re-resolves on failure. Adding a model is a one-line change to the chain constant.

##### Acceptance Criteria:
- **WHEN** the primary model is 503 **THEN** the client transparently uses a different model without any caller change.
- **WHEN** a cached model later starts failing **THEN** the client re-resolves the chain rather than repeatedly failing on the stale pin.

##### 2.1.2.2 NFR-202: No Network Dependency for the Fallback Path

##### Description:
The Taxonomy Engine requires no key, no network, and no model resolution, and completes in under 50ms. The full pipeline — ingest, analyse, rank, date — must complete with the LLM disabled.

##### Acceptance Criteria:
- **WHEN** `LLM_API_KEY` is unset **THEN** `POST /v1/people/{id}/analyze` still returns 200 with `generated_by = 'taxonomy'`.
- **WHEN** the network is unavailable entirely **THEN** analysis of already-ingested data still succeeds via Taxonomy.

##### 2.1.2.3 NFR-203: Key Confidentiality

##### Description:
The API key must never appear in logs, responses, error messages, or the database.

##### Acceptance Criteria:
- **WHEN** any log record is emitted **THEN** the key value does not appear in it.
- **WHEN** an error occurs during a model call **THEN** the response contains no portion of the key.
- **WHEN** the repository is committed **THEN** no file containing the key is tracked, and `.env` is listed in `.gitignore`.

### 2.2 Architecture and System Design

#### 2.2.1 Security and Compliance

- **Retrieved text is untrusted input and is treated as such.** Sanitised at the ingestion boundary (ING-106) and passed to the model inside explicit delimiters, with the system prompt stating that content between the delimiters is data to be analysed and never instructions to follow. This is the defence against a bio crafted to hijack the analysis.
- **No trait may exist without provenance.** ANA-202's quote validation is a correctness control as much as a security one: it makes fabricated claims structurally difficult to persist.
- **Guardrails are stored, not improvised.** Persona guardrails are persisted in `agent_personas.guardrails` and injected verbatim into every ST-103 prompt, so the behavioural limits cannot drift between conversations.
- **Key handling** as in NFR-203.
- **No PII beyond what is already public.** The stage stores only what the public endpoints returned, plus derived traits. It does not infer or store sensitive personal attributes.

#### 2.2.2 System Performance

| Metric | Target | Basis |
|---|---|---|
| LLM extraction call (25 people, concurrency 4) | 2–4s per person, ~30s total | Flash-tier latency |
| Taxonomy extraction | < 50ms | Pure regex over source text |
| Provenance validation | < 2ms per person | Substring search over ≤12k chars |
| Schema retry overhead | +1 call worst case | Only on validation failure |
| `GET /v1/people/{id}/profile` | < 80ms | Two indexed queries |

Latency is dominated entirely by the LLM path, which is why concurrency of 4 and a single-pass design matter: 25 sequential LLM calls would take several minutes, which does not fit a live demo.

#### 2.2.3 Availability and Reliability

Availability here means the analysis stage always produces output, and the mechanisms are specific:

- Ordered model chain, cached resolution, and re-resolution on failure
- Two retries with backoff on 503 before falling through
- Pydantic validation with one corrective retry before falling through
- Taxonomy Engine as the terminal fallback — no network, no key, no failure mode
- `error_log` captures every failed attempt with its cause, so a fallback is explainable after the fact
- Analysis is transactional: a crash mid-write leaves the previous trait set intact rather than a half-populated one

#### 2.2.4 Cost Efficiency

All models used are free-tier. The efficiency measures that matter:

- **Batch across people, not across traits** — one call per person, never one call per trait. 25 people = 25 calls, not 300.
- **Bounded source text** — 12 000 characters maximum input keeps tokens predictable and cost negligible even if the key were upgraded.
- **No retry storm** — the retry budget is capped at 2 per call, so a quota-exhausted key cannot generate hundreds of failing calls.
- **Taxonomy as a cost ceiling** — when the LLM path is undesirable, disabling it via configuration drops cost to zero with no functional loss.

#### 2.2.5 Traceability and Observability

- `model_usage` records, per call: model name, provider, stage, prompt and completion token counts, latency, success flag, and error type. This table is the evidence for the write-up's model-availability findings.
- `provenance_strategy` on each trait records whether `llm` or `taxonomy` produced it, and the model name when `llm`.
- `generated_by` on the profile response makes the LLM-vs-Taxonomy split visible to the user and the grader on every request.
- Structured logs on model resolution, schema validation failure, and fallback activation.
- **Evidence coverage rate** is the key quality metric: the proportion of persisted traits with a verified quote. A run below ~70% indicates the source was thin or the model was over-creative, and should be visible.

---

## Section 3: In Scope and Out Scope

### 3.1 In Scope Details

- Typed trait extraction across the five LLM-extracted types
- Code-derived `conversation_opener`, `date_idea`, and `dealmaker` traits
- Evidence-quote validation and the fabrication guard
- Provenance assignment: `linkedin`, `instagram`, `cross_source`, `inferred`
- Per-type confidence scoring with demotion and drop thresholds
- LLM Engine with an ordered, dynamically resolved model chain
- Provider-level `responseSchema` constraint plus Pydantic validation with corrective retry
- Deterministic Taxonomy Engine fallback
- Agent persona generation: name, tone, summary, guardrails
- Deterministic tone assignment and unique name assignment
- Analysis idempotency and re-analysis safety
- `model_usage` and `error_log` recording
- Reduced-confidence handling for `partial` ingest state

### 3.2 Out Scope Details

- Any trait not supported by the retrieved source text
- Inferring traits from a person's *network* — connections, mutual follows, or follower identity are not analysed
- Sentiment or personality-type analysis of the person themselves
- Inferring or storing sensitive attributes: age, ethnicity, religion, sexuality, health
- Multi-step or agentic analysis loops — this is a single structured call, not a reasoning chain
- Fine-tuning, embeddings, or vector similarity. Typed sets are sufficient and far more explainable
- Editing or correcting a person by hand from the UI
- Image, video, or visual content analysis
- Caching LLM responses across people, or any cross-person data sharing

---

## Section 4: Solution Diagrams

#### 4.1 UI/UX Design Diagram

The profile page is the first screen after link submission, per the required flow. It shows the agent identity card, the five trait sections each displaying label, confidence bar, provenance chip, and the evidence quote, and a source provenance panel showing which strategy won per source.

**Diagram Location:** `design/analysis_flow.mmd`

#### 4.2 Architecture Design Diagram

**Diagram Location:** `design/er_diagram.mmd` (entities: `profile_traits`, `agent_personas`, `model_usage`, `error_log`)

#### 4.3 Infrastructure Design Diagram

The FastAPI process hosts the LLM Engine (with model resolution cache) and the Taxonomy Engine behind a single `AnalysisEngine` interface. Outbound: Gemini `generateContent`. Downstream: PostgreSQL, and ST-103 which consumes `agent_personas` and `profile_traits`.

**Diagram Location:** `api/external-api.yaml` (Gemini path definition)
