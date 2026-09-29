## SOLDEF-AGENTD-ST-104: Deterministic Compatibility Ranking

### Project Details
- **Project ID:** PROJ-AGENTD
- **Project Name:** Agentic Dating Site

### Story Details
- **Story ID:** ST-104
- **Story Name:** Deterministic Compatibility Ranking
- **Story Description:**
  Score every pair of analysed people across eight weighted, individually inspectable dimensions, apply dealbreaker penalties, and produce per-person ranked lists plus an N×N matrix — with a human-readable rationale explaining exactly why each pair scored what it did.

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

This stage is deliberately the least "AI" part of the product. Everything before it involves scraping a hostile web and asking a language model for structured data; this stage is arithmetic over typed sets. That is not a compromise — it is the point.

The reason is **explainability**. A user who is told they are 84% compatible with someone needs to be able to ask *why*, and the answer has to be specific: shared hiking and coffee, aligned on sustainability, a conflict on distance. That answer cannot be produced by a similarity score, and a score that cannot be decomposed cannot be trusted. So compatibility is computed across eight named dimensions with fixed weights, each dimension writes its own sub-score, and the rationale is assembled from the same inputs that produced the numbers. The number and the explanation cannot disagree, because they come from the same computation.

**Set operations, not embeddings.** Because ST-102 emits traits as typed labels with confidence and provenance, overlap between two people is a Jaccard-style computation over labelled sets. Semantic similarity would be more forgiving of vocabulary differences — someone who writes "trail running" and someone who writes "ultrarunning" would be recognised as similar by an embedding model but not by string matching. The mitigation is a synonym map applied during normalisation (`trail runner` → `trail running`, `ultrarunning` → `trail running`), which recovers most of the benefit with none of the opacity.

**Dealbreakers are penalties, not filters.** A hard exclusion would be the obvious design, and it is wrong: a person with one dealbreaker is not a zero candidate, they are a candidate whose score reflects that constraint. A −15 penalty per violated dealbreaker, capped at −30, lets a strong shared foundation survive a single mismatch while preventing dealbreakers from being ignored. Mutual top picks are computed symmetrically, and a pair where both rank each other first is flagged, because mutual interest is a meaningfully different signal from a high score alone.

The whole stage is idempotent, runs in well under a second for 25 people, and requires no LLM call at all.

### 1.2 Requirement Details

- **RNK-401: Trait Normalisation and Canonical Label Mapping**
- **RNK-402: Eight Weighted Scoring Dimensions**
- **RNK-403: Dealbreaker Penalty Application**
- **RNK-404: Per-Person Rank and Mutual Top Pick Computation**
- **RNK-405: Human-Readable Rationale Generation**
- **RNK-406: Idempotent Re-computation**
- **RNK-407: Matrix and Pair Endpoint Payloads**

##### 1.2.1 RNK-401: Trait Normalisation and Canonical Label Mapping

##### Description:
Normalise trait labels before any comparison, so that vocabulary differences do not read as incompatibility.

##### Normalisation Steps:
1. Lowercase and trim
2. Collapse internal whitespace
3. Strip leading articles and filler ("a love of", "really into", "passionate about")
4. Singularise simple plurals
5. Map through `CANONICAL_LABELS` synonym table
6. Drop generic labels below a specificity threshold

##### Synonym Table (representative):
| Raw variants | Canonical |
|---|---|
| trail running, trail runner, ultrarunning, trail runs | trail running |
| coffee, filter coffee, third-wave, specialty coffee, third wave coffee | specialty coffee |
| hiking, hikes, trekking, trail hikes | hiking |
| sustainability, sustainable, eco, climate-positive | sustainability |
| fintech, financial technology | fintech |
| remote work, wfh, work from home, distributed | remote work |

##### Specificity Filter:
Labels with fewer than 3 characters after normalisation, or in `STOP_LABELS = {stuff, things, misc, other, general, fun, life, friends}`, are dropped. A person is not compatible because they both like "stuff".

##### Acceptance Criteria:
- **WHEN** one person has `trail runner` and another has `ultrarunning` **THEN** both normalise to `trail running` and register as a shared hobby.
- **WHEN** a label is `"a love of photography"` **THEN** it normalises to `photography`.
- **WHEN** a label normalises to fewer than 3 characters or is in `STOP_LABELS` **THEN** it is excluded from all scoring.
- **WHEN** two labels differ only in case or whitespace **THEN** they are treated as identical.

##### 1.2.2 RNK-402: Eight Weighted Scoring Dimensions

##### Description:
Score each pair across eight named dimensions. Weights sum to 100.

##### Eligibility Filter (REQ-5.7):
A person is eligible for ranking only when their `ingest_state` is `ingested` or `partial` **and** they have at least one persisted trait. Persons with `ingest_state = 'failed'` or `pending` are excluded from scoring entirely — not scored at zero, which would rank them below genuinely incompatible people and misrepresent an ingestion failure as a compatibility result.

##### Weights:
| # | Dimension | Weight | Computation |
|---|---|---|---|
| 1 | `interest_overlap` | 25 | `25 * Jaccard(interests_A, interests_B)` |
| 2 | `hobby_complement` | 18 | `18 * mean(overlap, complementarity)` where complementarity favours distinct-but-adjacent pairs |
| 3 | `value_alignment` | 15 | `15 * Jaccard(values_A ∪ dealmakers_A, values_B ∪ dealmakers_B)` |
| 4 | `need_satisfaction` | 12 | For each need of A, full credit if it appears in B's values/interests/hobbies, partial at 0.5; averaged over both directions |
| 5 | `energy_match` | 10 | Activity-intensity pairing: both high, both low, or mixed |
| 6 | `life_stage` | 8 | Career-stage, relocation, and family-signal compatibility from bio and headline text |
| 7 | `geography` | 7 | Same city 7, same country 4, compatible relocation 2, distant 0 |
| 8 | `dealbreaker_conflict` | −15 each | Applied per violated dealbreaker, capped at −30 |

##### Set Similarity:
`Jaccard(A, B) = |A ∩ B| / |A ∪ B|`, computed on normalised labels filtered to a minimum set size of 2. With a single trait the intersection is unstable, so the dimension is skipped and its weight redistributed proportionally across the remaining dimensions, keeping the total at 100.

##### Hobby Complement:
Pure overlap rewards identical people, which is not what makes a good match. So hobby scoring is the mean of two components: **overlap** (shared activities) and **complementarity** (distinct activities that are not in conflict, scored 0.5 for each side having something the other lacks, capped at 1.0). Two people who both only run trails get high overlap and no complementarity; two people who run trails and play tennis get moderate on both.

##### Acceptance Criteria:
- **WHEN** a pair shares 3 of 5 interests **THEN** `interest_overlap` equals `25 * (3/7)` rounded to 2 decimals.
- **WHEN** a person has exactly 1 interest **THEN** `interest_overlap` is skipped and its weight is redistributed proportionally.
- **WHEN** a person has an interest labelled in `STOP_LABELS` **THEN** that label is excluded before any set operation.
- **WHEN** a person has `ingest_state = 'failed'` **THEN** they are excluded from scoring entirely and do not appear in any pair or in the matrix.
- **WHEN** two people are scored **THEN** the eight dimension values sum to the reported `total_score`, and the sum is between −30 and 100.
- **WHEN** two people share no traits in any dimension **THEN** `total_score` is 0, not negative.
- **WHEN** scoring completes **THEN** each of the eight dimension scores is individually retrievable.

##### 1.2.3 RNK-403: Dealbreaker Penalty Application

##### Description:
Apply a penalty per violated dealbreaker rather than excluding the pair outright.

##### Violation Detection:
A dealbreaker of person A is violated by person B when the dealbreaker's canonical label appears in B's traits, or when a pattern rule matches B's text — e.g. a `long-distance only` dealbreaker is violated when B's `location` differs from A's and neither has a `remote work` trait.

##### Penalties:
- −15 per violated dealbreaker
- Capped at −30 total
- `total_score` is floored at 0
- Every penalty is recorded with the specific dealbreaker and the party that violated it

##### Acceptance Criteria:
- **WHEN** one dealbreaker is violated **THEN** 15 points are deducted and the violated dealbreaker is named in `conflict_signals`.
- **WHEN** three dealbreakers are violated **THEN** the total penalty is 30, not 45.
- **WHEN** the penalty would push `total_score` below 0 **THEN** `total_score` is 0 and the raw score is preserved for transparency.
- **WHEN** no dealbreakers exist for either party **THEN** no penalty is applied and `dealbreaker_conflict` is 0.

##### 1.2.4 RNK-404: Per-Person Rank and Mutual Top Pick Computation

##### Description:
Turn pair scores into the per-person ranked list the product actually presents.

##### Rules:
- `rank_for_a` — A's rank among all candidates, ascending by `total_score`, **ties broken by `person_id`** so ranks are stable across runs
- `rank_for_b` — the mirror
- `mutual_top_pick` — true only when `rank_for_a = 1` **and** `rank_for_b = 1`
- Ranks are dense: 1, 2, 2, 4 is not used; equal scores share a rank and the next rank increments by one
- Only pairs where both parties are `ingested` or `partial` **and** have at least one trait are scored

##### Acceptance Criteria:
- **WHEN** 25 people are ranked **THEN** each person has exactly 24 candidates.
- **WHEN** two pairs tie on `total_score` **THEN** they share a rank and the next rank increments by one.
- **WHEN** a pair ranks first for both parties **THEN** `mutual_top_pick = true` for that pair.
- **WHEN** a pair ranks first for A and third for B **THEN** `mutual_top_pick = false`.
- **WHEN** `POST /v1/rank` is called twice with unchanged data **THEN** every rank is identical, because ties break on `person_id`.
- **WHEN** fewer than 2 analysable people exist **THEN** the endpoint responds 409 with `error_code = 'INSUFFICIENT_PEOPLE'`.

##### 1.2.5 RNK-405: Human-Readable Rationale Generation

##### Description:
Produce the explanation from the same inputs that produced the numbers, so the two can never disagree.

##### Rationale Structure:
`"{name} shares {n} interests and {m} hobbies with {other}. Strong alignment on {values}. {Dealbreaker sentence if applicable} Scored {score}/100."`

##### Shared and Conflict Signals:
- `shared_signals` — normalised labels appearing in both people's trait sets, ordered by trait-type priority (dealbreaker-adjacent concerns first, then values, needs, hobbies, interests), capped at 8
- `conflict_signals` — violated dealbreakers and value contradictions, each with the specific labels involved

##### Requirement:
The rationale is **template-generated in code**, not LLM-written. This is deliberate: an explanation that is generated independently of the computation can contradict it, and an explanation that contradicts the score is worse than no explanation.

##### Acceptance Criteria:
- **WHEN** a pair is returned with a `rationale` **THEN** every shared interest and hobby count in the text matches `shared_signals`.
- **WHEN** a dealbreaker is violated **THEN** `conflict_signals` names the dealbreaker and the rationale includes a sentence about it.
- **WHEN** a pair has no shared traits **THEN** the rationale says so plainly rather than inventing common ground.
- **WHEN** the rationale is generated **THEN** no LLM call is made.

##### 1.2.6 RNK-406: Idempotent Re-computation

##### Description:
Re-running replaces prior results rather than appending.

##### Behaviour:
- `match_scores` is uniquely keyed on `(person_a_id, person_b_id)` with `person_a_id < person_b_id` enforced by a `CHECK` constraint, so a pair has one canonical row
- Re-computation performs an upsert per pair inside one transaction
- Recomputing does not delete `date_sessions` — the conversations that already happened remain, and stale predictions are retained on the session for comparison against the verdict the agents actually reached

##### Acceptance Criteria:
- **WHEN** `POST /v1/rank` is called twice **THEN** `count(match_scores)` equals the number of unique pairs, with no duplicates.
- **WHEN** a person's traits change and ranking re-runs **THEN** their `match_scores` rows are updated in place.
- **WHEN** ranking re-runs **THEN** existing `date_sessions` are untouched and their `predicted_score` is preserved.
- **WHEN** ranking re-runs and a pair's score changes materially **THEN** the stored `predicted_score` on existing sessions is left intact for audit.

##### 1.2.7 RNK-407: Matrix and Pair Endpoint Payloads

##### Description:
Support the three read shapes the UI needs.

##### Endpoints:
| Endpoint | Shape | Consumer |
|---|---|---|
| `GET /v1/people/{id}/rankings` | Flat list, rank ascending, limit default 10 | Person's "who fits me" screen |
| `GET /v1/matches/matrix` | N×N nested array, `null` diagonal, plus flat pair list | Heatmap |
| `POST /v1/rank` | Trigger, returns counts and the top match | Run summary |

##### Matrix Construction:
Built in memory from a single query over `match_scores`, not from N separate lookups, so 25 people yields one query rather than 25.

##### Acceptance Criteria:
- **WHEN** the matrix is requested for 25 people **THEN** it contains 25 rows of 25 entries with `null` on every diagonal position.
- **WHEN** the matrix is returned **THEN** `matrix[i][j] == matrix[j][i]` for all off-diagonal entries.
- **WHEN** `GET /v1/people/{id}/rankings` is called with `limit = 10` **THEN** exactly 10 entries are returned, ordered by rank ascending.
- **WHEN** a person has been analysed **THEN** their ranking response includes `agent_name` and `total_candidates`.

### 1.3 Project Artifacts

- `api/openapi.yaml` — `POST /v1/rank`, `GET /v1/people/{person_id}/rankings`, `GET /v1/matches/matrix`
- `design/er_diagram.mmd` — `match_scores` entity and its `CHECK (person_a_id < person_b_id)` constraint
- `design/ranking_pipeline.mmd` — dimension computation and gate ordering
- `design/requirements.md` — REQ-4 ranking requirements

### 1.4 Requirement Traceability

| Requirement | Acceptance criterion | Implemented by |
|---|---|---|
| REQ-5 | 1. Exactly `N * (N-1) / 2` rows, one per unordered pair | RNK-404, `CHECK (person_a_id < person_b_id)` |
| REQ-5 | 2. Exactly eight named dimensions, each with a declared weight | RNK-402 |
| REQ-5 | 3. Deterministic — identical input, identical score | RNK-404 (`person_id` tiebreak), NFR-403 |
| REQ-5 | 4. Dealbreaker penalty applied and recorded in `conflict_signals` | RNK-403 |
| REQ-5 | 5. `rank_for_a`, `rank_for_b`, `mutual_top_pick` populated for every pair | RNK-404 |
| REQ-5 | 6. Human-readable `rationale` naming the strongest shared signals | RNK-405 |
| REQ-5 | 7. Persons with `ingest_state = 'failed'` excluded | RNK-402 eligibility filter |
| REQ-5 | 8. N×N matrix endpoint for visualisation | RNK-407 |
| REQ-6 | 4. Rankings screen shows every person's ranked list | RNK-407 |
| REQ-7 | 3. Signals traceable to verbatim quotes | `shared_signals` → `profile_traits.evidence_quote` |

**Note on the pair count in REQ-5.1:** `N` is the number of **analysable** people — those with `ingest_state` of `ingested` or `partial` and at least one trait — not the total roster size. Persons with `ingest_state = 'failed'` are excluded per REQ-5.7, so for a 25-person roster where 1 person failed ingestion, the system produces 300 rows, not 276. The response reports `people_ranked` alongside `pairs_scored` so the basis of `N` is always explicit rather than implied.

### 1.4 Dependencies

- `profile_traits` from ST-102 — the only input; no LLM call in this stage
- SQLAlchemy 2.x async + `asyncpg`
- `pydantic` v2 — ranking response models
- Pure-stdlib set operations; no numpy, no similarity library, no embedding model

---

## Section 2: Non Functional Requirements

### 2.1 Infrastructure and Deployment

#### 2.1.1 Overview

This stage has no external dependencies at all. No network calls, no model, no API key, no third-party service. It reads two tables and writes one, entirely in process. That is a deliberate architectural position rather than a convenience: a ranking engine that requires a model call to run is a ranking engine that fails when the quota runs out, and a ranking engine whose output cannot be reproduced is a ranking engine nobody can verify.

Operationally this means `POST /v1/rank` is a synchronous request that completes in milliseconds for the target roster size. There is no background task, no queue, and no worker for this stage. The only scaling concern is the O(n²) pair count, which at 25 people is 300 pairs and at 100 people is 4 950 — both comfortably within a single in-memory loop. The design documents the pairwise ceiling explicitly so that a larger roster is a known, bounded cost rather than an unknown one.

#### 2.1.2 Requirement Details

- **NFR-401: Zero External Dependencies**
- **NFR-402: Bounded Pairwise Cost**
- **NFR-403: Deterministic Reproducibility**

##### 2.1.2.1 NFR-401: Zero External Dependencies

##### Description:
The stage makes no network requests and requires no credentials.

##### Acceptance Criteria:
- **WHEN** `POST /v1/rank` runs **THEN** zero outbound HTTP requests are made.
- **WHEN** `LLM_API_KEY` is unset **THEN** ranking still computes fully.

##### 2.1.2.2 NFR-402: Bounded Pairwise Cost

##### Description:
Pairwise scoring is O(n²) with a documented ceiling of 1 000 people (499 500 pairs). Beyond that, a blocking or approximate strategy would be required; the stage refuses rather than degrading silently.

##### Acceptance Criteria:
- **WHEN** 25 people are ranked **THEN** all 300 pairs are scored in under 500ms.
- **WHEN** the analysed population exceeds 1 000 **THEN** the stage refuses with an explicit error rather than proceeding unbounded.

##### 2.1.2.3 NFR-403: Deterministic Reproducibility

##### Description:
The same trait data always produces the same scores, ranks, and rationales. No randomness, no floating-point accumulation order dependence, no hash iteration order dependence.

##### Acceptance Criteria:
- **WHEN** ranking runs twice on unchanged data **THEN** every `total_score`, `rank_for_a`, `rank_for_b`, and `mutual_top_pick` is identical.
- **WHEN** ties are broken **THEN** the tiebreak is `person_id`, which is stable and not dependent on row order.

### 2.2 Architecture and System Design

#### 2.2.1 Security and Compliance

- **No sensitive attribute scoring.** The dimensions operate only on traits ST-102 already vetted, and ST-102 explicitly excludes inferring age, ethnicity, religion, sexuality, or health. Ranking adds no such inference of its own. This matters more in a dating context than in most, because a compatibility score is a claim about a person.
- **Provenance preserved end to end.** Every `shared_signal` traces back through `match_scores` → `profile_traits` → `evidence_quote` → source payload → winning strategy. Nothing in a ranking is unsourced.
- **No inferred personal data from the ranking stage itself.** The stage reads traits; it does not derive new ones.
- **Deletion cascades.** Deleting a person removes their `match_scores` rows on both sides, so no derived claim about a removed person survives.

#### 2.2.2 System Performance

| Metric | Target | Basis |
|---|---|---|
| Normalisation of all traits (25 people) | < 10ms | Synonym table lookup |
| 300 pair scores | < 500ms | Pure set operations |
| Rank assignment | < 5ms | Two sorts of 25 elements |
| Rationale generation | < 20ms | Template substitution |
| `GET /v1/matches/matrix` | < 150ms | Single query, in-memory assembly |
| `POST /v1/rank` end to end | < 1s | 300 upserts in one transaction |

#### 2.2.3 Availability and Reliability

- No external dependency, so the only failure mode is the database
- One transaction per ranking run: a failure leaves the previous `match_scores` intact rather than a partially updated set
- Idempotent upserts keyed on the ordered pair, so a re-run cannot duplicate
- Weight redistribution handles sparse traits without producing a divide-by-zero or a silently distorted total
- Dealbreaker penalty is capped, so a pathological trait set cannot drive scores arbitrarily negative
- Scores are floored at 0 with the raw value preserved, so the stored data remains auditable

#### 2.2.4 Cost Efficiency

Zero marginal cost — no model calls, no external services, no compute beyond a single in-process pass. The `weighted_candidate_pool` mechanism bounds the work: only top-ranked candidates are dated by default, so the O(n²) scoring is not followed by an O(n²) LLM bill. At 25 people, ranking costs roughly 0.5 seconds of CPU and nothing else.

#### 2.2.5 Traceability and Observability

- `match_scores.dimension_scores` stores all eight sub-scores per pair, so any total can be decomposed and audited
- `shared_signals` and `conflict_signals` store the exact labels behind the number
- `rationale` is generated from the same arrays, so explanation and score are structurally consistent
- `mutual_top_pick` makes symmetric interest queryable
- `predicted_score` on `date_sessions` lets the system compare the ranking's prediction against the verdict the agents actually reached — the only real evaluation signal in the project, and one worth reporting honestly in the write-up
- Rank stability on `person_id` tiebreak means a re-run with identical data produces a diff of zero, which makes regressions detectable

---

## Section 3: In Scope and Out Scope

### 3.1 In Scope Details

- Trait normalisation, synonym canonicalisation, and specificity filtering
- Eight weighted dimensions with per-dimension sub-scores
- Jaccard set similarity and weighted redistribution for sparse traits
- Hobby complementarity as distinct from hobby overlap
- Need-satisfaction scoring in both directions
- Energy, life-stage, and geography dimensions
- Dealbreaker detection and capped penalty application
- Per-person rank, dense ranking, and `person_id` tiebreak
- Mutual top pick detection
- Shared-signal and conflict-signal extraction
- Template-generated rationale consistent with the computed score
- Idempotent re-computation preserving date sessions
- Matrix, ranking-list, and run-summary payloads

### 3.2 Out Scope Details

- Machine-learned ranking, collaborative filtering, or preference learning
- Embeddings, vector similarity, or any semantic model
- Any LLM call in this stage
- Collaborative signals: who else liked whom, mutual friend networks, popularity priors
- Feedback loops from actual dates back into the scoring weights
- Weight tuning or exposing weights as a user-configurable setting
- Ranking people who have not been analysed
- Inferring new traits, attributes, or preferences during scoring
- Compatibility categories or labels such as "introvert match" — scores and signals only
- Any sensitive-attribute dimension: race, religion, nationality, disability, age

---

## Section 4: Solution Diagrams

#### 4.1 UI/UX Design Diagram

The rankings screen shows a person's ranked list with the total score, an eight-segment contribution bar breaking the score into its dimensions, shared-signal chips and conflict-signal chips, and the rationale sentence. A matrix view renders the N×N heatmap for cross-person comparison.

**Diagram Location:** `design/ranking_pipeline.mmd`

#### 4.2 Architecture Design Diagram

**Diagram Location:** `design/er_diagram.mmd` (entity: `match_scores`)

#### 4.3 Infrastructure Design Diagram

The Ranking Engine runs entirely in-process within the FastAPI application. Inputs: `profile_traits` (single query). Computation: in-memory set operations. Output: `match_scores` (single transaction). No network, no model, no worker.

**Diagram Location:** `api/openapi.yaml` (`/v1/rank`, `/v1/people/{person_id}/rankings`, `/v1/matches/matrix`)
