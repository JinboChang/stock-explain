# Stock Move Explainer (US)
Evidence-backed daily move attribution with precomputed RAG summaries.

Hover a daily candle and see:
- What happened (return, volume anomaly, gap, intraday range)
- Likely catalysts (top news and SEC filings with links and scores)
- AI synthesis (short, evidence-grounded summary)
- Or an explicit "No clear textual catalyst" when confidence is low

This is not a prediction or trading signal system. It is an evidence-backed
attribution tool that distinguishes event-driven moves from diffuse market moves.

Scope
- US only
- ~200 tickers
- Candidate window: D-1 to D+1 (ET)
- P95 latency target: < 300ms for hover responses

Why this is a DE / MLE / SWE project

Data Engineering
- Daily ingestion with watermark + safety lag
- Bronze/Silver/Gold lakehouse design
- Normalization, dedup, and entity linking (ticker mapping)
- Session-aware time normalization (UTC/ET, PRE/REG/POST buckets)
- Daily context features + candidate builder (D-1 to D+1 window)
- Backfills and idempotent reprocessing
- Partitioning, schema versioning, and late-arrival reprocessing policy
- Data quality checks and observability

Machine Learning Engineering
- Hybrid retrieval (BM25 + embeddings)
- Candidate scoring and explainability gate (HIGH/MED/NONE)
- Evidence-only RAG summaries with safety constraints
- Evaluation sets and metrics (Top-3 hit, explainability precision)

Software Engineering
- FastAPI serving layer with Redis cache
- Postgres gold tables for stable artifacts
- Frontend chart hover UX
- Performance and deployment (P95 < 300ms)

Architecture (end-to-end)

    +-----------------------+
    | External Data Sources |
    | - Prices (OHLCV)      |
    | - News API            |
    | - SEC filings         |
    +-----------+-----------+
                |
                v
    +-------------------------------+
    | Ingestion (Dagster/Airflow)   |
    | - watermark + safety lag      |
    | - idempotent writes           |
    +---------------+---------------+
                    |
                    v
    +-------------------------------+
    | Lakehouse (Bronze/Silver)     |
    | - Bronze: raw immutable       |
    | - Silver: normalized ET time  |
    | - Dedup + entity linking      |
    +---------------+---------------+
                    |
        +-----------+-----------+
        |                       |
        v                       v
    +-------------------+   +------------------------+
    | Search Index      |   | Gold Tables (Postgres) |
    | - BM25            |   | - daily_context        |
    | - Vector index    |   | - doc_candidates       |
    +---------+---------+   | - daily_explanations   |
              |             +-----------+------------+
              |                         |
              +------------+------------+
                           v
    +-----------------------------------+
    | Ranking + Explainability Gate     |
    | - score candidates                |
    | - gate: HIGH/MED/NONE             |
    | - RAG summary if not NONE         |
    +----------------+------------------+
                     |
                     v
    +-----------------------------------+
    | Serving (FastAPI + Redis)         |
    | - /explain, /timeline             |
    | - cache warm (last 30 days)       |
    +----------------+------------------+
                     |
                     v
    +-----------------------------------+
    | Frontend (Next.js)                |
    | - Candlestick chart               |
    | - Hover -> evidence card          |
    +-----------------------------------+

Feature and candidate builder (offline)
- Build gold_daily_context per (ticker, session_date)
- Generate doc candidates from linked news and SEC in D-1 to D+1 (ET)
- Outputs: gold_daily_context, gold_doc_candidates

Data model (Gold)
- gold_daily_context
  - ret_1d, volume_z, gap_pct, intraday_range
  - optional: market_ret_1d, sector_ret_1d
- gold_doc_candidates
  - (ticker, date, doc) features: rel_bm25, rel_embed, time_decay,
    src_weight, entity_conf, dup_penalty, final_score, rank
- gold_daily_explanations
  - explainability: HIGH | MEDIUM | NONE
  - conf_score
  - top_doc_refs (JSON)
  - ai_summary (nullable if NONE)
  - model_version, prompt_version, run_id

Explainability gate (anti-hallucination)

final_score =
  (0.35 * rel_embed +
   0.25 * rel_bm25_norm +
   0.20 * time_decay +
   0.10 * src_weight +
   0.10 * entity_conf) * dup_penalty

conf_score = avg(top3 final_score)

Thresholds (initial)
- HIGH: conf_score >= 0.70
- MEDIUM: 0.55 <= conf_score < 0.70
- NONE: conf_score < 0.55 -> show "No clear textual catalyst"

Serving API (online)
- GET /explain?ticker=TSLA&date=2025-11-07
  - returns context metrics, explainability status, top doc refs, summary
- GET /timeline?ticker=TSLA&from=2025-10-01&to=2025-12-31
  - returns OHLCV + computed features for chart rendering

Data quality and observability
- Freshness: expected partitions arrived by cutoff
- Completeness: ticker coverage >= 98%
- Validity: OHLC constraints (low <= open/close <= high), volume >= 0
- Drift: doc counts, explainability-rate shifts

Evaluation
- Explainability precision (HIGH/MED)
- Top-3 plausible hit rate
- Coverage (fraction of days with HIGH/MED)

Safety and disclaimer
- No trading recommendations
- No causal certainty claims
- Summaries grounded in retrieved evidence
- Explicit NONE when evidence is insufficient

Roadmap
- Add sector and market baselines (SPY + sector ETFs)
- Improve entity linking and disambiguation
- Learning-to-rank for candidate scoring
- Better dedup (cluster-level canonical selection)
- Cloud deployment (ECS + RDS + OpenSearch managed)
