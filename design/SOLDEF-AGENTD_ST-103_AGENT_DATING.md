## SOLDEF-AGENTD-ST-103: Agent-to-Agent Dating Engine

### Project Details
- **Project ID:** PROJ-AGENTD
- **Project Name:** Agentic Dating Site

### Story Details
- **Story ID:** ST-103
- **Story Name:** Agent-to-Agent Dating Engine
- **Story Description:**
  Have the generated agents actually date each other — a multi-round, alternating conversation that draws only on real profile evidence, ends in a verdict, and proposes a concrete plan. The transcript is the product's centrepiece: it must read as two distinct people talking, not a template.

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

This is the story that makes the product legible. A ranking table is an assertion; a transcript is evidence. The stage takes two analysed people and produces a conversation their agents genuinely have, where every claim is anchored to a real trait and the outcome is a judgement with stated reasoning.

The design rests on three mechanisms.

**A single LLM call generates the entire session.** Rather than a message-by-message loop with accumulated state — which would need 2 calls per round, exponential retry surface, and a partially-written session on every failure — one call produces the whole transcript as a structured object. This is a deliberate trade of a small amount of token cost for a large reduction in failure modes: no partial sessions, no interleaving bugs, no context drift between rounds. A 3-round session is 6–8 messages and fits comfortably in one response with `thinkingBudget: 0`.

**Every message must cite evidence, and the citation is verified.** The prompt requires each message to reference specific traits from `cited_evidence`. Post-processing checks that each cited trait actually exists in the participants' trait sets. A message whose citations do not resolve is regenerated once; if it still fails, the system falls back to a deterministic template message that cites nothing but stays in character. This preserves the no-fabrication rule at the point where it is most tempting to violate it — an agent being asked to be charming about someone it knows nothing about will otherwise invent shared history, and an invented shared memory in a dating transcript is exactly the kind of thing that makes a demo feel fake.

**Verdicts are evidence-gated.** A verdict of `second_date` requires the two participants to share at least two typed traits and to have no active dealbreaker conflict. This is checked in code against the trait sets, not trusted from the model. The model proposes the verdict and its reasoning; the deterministic gate can only make the result *more* conservative, never less. So the system can never produce a `second_date` between two people whose agents found nothing in common, regardless of what the language model felt like writing.

### 1.2 Requirement Details

- **DAT-301: Deterministic Pair Selection**
- **DAT-302: Single-Call Structured Session Generation**
- **DAT-303: Persona Injection and Voice Differentiation**
- **DAT-304: Evidence Citation Enforcement and Verification**
- **DAT-305: Verdict and Plan Generation with Deterministic Gate**
- **DAT-306: Dealbreaker-Aware Conversation Steering**
- **DAT-307: Deterministic Date Engine Fallback**
- **DAT-308: Session Persistence and Partial Failure Tolerance**

##### 1.2.1 DAT-301: Deterministic Pair Selection

##### Description:
Choose which pairs date, in a way that a reviewer can reproduce exactly.

##### Modes:
| Mode | Selection | Use |
|---|---|---|
| `top_n` (default) | Top N ranked pairs per person from ST-104, de-duplicated unordered | Demonstrates the ranking driving the dating |
| `all` | Every unordered pair | Maximum coverage |
| `explicit` | Only the `pairs` in the request body | Spot-checking a specific pair |

##### De-duplication:
A pair is an unordered set, so `(7, 19)` and `(19, 7)` are one session. Each pair dates at most once per run unless `force` is passed. Selection is sorted by `person_id` to guarantee identical output across runs.

##### Acceptance Criteria:
- **WHEN** `mode = 'top_n'` with `top_n = 5` and 25 people **THEN** each person appears in at most 5 pairs and every pair appears exactly once.
- **WHEN** the same run is executed twice with identical inputs **THEN** the selected pair list is byte-identical.
- **WHEN** `mode = 'explicit'` with `pairs = [[7, 12]]` **THEN** exactly one session is created.
- **WHEN** fewer than two analysed people exist **THEN** the endpoint responds 409 with `error_code = 'INSUFFICIENT_PEOPLE'`.

##### 1.2.2 DAT-302: Single-Call Structured Session Generation

##### Description:
Generate the complete session — all rounds, all messages, the verdict, and the proposed date — in one `generateContent` call whose `responseSchema` describes the whole transcript.

##### Response Shape:
```
{
  "messages": [{ "round_no", "speaker", "content", "cited_traits": [str] }],
  "verdict": "second_date" | "friendzone" | "unmatched",
  "verdict_reasoning": str,
  "proposed_date": str | null,
  "chemistry": float
}
```

##### Configuration:
`responseMimeType = application/json`, `thinkingConfig.thinkingBudget = 0`, `temperature ≈ 0.9` (creativity is wanted here, unlike in analysis), `maxOutputTokens = 4096`.

##### Why one call:
- No partial session on failure — the transaction either writes a complete session or nothing
- No accumulated context drift across rounds
- 25 pairs × 1 call = 25 calls, versus 6–8 calls per session in a message loop
- Deterministic: one input, one output, fully reproducible

##### Acceptance Criteria:
- **WHEN** a session is generated **THEN** the transcript contains at least `2 * rounds` messages, alternating between `initiator` and `responder` (REQ-4.1: 6 messages for the default 3 rounds).
- **WHEN** a session completes **THEN** `rounds_completed` equals the requested rounds.
- **WHEN** the LLM Engine fails at any point **THEN** the remaining rounds are completed by the deterministic dialogue engine, `llm_succeeded` is set to `false`, and `engine` is set to `deterministic` — satisfying REQ-4.6
- **WHEN** the model returns output failing schema validation **THEN** the call is retried once with the validation error as feedback, then falls back to the deterministic engine.
- **WHEN** `temperature` is set **THEN** it is at least 0.8, since deterministic output would defeat the purpose of the stage.

##### Note on "fails mid-session" (REQ-4.6):
REQ-4.6 assumes a message-by-message loop in which a failure can occur partway through. The single-call design makes that state unreachable by construction — there is no partial generation to fail. The requirement's intent is still satisfied, and satisfied more completely: rather than completing only the remaining rounds, the deterministic engine produces the **entire** session, so the transcript is never partial because of an LLM failure. `rounds_completed` on such a session equals the requested round count, and `llm_succeeded = false` records that the fallback was used.

##### 1.2.3 DAT-303: Persona Injection and Voice Differentiation

##### Description:
Both agents' persisted personas are injected into the prompt so they speak consistently with their profile pages, and are instructed not to mirror each other.

##### Prompt Inclusions:
- Initiator: name, tone, summary, guardrails, top 5 traits with types
- Responder: name, tone, summary, guardrails, top 5 traits with types
- Both: the shared traits and any dealbreakers, listed explicitly
- An explicit instruction that the two agents must use **different registers** and must not agree with everything in round one

##### Shared-Context Handling:
Shared traits and dealbreakers are computed in code and passed as structured context, so the model is not left to discover common ground by reading two profile dumps. If zero traits overlap, the prompt says so directly, and the deterministic gate will prevent a `second_date`.

##### Acceptance Criteria:
- **WHEN** a session prompt is built **THEN** it contains both agents' `guardrails` verbatim.
- **WHEN** two agents share three or more traits **THEN** the prompt instructs them to diverge in register rather than mirror each other.
- **WHEN** the participants share no traits **THEN** the prompt states this explicitly and requests an honest, exploratory conversation rather than manufactured common ground.
- **WHEN** two sessions for the same pair are generated with different temperatures **THEN** the transcripts differ in wording while preserving both verdicts' reasoning.

##### 1.2.4 DAT-304: Evidence Citation Enforcement and Verification

##### Description:
Every message references real traits, and the references are checked against the database.

##### Verification:
For each message, every entry in `cited_traits` must match a `label` present in either participant's `profile_traits`. Unresolved citations are stripped and the message is flagged. If a message ends with no valid citations **and** the session was LLM-generated, the session is regenerated once; a second failure routes the whole session to the deterministic engine.

##### Citation Storage:
`date_messages.cited_evidence` stores a JSON array of `{trait_type, label, source}` per message, and this is what the UI renders as the evidence chips under each line of dialogue. **This is what makes the transcript inspectable rather than decorative** — every claim in the conversation can be traced to a row in `profile_traits` and from there to a quote in a public profile.

##### Acceptance Criteria:
- **WHEN** a message cites a trait that exists in a participant's trait set **THEN** the citation is retained in `cited_evidence` with its type and source.
- **WHEN** a message cites a trait that exists in neither participant's set **THEN** the citation is removed before persistence.
- **WHEN** every message in a session has its citations removed **THEN** the session is regenerated once, then routed to the deterministic engine.
- **WHEN** a session is persisted **THEN** every message in it carries at least one valid `cited_evidence` entry (REQ-4.2). This is an invariant, not a hope: LLM sessions that fail the citation check are regenerated, and deterministic sessions are slot-filled from real trait labels so they cite by construction.
- **WHEN** a message makes a factual claim with no resolvable citation **THEN** the claim is treated as a fabrication and triggers the regeneration path.
- **WHEN** a transcript is retrieved via `GET /v1/dates/{id}` **THEN** every message carries a non-empty `cited_evidence` array.

##### 1.2.5 DAT-305: Verdict and Plan Generation with Deterministic Gate

##### Description:
The model proposes a verdict; a deterministic gate constrains what the system will actually record.

##### Gate Rules (applied in code, after generation):
| Gate condition | Effect |
|---|---|
| Shared typed traits < 2 | `second_date` is downgraded to `friendzone` |
| Any active dealbreaker conflict | `second_date` is downgraded to `unmatched` |
| `ingest_state = 'partial'` for either party | `second_date` requires the model to also cite a shared trait |

The gate can only make results more conservative. It can never upgrade a verdict the model proposed, so an over-optimistic model cannot manufacture matches.

##### Verdict Definitions:
| Verdict | Meaning |
|---|---|
| `second_date` | Both agents found real common ground and want to continue |
| `friendzone` | Liked each other, but something does not line up |
| `unmatched` | A dealbreaker or a fundamental mismatch ends it |

##### Plan Generation:
When the verdict is `second_date`, `proposed_date` must be concrete: a named activity drawn from an actual shared hobby, a time, and a location hint drawn from a real shared location. `"Saturday 08:30 — the ridge trail near Sintra, then coffee at a third-wave roastery"` is acceptable; `"let's hang out sometime"` is not. A proposal that references an activity neither participant lists is rejected and regenerated.

##### Acceptance Criteria:
- **WHEN** the model proposes `second_date` for a pair with fewer than two shared traits **THEN** the persisted verdict is `friendzone`.
- **WHEN** the model proposes `second_date` for a pair with an active dealbreaker conflict **THEN** the persisted verdict is `unmatched`.
- **WHEN** the verdict is `second_date` **THEN** `proposed_date` names a specific activity present in a shared hobby and is not a generic placeholder.
- **WHEN** `proposed_date` references an activity in neither participant's traits **THEN** the session is regenerated once.
- **WHEN** the gate modifies a verdict **THEN** `verdict_reasoning` is rewritten to state that the deterministic gate applied and why.

##### 1.2.6 DAT-306: Dealbreaker-Aware Conversation Steering

##### Description:
Dealbreakers must shape the conversation, not merely the score. An agent whose owner has stated a dealbreaker is instructed to surface it early and honestly rather than talking past it.

##### Steering Rules:
- If a dealbreaker is `inferred` rather than quoted, the agent may raise it cautiously and must not state it as fact
- If a dealbreaker is directly contradicted by the other participant's evidence, the conversation is steered toward acknowledging the conflict
- Agents may **not** propose anything that violates a stated dealbreaker, per the persisted guardrails

##### Acceptance Criteria:
- **WHEN** one participant has a `dealbreaker` trait and the other does not **THEN** the conversation addresses the dealbreaker within the session.
- **WHEN** a dealbreaker trait has `provenance_source = 'inferred'` **THEN** the agent raises it hedged, not as a stated fact.
- **WHEN** a proposed date would violate a stated dealbreaker **THEN** it is rejected and the session is regenerated.

##### 1.2.7 DAT-307: Deterministic Date Engine Fallback

##### Description:
When the LLM is unavailable or fails twice, produce a genuine session with no network call — so a quota-exhausted key still yields a full dating demonstration.

##### Mechanism:
Template-based generation with slot-filling from the actual trait sets:
- **Opening** references a shared trait, or honestly notes there is no shared ground yet
- **Middle** rounds probe a specific trait from each side in turn
- **Close** states the verdict, computed by applying the same deterministic gate
- **Verdict** is computed in code from shared-trait count and dealbreaker conflicts
- **`proposed_date`** is assembled from a shared hobby plus a shared location, or `NULL` when there is none

##### Marking:
`date_sessions.engine = 'deterministic'` and `llm_succeeded = false`, so the UI and the write-up can distinguish these sessions from LLM ones. These are not hidden — they are labelled.

##### Acceptance Criteria:
- **WHEN** the LLM is unavailable and two people share three traits **THEN** a complete session is produced with `engine = 'deterministic'`, `llm_succeeded = false`, and a verdict of `second_date`.
- **WHEN** the deterministic engine runs **THEN** it completes in under 100ms with zero network requests.
- **WHEN** the deterministic engine produces a session **THEN** every message references a real trait label, since messages are built by slot-filling from `profile_traits` rather than free-generated.
- **WHEN** the deterministic engine proposes a date **THEN** the activity comes from a shared hobby and the location from a shared location, or `proposed_date` is `NULL`.

##### 1.2.8 DAT-308: Session Persistence and Partial Failure Tolerance

##### Description:
Persist every session, including incomplete ones, and never lose a run to a single bad pair.

##### Behaviour:
- A session is created with `status = 'running'` before generation begins, so a crash leaves an auditable record rather than nothing
- On success: `status = 'complete'`, `rounds_completed`, verdict, and all messages written in one transaction
- On failure: `status = 'failed'`, `error_log` records the cause, and the run continues to the next pair
- The response always reports `sessions_completed` and `sessions_failed`, so partial results render immediately
- Re-running the same pairing updates existing sessions rather than duplicating them

##### Acceptance Criteria:
- **WHEN** a pair's generation raises an exception **THEN** the remaining pairs still run and the response reports `sessions_failed >= 1`.
- **WHEN** a session fails **THEN** a `date_sessions` row exists with `status = 'failed'` and an `error_log` entry naming the cause, and `rounds_completed` holds the number of rounds actually produced before the failure (REQ-4.7). Because a session is generated in a single call, this is 0 for a generation failure and the full count for a fallback-completed session.
- **WHEN** the same pairing is run twice **THEN** `count(date_sessions)` for that pair does not increase.
- **WHEN** a run completes **THEN** `sessions_completed + sessions_failed` equals the number of selected pairs.

### 1.3 Project Artifacts

- `api/openapi.yaml` — `POST /v1/dates/run`, `GET /v1/dates`, `GET /v1/dates/{session_id}`
- `design/er_diagram.mmd` — `date_sessions`, `date_messages`
- `design/dating_sequence.mmd` — session lifecycle and gate logic
- `design/SOLDEF_AGENTD_ST-102_ANALYSIS.md` — the persona, guardrail, and `date_idea` contracts this story consumes
- `design/requirements.md` — REQ-3 dating requirements

### 1.4 Requirement Traceability

| Requirement | Acceptance criterion | Implemented by |
|---|---|---|
| REQ-4 | 1. Two agents alternate across ≥ 3 rounds, ≥ 6 messages | DAT-302 |
| REQ-4 | 2. Every message cites its source trait in `cited_evidence` | DAT-304 |
| REQ-4 | 3. Proposed date derived from **both** people's shared traits | DAT-305, consuming ST-102 `date_idea` |
| REQ-4 | 4. `verdict` ∈ {`second_date`, `friendzone`, `unmatched`} + `verdict_reasoning` | DAT-305 |
| REQ-4 | 5. Dealbreaker conflict reflected in the reasoning | DAT-306, reinforced by the DAT-305 gate |
| REQ-4 | 6. LLM failure → deterministic engine completes, `llm_succeeded = false` | DAT-307 (see note in DAT-302) |
| REQ-4 | 7. Incomplete session still recorded as `failed` with `rounds_completed` | DAT-308 |
| REQ-4 | 8. No agent claims a fact absent from its person's sources | ANA-202 upstream + DAT-304 gate |
| REQ-6 | 2. Live progress through ingestion, analysis, and dating | Background run + `GET /v1/dates` polling |
| REQ-7 | 3. Verbatim quote traceable from message to source | `cited_evidence` → `profile_traits.evidence_quote` |

### 1.4 Dependencies

- Gemini `generateContent` via the LLM Engine from ST-102, including model fallback chain
- `profile_traits` and `agent_personas` from ST-102 — the only inputs
- `match_scores` and `rank_for_a` / `rank_for_b` from ST-104 for `top_n` pair selection
- `pydantic` v2 — `DatingSession`, `DateMessage` response schemas
- SQLAlchemy 2.x async + `asyncpg`

---

## Section 2: Non Functional Requirements

### 2.1 Infrastructure and Deployment

#### 2.1.1 Overview

The dating stage multiplies the cost of the stage before it: ST-102 makes roughly one model call per person, this stage makes one per *pair*. With 25 people that is 25 analysis calls but potentially 300 dating calls if every pair dated. This is the single most important deployment consideration in the project, and it is why `top_n` is the default mode rather than `all`. Defaulting to all-pairs would put the demo over the free-tier quota on its first run and produce a dating screen full of deterministic fallbacks.

The default is therefore `mode = 'top_n', top_n = 5`, which for 25 people yields roughly 60–80 unique pairs after de-duplication — comfortably inside quota, and a better demonstration, because the audience sees the ranking actually drive who dates whom rather than seeing a combinatorial sweep. Concurrency is bounded at 3 sessions in flight, and a bulk run executes as a background task so the HTTP request returns immediately with counts.

The stage is fully functional with the LLM disabled, via the deterministic engine, so a quota failure degrades the transcript quality rather than the feature.

#### 2.1.2 Requirement Details

- **NFR-301: Bounded Session Concurrency and Cost**
- **NFR-302: Model Calls Must Be Bounded per Session**
- **NFR-303: Long-Run Execution Must Not Hold the Request**

##### 2.1.2.1 NFR-301: Bounded Session Concurrency and Cost

##### Description:
Default to `top_n = 5`; cap concurrency at 3 sessions; cap `rounds` at 5; cap `all` mode to 300 pairs.

##### Rationale:
One call per session is the design's main cost control. At 3 concurrent sessions and roughly 2s per call, a 70-pair run finishes in about 50 seconds and uses well under the free-tier daily quota.

##### Acceptance Criteria:
- **WHEN** a run is started without specifying `mode` **THEN** it runs in `top_n` mode with `top_n = 5`.
- **WHEN** a run has 70 pairs **THEN** no more than 3 `generateContent` calls are in flight at any instant.
- **WHEN** `mode = 'all'` with 25 people **THEN** the run is capped at 300 pairs and reports any truncation.

##### 2.1.2.2 NFR-301a: Model Calls Bounded per Session

##### Description:
Maximum 3 calls per session: one initial generation, one corrective retry, one regeneration on citation or plan failure. After that the session routes to the deterministic engine.

##### Acceptance Criteria:
- **WHEN** a session has already made 3 model calls **THEN** any further attempt is refused and the session uses the deterministic engine.
- **WHEN** a run completes **THEN** total model calls are less than 3 × the number of selected pairs.

##### 2.1.2.3 NFR-303: Long-Run Execution Must Not Hold the Request

##### Description:
A bulk run is a background task. The request returns counts immediately; the UI polls `GET /v1/dates` and renders results as they land.

##### Acceptance Criteria:
- **WHEN** `POST /v1/dates/run` is called for 70 pairs **THEN** it returns within 2 seconds without waiting for completion.
- **WHEN** partial results exist **THEN** they are immediately visible via `GET /v1/dates`.

### 2.2 Architecture and System Design

#### 2.2.1 Security and Compliance

- **Guardrails are enforced, not just prompted.** Persona guardrails from ST-102 are injected verbatim into every dating prompt, and the deterministic gate independently prevents fabricated `second_date` verdicts. Prompt-level instruction is treated as advisory; the gate is the actual control.
- **No fabrication of the other person.** Citation verification (DAT-304) removes any claim not backed by a real trait. An agent that knows nothing about its counterpart says so.
- **The agents are explicitly fictional.** A header on the dating screen and in every API response states that the participants are AI agents representing public profile data, so no transcript can be mistaken for a real person's words.
- **No private data.** Only traits derived from public endpoints are used. No message content derived from a non-public source can enter a prompt.
- **Truncation of retrieved text persists.** The 12 000-character bound from ST-102 continues to apply to everything the dating prompt sees.

#### 2.2.2 System Performance

| Metric | Target | Basis |
|---|---|---|
| Single session generation (LLM) | 2–5s | One call, `thinkingBudget: 0` |
| Single session (deterministic) | < 100ms | Template + trait lookups |
| 70-pair run, concurrency 3 | 50–90s | Bounded by LLM latency |
| Citation verification | < 5ms per session | Set membership over trait labels |
| `GET /v1/dates/{id}` with 8 messages | < 60ms | Two indexed queries |
| `GET /v1/dates` for 300 sessions | < 150ms | Single paginated query |

#### 2.2.3 Availability and Reliability

- One call per session means no partial state within a session — the failure surface is a single atomic write
- `status = 'running'` is written before generation, so a crash is visible rather than silent
- One pair's failure never aborts the run; the response reports both counts
- The deterministic engine is a total fallback: quota exhausted, network down, or schema permanently broken all still yield a full transcript
- Re-running a pairing is idempotent, so a failed session can be retried without duplication
- The deterministic gate cannot be bypassed by any prompt, because it runs after generation in code

#### 2.2.4 Cost Efficiency

The dominant cost lever is pair count, so `top_n` is the default and the concurrency cap is 3. Beyond that:

- One call per session instead of a per-message loop — roughly a 6–8× reduction in calls
- `maxOutputTokens = 4096` and `thinkingBudget = 0` prevent a thinking model from burning the budget before emitting dialogue
- Traits are truncated to the top 5 per participant in the prompt, so input tokens stay bounded regardless of how many traits a person has
- The deterministic engine costs nothing and is used whenever the LLM path is not worth the call

#### 2.2.5 Traceability and Observability

- `date_sessions.engine` (`llm` / `deterministic`) and `llm_succeeded` make the quality of every transcript inspectable
- `cited_evidence` on each message provides the full chain: message → trait → evidence quote → source payload → strategy
- `verdict_reasoning` records the model's stated reason, and is rewritten when the deterministic gate overrides
- `round_chemistry` per message gives a quantitative signal for the UI's chemistry display
- `model_usage` accumulates token counts for the dating stage, making the cost of the demonstration measurable rather than assumed
- `error_log` captures every failed session with its cause and the call attempt count

---

## Section 3: In Scope and Out Scope

### 3.1 In Scope Details

- Deterministic pair selection in `top_n`, `all`, and `explicit` modes
- Unordered-pair de-duplication and idempotent re-runs
- Single-call structured session generation with schema constraint
- Persona, tone, guardrail, and shared-context injection
- Evidence citation on every message, with verification against real traits
- Verdict generation with a deterministic gate that can only reduce optimism
- Concrete plan generation grounded in real shared hobbies and locations
- Dealbreaker-aware conversation steering
- Full deterministic dating engine fallback
- Session persistence with `running` / `complete` / `failed` lifecycle
- Partial-failure tolerance across a run

### 3.2 Out Scope Details

- Real-time or streaming conversation — sessions are generated and stored atomically, not streamed turn by turn
- Humans participating in the conversation, or any human-in-the-loop review step
- The agents forming their own preferences, rejecting people, or making decisions outside a pairing
- Voice, video, or avatar generation
- Multi-party dates, group scenarios, or dates with more than two agents
- Free-form date scheduling with calendars, availability checking, or booking
- Persuading a user to like someone — the system reports findings, it does not advise
- Learning agent preferences across sessions, or any cross-session memory
- Any claim, message, or proposal not backed by a verified trait

---

## Section 4: Solution Diagrams

#### 4.1 UI/UX Design Diagram

The dating screen renders as a chat transcript: alternating speaker cards with distinct agent colours by tone, an evidence chip under every message, a per-round chemistry indicator, and a verdict banner with the reasoning and the proposed date. Sessions generated by the deterministic engine carry a visible `deterministic` badge so the fallback is never presented as an LLM result.

**Diagram Location:** `design/dating_sequence.mmd`

#### 4.2 Architecture Design Diagram

**Diagram Location:** `design/er_diagram.mmd` (entities: `date_sessions`, `date_messages`, and the `profile_traits` / `agent_personas` joins)

#### 4.3 Infrastructure Design Diagram

The FastAPI process hosts the Dating Engine, the deterministic fallback, and the verdict gate. Inputs: `profile_traits`, `agent_personas`, `match_scores`. Outbound: Gemini `generateContent`, bounded to 3 concurrent calls. Downstream: `date_sessions`, `date_messages`.

**Diagram Location:** `api/openapi.yaml` (`/v1/dates/run`, `/v1/dates`, `/v1/dates/{session_id}`)
