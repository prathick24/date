# Requirements Document

## Introduction

This document specifies the functional requirements for the **Agentic Dating Site**, a platform where every person is represented by a software agent. That agent reads its person's two public profiles, forms a dated opinion of them, and then dates on their behalf. The agents date each other.

The system is decomposed into four pipeline stages, each documented as an independent Solution Design document:

| Story ID | Story Name | SD Document |
|----------|------------|-------------|
| ST-101 | Source Ingestion Pipeline | `SOLDEF_AGENTD_ST-101_INGESTION.md` |
| ST-102 | Agent Profile Analysis | `SOLDEF_AGENTD_ST-102_ANALYSIS.md` |
| ST-103 | Agent Dating Engine | `SOLDEF_AGENTD_ST-103_AGENT_DATING.md` |
| ST-104 | Compatibility Ranking | `SOLDEF_AGENTD_ST-104_RANKING.md` |

## Glossary

- **Person**: A real human being represented in the system. Exactly two official sources are permitted per person: a public LinkedIn profile and a public Instagram profile.
- **Agent**: The software construct that acts on a Person's behalf. An agent is created for every Person and may only speak from its Person's two sources.
- **Source**: One of the two permitted information origins for a Person — `linkedin` or `instagram`.
- **Strategy**: A specific technical method used to retrieve a Source. A Source may have multiple Strategies attempted in order until one succeeds.
- **Provenance**: Per-field metadata recording which Source, which Strategy, and what verbatim quote produced a given value.
- **Trait**: A single analysed attribute of a Person — a hobby, interest, need, value, dealbreaker, dealmaker, conversation opener, or date idea.
- **Compatibility Dimension**: One of the eight named axes used to score a pair of People.
- **Date Session**: A single agent-to-agent date between two People, composed of an ordered transcript of messages.
- **Round**: One exchange pair within a Date Session. A standard session runs three rounds.
- **Verdict**: The closing decision of a Date Session — `second_date`, `friendzone`, or `unmatched`.
- **LLM Engine**: The Gemini-backed generator used for analysis, persona creation, and dating dialogue.
- **Taxonomy Engine**: The deterministic, zero-network fallback analyser that produces traits from a weighted keyword taxonomy.
- **Roster**: The seeded set of 25 real People used for the demonstration dataset.

---

## Requirement 1: Two-Source Person Registration

**User Story:** As a user of the site, I want to register a person by pasting their LinkedIn and Instagram links, so that an agent can be created to represent them.

#### Acceptance Criteria

1. WHEN a user submits a person with exactly one LinkedIn URL and exactly one Instagram URL, THEN THE SYSTEM SHALL create a `people` row with `ingest_state` set to `pending`
2. WHEN a person already exists with the same `linkedin_url` OR the same `instagram_handle`, THEN THE SYSTEM SHALL reject the request with `DUPLICATE_PERSON`
3. WHEN the LinkedIn URL is not a valid `linkedin.com/in/{slug}` URL, THEN THE SYSTEM SHALL reject the request with `INVALID_INPUT`
4. WHEN the Instagram handle is not a valid handle of 1–30 characters matching `[A-Za-z0-9._]`, THEN THE SYSTEM SHALL reject the request with `INVALID_INPUT`
5. WHEN either URL is missing, THEN THE SYSTEM SHALL reject the request with `MISSING_SOURCE` naming which of the two sources is absent
6. THE SYSTEM SHALL normalise Instagram input so that `nasa`, `@nasa`, `instagram.com/nasa`, and `https://www.instagram.com/nasa/` all resolve to the canonical handle `nasa`
7. WHEN a person is registered, THEN THE SYSTEM SHALL NOT attempt any network fetch — registration and ingestion are separate concerns

---

## Requirement 2: Multi-Strategy Source Retrieval

**User Story:** As a user, I want the system to actually retrieve my person's public profiles even though both platforms block naive scrapers, so that the agent has real material to read.

#### Acceptance Criteria

1. WHEN ingesting a Person, THEN THE SYSTEM SHALL attempt at least three ordered Strategies for the `linkedin` Source and at least three ordered Strategies for the `instagram` Source
2. WHEN a Strategy returns a usable payload, THEN THE SYSTEM SHALL record that Strategy in `source_profiles.winning_strategy` and stop attempting further Strategies for that Source
3. WHEN every Strategy for a Source fails, THEN THE SYSTEM SHALL record `success = false`, the last `http_status`, and a `failure_reason` describing the block (for example `HTTP 999 bot-blocked`)
4. THE SYSTEM SHALL never fabricate, infer, or hallucinate Source content — a failed Source SHALL yield zero traits
5. WHEN one Source succeeds and the other fails, THEN THE SYSTEM SHALL set `ingest_state` to `partial` and still proceed to analysis using only the successful Source
6. WHEN both Sources fail, THEN THE SYSTEM SHALL set `ingest_state` to `failed` and exclude the Person from ranking
7. THE SYSTEM SHALL apply an `asyncio` timeout to every outbound HTTP call
8. THE SYSTEM SHALL retry only idempotent `GET` requests, using exponential backoff, and SHALL NOT retry on `401` or `999` responses

---

## Requirement 3: Agent Profile Analysis

**User Story:** As a user, I want each person to have a profile page showing what their agent found in them, so that I can see how the agent read my person.

#### Acceptance Criteria

1. WHEN a Person's Sources are ingested, THEN THE SYSTEM SHALL produce a profile containing at minimum `hobbies`, `interests`, and `needs` lists
2. EVERY trait SHALL carry `confidence`, `provenance_source`, and `evidence_quote` recording the verbatim text it was derived from
3. WHEN a trait appears in both Sources, THEN THE SYSTEM SHALL mark its `provenance_source` as `cross_source`
4. WHEN the LLM Engine is unavailable, THEN THE SYSTEM SHALL fall back to the Taxonomy Engine and still return a valid profile
5. WHEN the LLM Engine returns output that fails schema validation, THEN THE SYSTEM SHALL retry once and THEN fall back to the Taxonomy Engine
6. THE SYSTEM SHALL create exactly one `agent_personas` row per Person, with an `agent_name` and a `tone`
7. WHEN a Person has fewer than three total traits across both Sources, THEN THE SYSTEM SHALL mark the profile as low-confidence rather than inventing content
8. THE SYSTEM SHALL strip any instruction-like text found inside fetched Source content before passing it to the LLM

---

## Requirement 4: Agent-to-Agent Dating

**User Story:** As a user, I want to watch the agents actually go out and date, so that I can see the harness dating on each person's behalf.

#### Acceptance Criteria

1. WHEN a Date Session runs, THEN TWO agents SHALL alternate speaking across at least three Rounds, producing at least six messages
2. EVERY message SHALL cite the trait or Source quote it was built from in `cited_evidence`
3. THE SYSTEM SHALL derive the proposed date from the shared hobbies or interests of BOTH People, not from a fixed template
4. WHEN the session completes, THEN THE SYSTEM SHALL record a `verdict` of `second_date`, `friendzone`, or `unmatched` together with `verdict_reasoning`
5. WHEN a Person's `dealbreaker` trait matches the other Person's `trait_type` for the same dimension, THEN THE SYSTEM SHALL reflect the conflict in the verdict reasoning
6. WHEN the LLM Engine fails mid-session, THEN THE SYSTEM SHALL complete the remaining Rounds with the deterministic dialogue engine and set `llm_succeeded = false`
7. THE SYSTEM SHALL record the session even when it is incomplete, with `status` set to `failed` and `rounds_completed` reflecting what was produced
8. NO agent SHALL claim a fact about its Person that is absent from that Person's two Sources

---

## Requirement 5: Compatibility Ranking

**User Story:** As a user, I want every person to have a ranked list of who fits them best, so that I can see the agent's judgement.

#### Acceptance Criteria

1. WHEN ranking is computed for N People, THEN THE SYSTEM SHALL produce exactly `N * (N-1) / 2` `match_scores` rows, one per unordered pair
2. THE SYSTEM SHALL score every pair across exactly eight named Compatibility Dimensions, each with a declared weight
3. THE SYSTEM SHALL compute rankings deterministically — identical input SHALL always produce an identical score
4. WHEN a Person has a `dealbreaker` trait that the other Person violates, THEN THE SYSTEM SHALL subtract the declared penalty and record the violation in `conflict_signals`
5. WHEN rankings are computed, THEN THE SYSTEM SHALL populate `rank_for_a`, `rank_for_b`, and `mutual_top_pick` for every pair
6. THE SYSTEM SHALL store a human-readable `rationale` naming the two or three strongest shared signals
7. THE SYSTEM SHALL exclude Persons whose `ingest_state` is `failed` from all rankings
8. THE SYSTEM SHALL provide an N×N compatibility matrix endpoint for visualisation

---

## Requirement 6: Live End-to-End Operation

**User Story:** As a reviewer, I want to open the site, paste my own links, and watch the whole pipeline run, so that I can verify every feature actually works.

#### Acceptance Criteria

1. WHEN the site starts, THE SYSTEM SHALL expose `GET /health` and `GET /ready` endpoints
2. WHEN a user pastes a LinkedIn and an Instagram URL into the UI, THEN THE SYSTEM SHALL show live progress through ingestion, analysis, and dating
3. WHEN a user opens a Person's profile, THEN THE SYSTEM SHALL display the traits, the confidence, and the source each trait came from
4. WHEN a user opens the rankings screen, THEN THE SYSTEM SHALL display every Person's ranked list
5. THE SYSTEM SHALL ship a seeded demonstration of 25 real People whose pipelines have already been run, reachable without typing anything
6. WHEN the LLM API key is absent, THEN THE SYSTEM SHALL remain fully operational using the Taxonomy Engine and deterministic dating
7. THE SYSTEM SHALL never return a 500 to the browser for a recoverable upstream failure — recoverable failures SHALL surface as a typed error with an actionable message
8. WHEN the same two links are submitted twice, THE SYSTEM SHALL return the existing person rather than creating a duplicate

---

## Requirement 7: Provenance and Auditability

**User Story:** As a reviewer, I want to see exactly where every piece of information came from, so that I can trust the analysis is grounded in the two official sources and nothing else.

#### Acceptance Criteria

1. THE SYSTEM SHALL record every outbound fetch attempt in `source_profiles`, including failed attempts
2. THE SYSTEM SHALL record every LLM call in `model_usage` with model id, token counts, latency, and whether the Taxonomy fallback was used
3. WHEN the analysis produced a trait, THE SYSTEM SHALL store the verbatim `evidence_quote` from the source text
4. THE SYSTEM SHALL store all errors in `error_log` with file name, function name, and message
5. THE SYSTEM SHALL never log, persist, or transmit the LLM API key
6. THE SYSTEM SHALL never store scraped passwords, private messages, or any non-public content — only publicly visible profile text
