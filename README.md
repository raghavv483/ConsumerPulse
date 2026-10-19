# ConsumerPulse
# ConsumerPulse

**A Voice-of-Consumer intelligence engine for D2C beauty and personal care.**

Turns raw marketplace reviews into answers a brand manager can act on — which SKUs are
slipping, what consumers are actually complaining about, and which product issues R&D
should fix first — and exposes the whole thing over **MCP** so you can just ask in
natural language and get back cited, source-linked answers.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![BigQuery](https://img.shields.io/badge/Google_BigQuery-669DF6?logo=googlebigquery&logoColor=white)
![Pandas](https://img.shields.io/badge/pandas-150458?logo=pandas&logoColor=white)
![ChromaDB](https://img.shields.io/badge/ChromaDB-FF6B6B)
![MCP](https://img.shields.io/badge/MCP-Model_Context_Protocol-000000)
![Streamlit](https://img.shields.io/badge/Streamlit-FF4B4B?logo=streamlit&logoColor=white)

---

## Build status

This repo is built in three independent phases. Tick these off as they land.

- [ ] **Phase 1 — Analytics spine.** pandas cleaning → BigQuery → SQL transforms → Streamlit dashboard
- [ ] **Phase 2 — AI layer.** LLM review classification → ChromaDB embeddings → cited RAG → Ragas evaluation
- [ ] **Phase 3 — MCP + agent.** MCP server exposing warehouse + RAG as tools, driven from any MCP client

> Sections below marked _(fill in after running)_ are placeholders. Replace them with real
> numbers from your own run before sharing this repo.

---

## The problem

A D2C beauty brand sells across Amazon, Nykaa, quick-commerce and its own site. Every month
it collects tens of thousands of reviews. Somewhere in that pile is the answer to questions
like:

> *"Our face wash dropped 0.4 stars last quarter. Which SKUs, why, and what should R&D fix first?"*

Today that answer takes an analyst a week of spreadsheet work, and the reasoning is thrown
away as soon as the deck is delivered. ConsumerPulse makes it a question you ask in one line
and get back in seconds, with the actual consumer reviews cited as evidence.

---

## What it does

| Capability | Description |
|---|---|
| **Ingestion & validation** | pandas pipeline: dedupe, normalize timestamps, enforce schema, quarantine malformed rows |
| **Warehouse analytics** | BigQuery tables + SQL transforms using CTEs and window functions for rolling rating trends, theme share, and SKU ranking |
| **LLM enrichment** | Every review classified into `{theme, sentiment, severity, product_attribute}` and written back as an enriched table |
| **Semantic search** | ChromaDB embeddings over review text for retrieval by meaning, not keyword |
| **Cited RAG** | Natural-language questions answered from retrieved reviews, with source review IDs attached to every claim |
| **MCP server** | Four tools exposed over Model Context Protocol so any MCP client can query the warehouse conversationally |
| **Evaluation** | Ragas scoring on a golden Q&A set — faithfulness, answer relevancy, context precision |
| **Dashboard** | Streamlit app for trend charts, theme breakdowns and SKU comparison |

---

## Architecture

```
  Marketplace reviews (CSV / JSON)
            │
            ▼
   ┌─────────────────┐
   │  ingest.py      │   pandas — clean, dedupe, normalize, validate
   └────────┬────────┘
            ▼
   ┌─────────────────────────────────────────────┐
   │              BigQuery                       │
   │  raw_reviews ──► SQL transforms (CTEs +     │
   │                  window functions)          │
   │     ├─ rating_trend.sql    MoM + rolling avg│
   │     ├─ complaint_themes.sql  share, QoQ Δ   │
   │     └─ sku_cohorts.sql     repeat signal    │
   │  enriched_reviews ◄── LLM labels written back│
   └────────┬────────────────────────┬───────────┘
            │                        │
            ▼                        ▼
   ┌─────────────────┐      ┌─────────────────┐
   │   enrich.py     │      │   analytics.py  │
   │  Groq + Llama   │      │  SQL → DataFrame│
   │  classification │      └────────┬────────┘
   └────────┬────────┘               │
            ▼                        │
   ┌─────────────────┐               │
   │    ChromaDB     │               │
   │  embeddings     │               │
   └────────┬────────┘               │
            ▼                        │
   ┌─────────────────┐               │
   │     rag.py      │               │
   │ retrieve + cite │               │
   └────────┬────────┘               │
            │                        │
            └────────┬───────────────┘
                     ▼
            ┌─────────────────┐
            │  mcp_server.py  │  4 tools over MCP
            └────────┬────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
   MCP client              Streamlit dashboard
   (Claude Desktop)        (app.py)
```

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Data processing | **pandas** | Cleaning and validation before the warehouse |
| Warehouse | **Google BigQuery** | Serverless, SQL-native, free sandbox tier |
| Local dev warehouse | **DuckDB** | Same SQL, runs offline with no cloud credentials |
| LLM | **Groq** (`llama-3.3-70b-versatile`) | Low-latency batch classification |
| Vector store | **ChromaDB** | Lightweight embeddings, no external service |
| Protocol | **MCP** (`mcp` Python SDK) | Exposes the warehouse as tools to any MCP client |
| Evaluation | **Ragas** | Faithfulness, answer relevancy, context precision |
| UI | **Streamlit** + Plotly | Fast analytical dashboard |

---

## Repository structure

```
consumerpulse/
├── README.md
├── requirements.txt
├── .env.example
├── app.py                      # Streamlit dashboard
├── data/                       # local review data (gitignored)
├── sql/
│   ├── 01_schema.sql           # table definitions
│   ├── rating_trend.sql        # MoM + 3-month rolling average per SKU
│   ├── complaint_themes.sql    # theme share by SKU × quarter, QoQ delta
│   └── sku_cohorts.sql         # reviewer cohort / repeat-purchase signal
├── scripts/
│   └── fetch_data.py           # pull the public review dataset
└── src/voc/
    ├── config.py               # env + settings
    ├── ingest.py               # pandas clean → warehouse load
    ├── warehouse.py            # BigQuery / DuckDB abstraction
    ├── analytics.py            # run SQL files → DataFrames
    ├── enrich.py               # LLM classification → enriched table
    ├── rag.py                  # ChromaDB index + cited answers
    ├── evaluate.py             # Ragas on the golden set
    └── mcp_server.py           # MCP tool definitions
```

---

## Getting started

### Prerequisites

- Python 3.11+
- A Google Cloud project with the **BigQuery sandbox** enabled (free, no billing card required)
- A **Groq** API key (free tier) — only needed for Phase 2 onward

### 1. Install

```bash
git clone https://github.com/<your-username>/consumerpulse.git
cd consumerpulse
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure

```bash
cp .env.example .env
```

```ini
# .env
GROQ_API_KEY=your_key_here
GROQ_MODEL=llama-3.3-70b-versatile

GCP_PROJECT_ID=your-project-id
BQ_DATASET=consumerpulse
GOOGLE_APPLICATION_CREDENTIALS=./credentials/service-account.json

# Use "duckdb" to run entirely offline, or "bigquery" for the cloud warehouse
WAREHOUSE_BACKEND=bigquery
CHROMA_PERSIST_DIR=./.chroma
```

> `.env` and `credentials/` are gitignored. Never commit either.

### 3. Get the data

This project uses real public marketplace reviews from the beauty and personal-care category.

```bash
python scripts/fetch_data.py --category All_Beauty --limit 50000
```

The default source is the **Amazon Reviews 2023** dataset (`McAuley-Lab/Amazon-Reviews-2023`
on HuggingFace), which ships an `All_Beauty` category split with ratings, timestamps and
product IDs. Any review CSV with the columns below will work:

| Column | Type | Notes |
|---|---|---|
| `review_id` | string | Unique |
| `product_id` | string | SKU identifier |
| `product_title` | string | |
| `rating` | int | 1–5 |
| `review_text` | string | |
| `review_date` | date | |

### 4. Run the pipeline

```bash
python -m voc.ingest          # clean + load into the warehouse
python -m voc.analytics       # run the SQL transforms
python -m voc.enrich          # LLM classification  (Phase 2)
python -m voc.rag --index     # build the embedding index  (Phase 2)
```

### 5. Launch

```bash
streamlit run app.py          # dashboard at http://localhost:8501
python -m voc.mcp_server      # MCP server over stdio  (Phase 3)
```

---

## The analytics layer

Transforms live as plain `.sql` files in `sql/` so they can be read, reviewed and run
directly against BigQuery. Example — rolling rating trend with a month-over-month delta:

```sql
WITH monthly AS (
  SELECT
    product_id,
    DATE_TRUNC(review_date, MONTH) AS month,
    AVG(rating)                    AS avg_rating,
    COUNT(*)                       AS review_count
  FROM `${PROJECT}.${DATASET}.raw_reviews`
  GROUP BY product_id, month
),
trended AS (
  SELECT
    product_id,
    month,
    avg_rating,
    review_count,
    AVG(avg_rating) OVER (
      PARTITION BY product_id ORDER BY month
      ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS rolling_3m_rating,
    LAG(avg_rating) OVER (
      PARTITION BY product_id ORDER BY month
    ) AS prev_month_rating
  FROM monthly
)
SELECT
  *,
  ROUND(avg_rating - prev_month_rating, 2) AS mom_delta
FROM trended
WHERE review_count >= 10
ORDER BY mom_delta ASC;
```

---

## LLM enrichment

Each review is classified into a structured record and written back to the warehouse, which
makes qualitative text queryable with ordinary SQL:

```json
{
  "review_id": "R1A2B3C4",
  "theme": "packaging",
  "sentiment": "negative",
  "severity": "high",
  "product_attribute": "pump dispenser",
  "summary": "Pump jammed after two weeks of use"
}
```

Classification is batched and cached on disk, so re-runs cost nothing and the pipeline is
idempotent.

---

## Cited RAG

Answers are grounded in retrieved reviews, and every response carries the review IDs it drew
from, so a claim can always be traced back to a real consumer:

```
Q: What are the top complaints for the vitamin C serum this quarter?

A: Three themes dominate:
   1. Oxidation / colour change within weeks  (41 reviews)
   2. Dropper leaking during shipping          (27 reviews)
   3. Strong fragrance causing irritation      (19 reviews)

   Sources: R8821, R8904, R9013, R9122, R9245 ...
```

---

## MCP server

The warehouse and RAG layer are exposed as MCP tools, so any MCP-compatible client can query
the data conversationally.

| Tool | Purpose |
|---|---|
| `rating_trend` | Rolling and month-over-month rating movement for a SKU or category |
| `top_complaints` | Ranked complaint themes for a SKU over a time window |
| `compare_skus` | Side-by-side theme and rating comparison across SKUs |
| `ask_reviews` | Free-form question answered over the review corpus, with citations |

Register it with Claude Desktop by adding this to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "consumerpulse": {
      "command": "python",
      "args": ["-m", "voc.mcp_server"],
      "cwd": "/absolute/path/to/consumerpulse",
      "env": { "WAREHOUSE_BACKEND": "bigquery" }
    }
  }
}
```

---

## Evaluation

RAG quality is measured with Ragas against a golden set of questions in
`data/golden_set.json`, each with a human-written reference answer.

```bash
python -m voc.evaluate
```

| Metric | Score |
|---|---|
| Faithfulness | _(fill in after running)_ |
| Answer relevancy | _(fill in after running)_ |
| Context precision | _(fill in after running)_ |
| Golden set size | _(fill in after running)_ |

---

## Questions it can answer

- Which five SKUs had the largest rating decline last quarter?
- What changed in consumer complaints for the face wash between Q1 and Q2?
- Which product attribute generates the most high-severity complaints?
- Are packaging complaints rising faster than formulation complaints?
- Show me the actual reviews behind the drop in ratings for product X.

---

## Roadmap

- [ ] Phase 1 — analytics spine
- [ ] Phase 2 — LLM enrichment, RAG, evaluation
- [ ] Phase 3 — MCP server and agent
- [ ] Multi-marketplace ingestion (Amazon, Nykaa, quick-commerce)
- [ ] Anomaly alerting on sudden theme spikes
- [ ] Competitor benchmarking across brands
- [ ] Scheduled refresh via Cloud Scheduler

---

## Acknowledgements

Review data from the **Amazon Reviews 2023** dataset (McAuley Lab, UCSD). This project is an
independent portfolio build and is not affiliated with or endorsed by any brand whose
products appear in the public review data.

## License

MIT — see [LICENSE](LICENSE).
