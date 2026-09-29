# Agentic Dating

An agentic dating site. Every person is represented by an agent. That agent dates on the
person's behalf. The agents date each other.

- Each person has exactly two sources: their public LinkedIn profile and their public Instagram profile.
- The agent reads both sources, analyses the person, and publishes a profile page.
- The agents then date each other — multi-round conversations with evidence cited.
- Every person gets a ranking of who fits them best.

**Status:** design complete, implementation in progress. Development happens on the `feat-date` branch.

## Pipeline

| Stage | Story | What it does | Solution design |
|---|---|---|---|
| 1 | ST-101 | Ingestion — fetch both public profiles through an ordered strategy chain | [`design/SOLDEF_AGENTD_ST-101_INGESTION.md`](design/SOLDEF-AGENTD_ST-101_INGESTION.md) |
| 2 | ST-102 | Analysis — typed, evidence-backed traits and a speaking agent per person | [`design/SOLDEF_AGENTD_ST-102_ANALYSIS.md`](design/SOLDEF_AGENTD_ST-102_ANALYSIS.md) |
| 3 | ST-103 | Dating — agents hold multi-round conversations and reach a verdict | [`design/SOLDEF_AGENTD_ST-103_AGENT_DATING.md`](design/SOLDEF_AGENTD_ST-103_AGENT_DATING.md) |
| 4 | ST-104 | Ranking — deterministic eight-dimension compatibility scoring | [`design/SOLDEF_AGENTD_ST-104_RANKING.md`](design/SOLDEF_AGENTD_ST-104_RANKING.md) |

## Design pack

| Artifact | Contents |
|---|---|
| [`design/requirements.md`](design/requirements.md) | Seven requirement groups with EARS-style acceptance criteria |
| [`design/er_diagram.mmd`](design/er_diagram.mmd) | Ten entities: people, sources, traits, personas, scores, dates, messages, usage, errors |
| [`api/openapi.yaml`](api/openapi.yaml) | Internal API contract |
| [`api/external-api.yaml`](api/external-api.yaml) | Every upstream call, with the behaviour observed for each |
| [`design/ingestion_pipeline.mmd`](design/ingestion_pipeline.mmd) | ST-101 strategy chain resolution |
| [`design/analysis_flow.mmd`](design/analysis_flow.mmd) | ST-102 LLM path, Taxonomy fallback, evidence validation |
| [`design/dating_sequence.mmd`](design/dating_sequence.mmd) | ST-103 session lifecycle and verdict gate |
| [`design/ranking_pipeline.mmd`](design/ranking_pipeline.mmd) | ST-104 dimension computation |

## Three design commitments

**Failure is a recorded state, not an exception.** Both platforms block naive scrapers —
LinkedIn with HTTP 999, Instagram with a login wall. Ingestion walks an ordered strategy
chain per source and records which strategy won, how long it took, and why each attempt
failed. A person whose LinkedIn is blocked but whose Instagram resolved is `partial`, not
`failed`, and the pipeline continues on one source.

**No claim without a quote.** Every extracted trait carries the verbatim text it came
from. A trait whose quote cannot be found in the source text is demoted or dropped. Every
line of dialogue cites the trait it was built from. If a model invents something, the
system notices and refuses to store it.

**The LLM is an enhancement, not a dependency.** Every stage has a deterministic path that
makes no model call, so the whole pipeline completes with no API key. On the free tier,
`gemini-3.1-flash-lite` is the only model verified working; `gemini-2.5-*` is retired and
the Pro tier is quota-blocked, so the client holds an ordered fallback chain and pins to
whatever responds. Transcripts are labelled `llm` or `deterministic` — the fallback is
shown, never disguised.

## Quick start

```bash
make setup
make run
```

## Configuration

Copy `.env.sample` to `.env`. The only required secret is `LLM_API_KEY`; without it the
Taxonomy Engine and the deterministic dating engine take over and everything still runs.
