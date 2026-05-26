# AI Co-Scientist

A multi-agent system for tournament-style scientific hypothesis generation, ranking, and synthesis. Built on the architecture described in [`reference/`](reference/) (Google's Co-Scientist), implemented in Python on top of the raw Anthropic SDK.

The system takes a natural-language research goal, runs six specialized LLM agents in a coordinated loop, and produces a *Research Overview* of the top-ranked hypotheses:

- **Generation** — proposes hypotheses via literature review and simulated scientific debate
- **Reflection** — reviews hypotheses for novelty, correctness, and testability; deep-verifies assumptions
- **Ranking** — runs an Elo tournament with simulated debates between hypotheses
- **Evolution** — combines, simplifies, and reimagines top-ranked hypotheses
- **Proximity** — embeds and clusters hypotheses to drive dedup and informative pairings
- **Meta-review** — synthesizes system-wide feedback and the final research overview

A **Supervisor** schedules agents via a durable task queue (SQLite-backed) with bounded concurrency. The full design is in [`/Users/kuan-linhuang/.claude/plans/based-on-these-txt-unified-pearl.md`](../../.claude/plans/based-on-these-txt-unified-pearl.md).

## Architecture

```
                       co-scientist run "<goal>"
                                  │
                                  ▼
            ┌──────────────────────────────────────┐
            │            Supervisor                │  durable task queue (SQLite)
            │  • parse_goal → ResearchPlan         │  bounded concurrency
            │  • enqueue initial Generation tasks  │  lease + dead-letter + resume
            │  • main loop: claim → run → follow-up│  termination: BUDGET / WALL_CLOCK
            │  • _decide_next_steps when idle      │              / ELO_STABLE / IDLE / EXTERNAL
            │  • _finalize: meta-review final      │
            └──────────────────────────────────────┘
                                  │  tasks
            ┌─────────────────────┼─────────────────────────────┐
            ▼                     ▼                             ▼
   ┌──────────────┐      ┌──────────────┐              ┌──────────────┐
   │  Generation  │      │  Reflection  │              │   Ranking    │
   │  literature  │      │  full / verif│              │ pairwise vs  │
   │  +tool loop  │─►hyp│  +URL check  │─►review─►rank│   debate     │──►Elo
   └──────────────┘      └──────────────┘              └──────────────┘
            ▲                                                   │
            │                                                   ▼
   ┌──────────────┐      ┌──────────────┐              ┌──────────────┐
   │  Evolution   │◄─────│ Meta-review  │              │  Proximity   │
   │ combine /    │ feed │ system fdbk  │              │ FAISS recluster│
   │ simplify /   │ back │ final overview│             │ dedup + close│
   │ out_of_box   │      └──────────────┘              │ Elo pairings │
   └──────────────┘                                    └──────────────┘
            │
            ▼
       new hypotheses re-enter the cycle


  Shared infrastructure
  ─────────────────────
  • LLMProvider  ─ anthropic / openai / openrouter / gemini / groq /
                   together / mistral / ollama / openai_compatible
  • ToolRegistry ─ web_fetch + pubmed/arxiv/europe_pmc;
                   web_search auto-registered iff TAVILY/BRAVE key set;
                   science-skills via SKILL.md frontmatter
  • TokenBudget  ─ per-agent shares + global cap; reservation released on retry
  • EventBus     ─ in-memory fan-out to SSE for the live web UI
  • FaissStore   ─ IndexFlatIP, asyncio-locked, atomic save/load;
                   Voyage → OpenAI → hash-fallback embedder chain
  • SQLite       ─ 15 tables incl. sessions / hypotheses / reviews / tasks /
                   tournament_matches / transcripts / events / bench_*
                   (WAL, busy_timeout, schema_migrations idempotent runner)
```

## Status

**Through M9 — full system shipped. Multi-provider + bench landed.** 182 unit tests passing, ruff clean.

- **M0 — Skeleton.** Package layout, pydantic-settings config, SQLite schema + migrations (12 tables incl. `spans`/`events`/`elo_journal`), ULID + deterministic-hash IDs, structlog JSONL logging.
- **M1 — Storage, vectors, tools.** 10 repos; Voyage+OpenAI embedders; FAISS `IndexFlatIP` per-session store; built-in tools (`web_search`, `web_fetch`, `pubmed_search`, `arxiv_search`, `europe_pmc_search`); science-skills bridge that parses `SKILL.md` + shells out to scripts with a path-traversal guard.
- **M2 — Anthropic SDK layer.** 14 prompt templates; Jinja2 loader; retry honoring Retry-After for 429/529; `TokenBudget` with per-agent shares; model routing with never-degrade list; `AnthropicClient` with 4-tier `cache_control`, retry, transcript persistence, USD accounting; tool-loop driver that preserves thinking-block signatures and tracks URLs for citation honesty; `UNTRUSTED_SOURCE` quoting for prompt-injection defense.
- **M3 — Generation + Reflection.** `BaseAgent`; literature-strategy `GenerationAgent` with `record_hypothesis` tool, dedup via FAISS, hallucinated-URL filter; full-mode `ReflectionAgent` with `record_review` + URL filter.
- **M4 — Ranking + Elo tournament.** `AddToTournament` + `RunTournamentBatch` with pair selection weighted by `exp(-Δelo/200) · (1 - cosine_sim)`, debate-vs-pairwise mode switching, anchor-cached debates, idempotent `elo_journal` updates.
- **M5 — Supervisor scheduling.** Durable resume with lease reclaim + max-attempts dead-letter; hybrid termination (BUDGET / WALL_CLOCK / ELO_STABLE / EXTERNAL); `StabilityTracker` with snapshot history; `decide_next_steps` for idle refinement; pause/resume/abort via DB-flagged session.status. In-memory `EventBus` shared with the web UI.
- **M6 — Evolution + Proximity + Meta-review.** Evolution strategies (combine on most-distant top pair, simplify, feasibility, out_of_box) with parent_ids; Proximity batch recluster with sklearn agglomerative; periodic Meta-review system feedback (auto-injected into future Generation/Evolution prompts); final research overview synthesis.
- **M7 — Web UI.** FastAPI + Jinja2 + htmx + Pico.css + SSE. Pages: sessions index, new session form, session dashboard (live leaderboard, match feed, budget gauges), hypothesis detail with reviews, final overview. API endpoints for pause/resume/abort/feedback. `co-scientist serve` boots both the UI and a Supervisor in one process.
- **M8 — Safety + observability + evals.** Haiku-backed safety classifier with allow/warn/quarantine/block actions; citation verifier (fetch URL, check excerpt-substring); read-only `obs/metrics` (tokens, cost, cache hit ratio, P50/P95 latency, dead tasks) backing `/api/sessions/{id}/metrics`; LLM-as-judge rubric runner with bundled fixtures; `co-scientist eval [agent] [--offline]`.
- **M9 — Batch API + estimator + resume hardening.** `BatchPool` for sub-decile tournament matches (50% cheaper Batch API submission with safe requeue on failure); pre-flight `estimator` that warns when projected USD spend > 1.2x budget; `co-scientist estimate` subcommand.

**No network calls in CI** — Anthropic / embeddings / web tools are mocked or stubbed; live smoke tests are manual.

## Install

```bash
# Recommended: Python 3.11–3.13 (FAISS wheel availability)
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

cp .env.example .env
# fill in ANTHROPIC_API_KEY (default provider) or OPENAI_API_KEY,
# depending on which LLM vendor you select below.
```

## LLM provider

The agents talk to one LLM provider per session, configured in
[`config/default.toml`](config/default.toml) (override with your own
`co-scientist.toml`):

```toml
[llm]
provider = "anthropic"   # anthropic | openai | openai_compatible
```

| provider              | Endpoint                                                | Required key            | Example models                                            |
| --------------------- | ------------------------------------------------------- | ----------------------- | --------------------------------------------------------- |
| `anthropic` *(default)* | api.anthropic.com                                     | `ANTHROPIC_API_KEY`     | `claude-opus-4-7`, `claude-sonnet-4-6`                    |
| `openai`              | api.openai.com                                          | `OPENAI_API_KEY`        | `gpt-5`, `gpt-4o`, `o3-mini`                              |
| `openrouter`          | openrouter.ai — 200+ models from every major vendor     | `OPENROUTER_API_KEY`    | `anthropic/claude-3.5-sonnet`, `openai/gpt-5`, `google/gemini-2.5-pro`, `meta-llama/llama-3.3-70b-instruct` |
| `gemini` / `google`   | generativelanguage.googleapis.com (OpenAI-compat)       | `GEMINI_API_KEY`        | `gemini-2.5-pro`, `gemini-2.5-flash`                      |
| `groq`                | api.groq.com                                            | `GROQ_API_KEY`          | `llama-3.3-70b-versatile`, `mixtral-8x7b-32768`           |
| `together`            | api.together.xyz                                        | `TOGETHER_API_KEY`      | `meta-llama/Llama-3.3-70B-Instruct-Turbo`                 |
| `mistral`             | api.mistral.ai                                          | `MISTRAL_API_KEY`       | `mistral-large-latest`, `codestral-latest`                |
| `ollama`              | localhost:11434 — local models                          | *(none)*                | `llama3.3:70b`, `qwen2.5:32b`                             |
| `openai_compatible`   | Anything else; set `[llm.openai] base_url` explicitly   | `OPENAI_API_KEY` (or any non-empty string for keyless servers) | depends |

OpenRouter exposes one API for every vendor:

```toml
[llm]
provider = "openrouter"
[llm.openrouter]
referer = "https://your-app.example.com"   # optional, for catalog attribution
title   = "My Co-Scientist"

[models]
generation         = "anthropic/claude-3.5-sonnet"
reflection         = "openai/gpt-5"
ranking_pairwise   = "google/gemini-2.5-flash"
metareview_final   = "anthropic/claude-opus-4-7"
```

Gemini directly:

```toml
[llm]
provider = "gemini"

[models]
generation       = "gemini-2.5-pro"
reflection       = "gemini-2.5-pro"
ranking_pairwise = "gemini-2.5-flash"
metareview_final = "gemini-2.5-pro"
```

Mixing vendors per session requires picking the provider once; for
multi-vendor routing in a single session, use `provider = "openrouter"`
and let OpenRouter dispatch to the upstream API per model.

Cost is estimated via `co_scientist/llm/routing.py`'s `PRICE_TABLE`;
unknown models fall back to a sonnet-class default — edit the table or
set tighter `[run] budget_usd` if running on a new model.

**Provider feature support:**

| Feature              | anthropic | openai (o-series) | openai (gpt) | openai_compatible |
| -------------------- | --------- | ----------------- | ------------ | ----------------- |
| Tool / function call | ✅        | ✅                | ✅           | depends on endpoint |
| Extended reasoning   | ✅ (thinking) | ✅ (`reasoning_effort`) | ❌ (dropped) | endpoint-specific |
| Prompt-cache breakpoints | ✅    | ❌                | ❌           | ❌                |
| Batch API (50%-off ranking) | ✅ | ❌            | ❌           | ❌                |

## Initialize

```bash
co-scientist init
co-scientist list
```

`init` creates `data/` (artifacts, vectors, logs) and applies migrations to `data/co_scientist.db`.

## Configuration

Layered: [`config/default.toml`](config/default.toml) → `~/.co-scientist/config.toml` → `./co-scientist.toml` → `--config <path>`. Secrets come from environment only (see [`.env.example`](.env.example)).

## Bench: compare models head-to-head

`co-scientist bench` runs the same goal under N different `(provider, model)`
configurations and ranks them via a single shared Elo tournament. Each
candidate independently generates hypotheses; then every candidate-pair
plays `--matches` head-to-head debates, judged by ONE fixed judge model
(picked separately so no candidate scores its own work).

### Presets at a glance

| `--preset`               | What it does |
| ---                      | --- |
| `paper`                  | Co-Scientist paper baselines (plus Haiku) via OpenRouter, head-to-head Elo only |
| `paper-aml`              | Same candidates + the paper's AML drug-repurposing goal + 5-drug gold-set recall (Binimetinib, Pacritinib, Cerivastatin, Pravastatin, DMF) |
| `paper-aml-vs-raw`       | `paper-aml` but each model runs **both** in the full pipeline AND as a single raw LM call — isolates the multi-agent harness's value-add |
| `frontier-aml-vs-raw`    | Same pipeline-vs-raw setup but with current frontier models (Claude Opus 4.7, GPT-5, Gemini 3 Pro / Flash) |

### Pipeline vs raw LM (one model, isolated)

The `--preset *-vs-raw` presets pit each model's **full co-scientist Generation
pipeline** (literature tools + tool loop + dedup + `record_hypothesis`)
against a **single raw LM call** with the same model + a forced
`record_hypothesis` function call (no tools). Lets you measure how much
of the system's output quality comes from the multi-agent harness vs the
underlying model.

Live 2-model vs-raw run on the AML goal (12 matches, judged by
`google/gemini-3-flash-preview`, $0.15 total):

```
                                          W-L   mean Elo  $ spent   latency
flash3   [pipeline]  google/gemini-3-...  4-2    1234     $0.0275   14.9s
flash3   [raw]       google/gemini-3-...  5-1    1228     $0.0033    5.0s
gpt4o    [pipeline]  openai/gpt-4o        3-3    1201     $0.1040   18.5s
gpt4o    [raw]       openai/gpt-4o        0-6    1154     $0.0105    3.7s
```

Read: for `gemini-3-flash-preview`, the multi-agent harness is **within
noise** of the raw model (Elo 1234 vs 1228) but costs 8× more and is
3× slower. For `gpt-4o`, the harness **wins decisively** (Elo 1201 vs
1154, win-rate 3-3 vs 0-6). The value-add of the agent stack depends on
the base model — strong reasoning-tuned models often need it less.

### Quick start: `--preset paper`

Reproduce the Co-Scientist paper's preference-ranking baselines
(plus Haiku) in one command. OpenRouter retired the experimental Gemini
2.0 models the paper used; the preset substitutes the closest current
analogues and documents the swap in `co_scientist/bench/presets.py`.

```bash
co-scientist bench "Identify hypotheses about microbiome-driven inflammation" \
  --preset paper \
  --budget-per-candidate 1.5 --judge-budget 1.0
```

Live run on this goal — 12 matches, **$0.40 total**, judged by
`google/gemini-3-flash-preview` (the preset's suggested judge):

```
Bench bnc_01KSG7HM47116412H3NV3VKDF8 — 12 matches
┏━━━━━━┳────────────────────────┬─────────────────────────────────┬─────┬──────────┬────────┓
┃ rank ┃ label                  ┆ provider:model                  ┆ W-L ┆ mean Elo ┆ $spent ┃
┡━━━━━━╇────────────────────────┼─────────────────────────────────┼─────┼──────────┼────────┩
│  1   │ gemini-2-flash-thinking│ openrouter:google/gemini-2.0-fl…│ 6-0 │   1284   │ 0.0026 │
│  2   │ gemini-2-pro           │ openrouter:google/gemini-2.5-pro│ 4-2 │   1229   │ 0.0342 │
│  3   │ openai-o1              │ openrouter:openai/o1            │ 2-4 │   1165   │ 0.2326 │
│  4   │ claude-haiku-4.5       │ openrouter:anthropic/claude-ha…│ 0-6 │   1123   │ 0.1338 │
└──────┴────────────────────────┴─────────────────────────────────┴─────┴──────────┴────────┘
```

Note: when judge and a candidate share the same model family there is a
documented echo-bias. The judge-side default
(`google/gemini-3-flash-preview`) is configurable via `--judge`; consider
running with a different judge family if you want to control for that.

### AML drug-repurposing benchmark (gold-set scoring)

`co-scientist bench --preset paper-aml` runs every candidate against the
paper's AML drug-repurposing goal and scores **recall** against a curated
answer key from the Co-Scientist paper. The paper actually reports
**two** AML repurposing results, and we keep both gold sets so you can
score against either; past bench artifacts record which set they used
in `bench_runs.goldset_label`.

| label                              | size | what it is |
| ---                                | --- | --- |
| `aml-repurposing-paper-top3` (default for `paper-aml*`) | 3 | The top-3 of the paper's *ranked* list under the **strict methodology**: candidates with no prior published AML repurposing, no prior preclinical evidence in AML, and no external inputs (no DepMap dependency scores, no expert curation). → **Nanvuranlat (JPH-203 / KYT-0353), KIRA6, Leflunomide (Arava / HWA-486 / Teriflunomide / Aubagio)** |
| `aml-repurposing-paper-5`          | 5 | The broader 5-drug list referenced in the paper's main text. Includes well-known candidates, some with prior preclinical AML evidence. → **Binimetinib (MEK162 / Mektovi), Pacritinib (SB1518 / Vonjo), Cerivastatin (Baycol), Pravastatin (Pravachol), Dimethyl fumarate (DMF / BG-12 / Tecfidera)** |

The `paper-aml` preset uses the strict top-3 by default. Swap with `--goldset`:

```bash
# Strict top-3 methodology (default)
co-scientist bench --preset paper-aml --n 5 --matches 2

# Broader 5-drug list (the earlier vintage)
co-scientist bench --preset paper-aml --goldset aml-repurposing-paper-5 \
  --n 5 --matches 2

# Disable gold-set scoring entirely for a paper-aml run
co-scientist bench --preset paper-aml --goldset none
```

The matcher is whole-token, case-insensitive, and looks at every searched
field of every hypothesis (title / summary / full_text / `entities` /
citation excerpts). Drug **class** mentions (e.g. "DHODH inhibitor")
do **not** count — the candidate has to name the actual compound
(or one of its registered aliases).

The strict top-3 goal asks for:
> *"a ranked list of drug repurposing candidates for AML; each candidate must have no prior published AML repurposing, no prior preclinical evidence in AML; use only internal knowledge — no DepMap scores, no expert curation."*

This makes the bench harder than naming any FDA-approved drug — the model
has to **not** rely on already-known AML literature.

#### Where past results live

```bash
# Inspect every AML bench you've run, with the gold set it scored against:
sqlite3 data/co_scientist.db <<'SQL'
SELECT id, created_at, goldset_label,
       (SELECT COUNT(*) FROM bench_candidates WHERE bench_id=br.id) AS n_cand
  FROM bench_runs br
 WHERE research_goal LIKE '%AML%' OR research_goal LIKE '%leukemia%'
 ORDER BY created_at DESC;
SQL
```

Per-candidate hits + per-entity provenance (which alias matched, which
hypothesis, which field) live in `bench_candidates.gold_hit_names` and
the bench JSON artifact under
`data/artifacts/<session>/bench/<bench_id>.json`.

Note: with small `--n`, recall against either gold set is sensitive to
luck — the paper surfaced these after running 15 expert-curated goals.
For comparable numbers raise `--n` to 5+ and run multiple seeds.

### Custom candidates

`label=provider:model[@mode]`. `mode` is `pipeline` (default) or
`direct`. Pipeline goes through the full Generation agent stack;
direct is a single forced-tool LM call with no literature tools.

```bash
co-scientist bench "Identify hypotheses about X" \
  -c flash3=openrouter:google/gemini-3-flash-preview \
  -c flash3-raw=openrouter:google/gemini-3-flash-preview@direct \
  -c gpt5=openai:gpt-5 \
  -c opus=anthropic:claude-opus-4.7 \
  --judge anthropic:claude-sonnet-4-6
```

### Mechanics

- **Generation runs in parallel** per candidate under a deep-copied
  Config (`cfg.llm.provider`, `cfg.models.*`, thinking budgets zeroed
  for non-Anthropic).
- **Round-robin pairings**: every pair plays `--matches` head-to-heads
  (one random hypothesis from each side per match).
- **Structured verdict** via a forced `record_verdict` function call —
  no fragile `better idea: <N>` text parsing across providers.
- **Per-candidate stats** persisted to `bench_candidates`: hyp count,
  W-L, mean / top Elo, $ spent, p50 latency, last error.
- **Each match persisted** to `bench_matches` with both sides'
  hypothesis text, pre/post Elo, judge rationale, cost & latency.
- Bench runs are **isolated from regular sessions** — they don't write
  to `tournament_matches` or affect any session's leaderboard.

## Repository layout

```
co_scientist/
  agents/       # supervisor + 6 specialized agents (M3+)
  bench/        # cross-model bench runner (compare via Elo tournament)
  llm/          # provider abstraction (anthropic/openai/openrouter/gemini/...),
                # tool loop, budgets, routing, retry
  storage/      # SQLite schema + migrations, db connection, 15 repos
  tools/        # tool registry; web/search, science-skills, code exec
  vectors/      # embeddings (Voyage/OpenAI/hash-fallback) + FAISS index
  orchestrator/ # task queue, worker pool, termination, event bus
  safety/       # injection quoting, classifier, citation verifier
  obs/          # spans, metrics
  web/          # FastAPI + htmx + SSE UI + sanitized markdown renderer
  evals/        # per-agent + e2e + regression evals
  tests/        # 180+ unit tests + fixtures + smoke
config/
  default.toml
  prompts/      # Jinja2 templates per agent.mode
reference/      # input materials (pseudocode, prompts, diagrams)
data/           # gitignored; runtime artifacts
vendor/         # gitignored; pinned clone of google-deepmind/science-skills
```

## License

Apache-2.0.
