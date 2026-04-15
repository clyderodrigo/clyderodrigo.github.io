# Fast Enough to Matter: Agent Retrieval Latency in Enterprise Data Pipelines

## Key Takeaways

- The dominant latency in enterprise AI agents is the query execution roundtrip, not model inference. Optimize the data layer first.
- Dimensional data (warehouses, OLAP) and transactional data (OLTP) require different retrieval strategies. Don't apply the same pattern to both.
- Semantic layers pre-validate metrics and dramatically reduce both latency and NL→SQL error rates for dimensional data.
- DuckDB over columnar lakehouse formats eliminates the managed warehouse roundtrip for sub-100GB analytical workloads.
- Vector search is for unstructured text. Structured enterprise data needs structured query execution.
- Layer approaches: semantic layer for defined KPIs, fine-tuning for stable query patterns, NL→SQL for ad hoc. Route before inferring.

---

An agent that answers correctly in 8 seconds loses to a dashboard that answers approximately in 200ms. In enterprise AI, the dominant latency problem is not model inference — it's the roundtrip from natural language to query to data lake and back. That's the problem this article is about.

This article covers the three architectural approaches to enterprise data retrieval — tool-based query execution, grounding, and fine-tuning — and the storage patterns that are better optimized for AI workloads than traditional relational or vector models.

---

## The Enterprise Retrieval Roundtrip

When an agent answers "What were Q3 net margins by region?" against a data warehouse, the latency profile looks nothing like a typical RAG call:

```
User Question
    │
    ▼
[1] Schema Resolution      ← identify tables, dimensions, metrics (~50–200ms)
    │
    ▼
[2] NL → Query Generation  ← model call to produce SQL/MDX/API call (~300–800ms)
    │
    ▼
[3] Query Execution        ← roundtrip to data lake / warehouse (~500ms–30s)
    │
    ▼
[4] Result Synthesis       ← model call to interpret and narrate results (~200–500ms)
    │
    ▼
[5] Response to User
```

Steps 3 and 2 dominate. Model inference on steps 2 and 4 is often the smallest component. Optimizing model selection or streaming (the usual advice) has almost no impact on what the user feels. The query roundtrip does.

Enterprise data also splits into two fundamentally different shapes:

- **Dimensional data** — data warehouses, star/snowflake schemas, OLAP cubes. Queries aggregate across large fact tables joined to dimension tables. BigQuery, Snowflake, Redshift, Databricks. Latency is query-engine latency: seconds to minutes for cold queries.
- **Transactional data** — OLTP systems, operational databases. Row-level, real-time, high write throughput. Postgres, Oracle, SQL Server. Latency is fast for simple lookups, slow for analytical queries pushed against them.

These require different retrieval strategies.

---

## Approach 1: Tool-Based Retrieval (NL-to-Query)

The agent calls a tool — `query_warehouse`, `run_sql`, `call_api` — that generates and executes a query at runtime. Every question triggers a full roundtrip.

```python
import anthropic
import duckdb

client = anthropic.Anthropic()

tools = [
    {
        "name": "query_data_lake",
        "description": (
            "Execute an analytical SQL query against the enterprise data lake. "
            "Available tables: sales_fact, customer_dim, product_dim, date_dim, region_dim. "
            "Use for questions about revenue, margins, volume, and trends."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "sql": {
                    "type": "string",
                    "description": "Valid DuckDB SQL query. Use only listed tables."
                }
            },
            "required": ["sql"]
        }
    }
]

def run_agent_query(user_question: str) -> str:
    messages = [{"role": "user", "content": user_question}]

    # Step 1: NL → SQL generation
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    # Step 2: Execute the generated query
    if response.stop_reason == "tool_use":
        tool_call = next(b for b in response.content if b.type == "tool_use")
        sql = tool_call.input["sql"]

        con = duckdb.connect("enterprise.duckdb")
        result_df = con.execute(sql).df()
        result_text = result_df.to_string(index=False)

        # Step 3: Synthesize result
        messages += [
            {"role": "assistant", "content": response.content},
            {"role": "user", "content": [
                {"type": "tool_result", "tool_use_id": tool_call.id, "content": result_text}
            ]}
        ]
        final = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )
        return final.content[0].text

    return response.content[0].text
```

**Latency profile:**
- NL → SQL generation: 300–800ms (model call with schema context)
- Query execution: 500ms–30s depending on warehouse, query complexity, cold/warm cache
- Result synthesis: 200–500ms

**Total: 1–32 seconds per question.**

**Use tool-based retrieval when:**
- Questions are ad hoc and unpredictable
- Data changes continuously (transactional systems, live operational data)
- Accuracy on exact figures is more important than speed
- You need full auditability (the SQL is inspectable)

**The hidden cost:** Schema resolution. The model needs to know what tables and columns exist. Without a well-structured schema prompt, NL→SQL accuracy degrades fast. For dimensional data, include the semantic meaning of each dimension — not just column names.

```python
SCHEMA_CONTEXT = """
Tables and their business meaning:
- sales_fact: one row per transaction. Columns: date_key, region_key, product_key, customer_key, revenue, units, cogs
- region_dim: region_key, region_name, country, territory (use for geographic breakdowns)
- date_dim: date_key, fiscal_year, fiscal_quarter, fiscal_month (always join on date_key for time filters)
- product_dim: product_key, product_name, category, subcategory, list_price
Metrics defined:
- net_margin = (revenue - cogs) / revenue
- Always filter date_dim.fiscal_year for year-scoped questions
"""
```

---

## Approach 2: Grounding — Pre-Materializing Enterprise Knowledge

Instead of executing queries at runtime, pre-compute answers and ground the agent in materialized results. The model reads pre-built data rather than querying live.

This is particularly effective for dimensional data where the "important" answers — KPIs, aggregated metrics, period-over-period comparisons — are knowable in advance.

### Semantic Layer Grounding

A semantic layer (dbt Semantic Layer, Cube.js, AtScale) pre-defines metrics and exposes them as queryable objects rather than raw SQL tables. The agent queries the semantic layer, not raw tables — queries are faster, less error-prone, and return pre-validated metrics.

```python
import httpx
import anthropic

client = anthropic.Anthropic()

# Semantic layer exposes metrics, not raw SQL
AVAILABLE_METRICS = """
Pre-defined metrics available via the semantic layer:
- net_margin(period, region, product_category) → float
- revenue_growth_pct(period, comparison_period, region) → float
- units_sold(period, region, product_category) → int
- customer_retention_rate(cohort_period, region) → float

Dimensions for filtering: region, product_category, fiscal_period (YYYY-QQ format)
"""

tools = [
    {
        "name": "query_semantic_layer",
        "description": f"Query pre-defined business metrics. {AVAILABLE_METRICS}",
        "input_schema": {
            "type": "object",
            "properties": {
                "metric": {"type": "string"},
                "dimensions": {
                    "type": "object",
                    "description": "Filter values, e.g. {'region': 'EMEA', 'fiscal_period': '2025-Q3'}"
                }
            },
            "required": ["metric", "dimensions"]
        }
    }
]

def query_semantic_layer(metric: str, dimensions: dict) -> dict:
    # Semantic layer handles SQL generation and caching internally
    response = httpx.post(
        "https://semantic-layer.internal/query",
        json={"metric": metric, "dimensions": dimensions},
        timeout=5.0
    )
    return response.json()
```

**Why this beats raw NL→SQL for dimensional data:** Semantic layers cache metric computations, enforce business logic centrally (e.g., "net margin always excludes intercompany"), and dramatically reduce the schema complexity the model needs to reason over. Query latency drops from seconds to sub-second for cached metrics.

### Pre-Computed Grounding for Transactional Data

For OLTP systems, analytical queries should never hit the transactional database directly. Pre-aggregate into a read-optimized store and ground the agent there.

```python
import json
from datetime import datetime, timezone

# Scheduled job: snapshot operational metrics into a grounding store
def materialize_operational_snapshot(oltp_conn, grounding_store):
    snapshot = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "open_orders": {
            "count": oltp_conn.execute("SELECT COUNT(*) FROM orders WHERE status='open'").scalar(),
            "total_value": oltp_conn.execute("SELECT SUM(total) FROM orders WHERE status='open'").scalar(),
            "oldest_open_days": oltp_conn.execute(
                "SELECT EXTRACT(DAY FROM NOW() - MIN(created_at)) FROM orders WHERE status='open'"
            ).scalar()
        },
        "inventory_alerts": oltp_conn.execute(
            "SELECT sku, quantity_on_hand, reorder_point "
            "FROM inventory WHERE quantity_on_hand < reorder_point"
        ).fetchall()
    }
    # Write to fast-read store (Redis, document DB, etc.)
    grounding_store.set("ops_snapshot", json.dumps(snapshot), ex=300)  # 5-min TTL
```

The agent reads the pre-built snapshot rather than hitting the transactional system. Latency: sub-50ms. Freshness: within the snapshot TTL.

---

## Approach 3: Fine-Tuning on Schema and Domain

Fine-tuning bakes schema knowledge, query patterns, and business rules into model weights. For companies with stable dimensional models (the DW schema doesn't change weekly), this trades upfront cost for persistent zero-latency schema resolution.

```python
from openai import OpenAI

client = OpenAI()

# Training data: (question, SQL) pairs specific to your schema
# Generated from query logs, BI tool history, analyst Q&A
file = client.files.create(
    file=open("enterprise_nl_sql_pairs.jsonl", "rb"),
    purpose="fine-tune"
)

job = client.fine_tuning.jobs.create(
    training_file=file.id,
    model="gpt-4o-mini-2024-07-18",
    hyperparameters={"n_epochs": 4}
)
```

Training sample format — include schema, business context, and target SQL:

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You generate SQL for our enterprise DW. Schema: sales_fact (date_key, region_key, product_key, revenue, cogs), region_dim (region_key, region_name, territory), date_dim (date_key, fiscal_year, fiscal_quarter). net_margin = (revenue-cogs)/revenue."
    },
    {
      "role": "user",
      "content": "What was APAC net margin in Q2 fiscal 2025?"
    },
    {
      "role": "assistant",
      "content": "SELECT AVG((s.revenue - s.cogs) / s.revenue) AS net_margin FROM sales_fact s JOIN region_dim r ON s.region_key = r.region_key JOIN date_dim d ON s.date_key = d.date_key WHERE r.territory = 'APAC' AND d.fiscal_year = 2025 AND d.fiscal_quarter = 2"
    }
  ]
}
```

**What fine-tuning removes:** The schema resolution step and a significant portion of the NL→SQL generation latency. A fine-tuned model generates correct SQL for known query patterns in ~100ms rather than 600ms for a base model working from a schema prompt.

**What fine-tuning does not remove:** Query execution latency. The warehouse roundtrip is unchanged.

**Use fine-tuning when:** Your dimensional model is stable, you have 500+ (question, SQL) pairs from real usage, and you can establish a retraining pipeline for model freshness.

---

## AI-Optimized Storage: Beyond Relational and Vector

Standard relational databases and generic vector stores are not the right primary store for AI agent data retrieval in enterprise contexts. The patterns that actually perform:

### Columnar / Lakehouse Formats (Dimensional Data)

Apache Iceberg and Delta Lake with columnar Parquet storage are the right foundation for analytical AI queries. They support partition pruning, column projection, and incremental snapshots — all of which reduce the data scanned per query.

```python
import duckdb

# DuckDB reads Iceberg directly, with predicate pushdown
# This query scans only Q3 2025 partitions, only revenue+cogs columns
con = duckdb.connect()
result = con.execute("""
    SELECT region_name, AVG((revenue - cogs) / revenue) as net_margin
    FROM iceberg_scan('s3://datalake/sales_fact/')
    JOIN iceberg_scan('s3://datalake/region_dim/') USING (region_key)
    WHERE fiscal_year = 2025 AND fiscal_quarter = 3
    GROUP BY region_name
""").df()
```

DuckDB is particularly effective for AI workloads: it runs embedded (no network roundtrip), queries Parquet/Iceberg/Delta directly on object storage, and handles analytical workloads that would require a full Spark cluster in older architectures. For agent tool calls over sub-100GB datasets, DuckDB eliminates the warehouse roundtrip entirely.

### Feature Stores (Pre-Computed AI-Ready Features)

For ML-heavy agents (scoring, ranking, anomaly detection), a feature store pre-computes and serves feature vectors at low latency. Feast and Tecton separate feature computation (batch/streaming) from feature serving (online, sub-10ms).

```python
from feast import FeatureStore

store = FeatureStore(repo_path="feature_repo/")

# Online retrieval: pre-computed, sub-10ms
feature_vector = store.get_online_features(
    features=[
        "customer_features:lifetime_value",
        "customer_features:churn_risk_score",
        "customer_features:days_since_last_order",
    ],
    entity_rows=[{"customer_id": customer_id}]
).to_dict()
```

This is the right pattern for agents that need to score, rank, or assess entities (customers, products, transactions) in real time. The alternative — joining raw transactional tables at query time — is 100–1000x slower.

### Materialized Semantic Caches (Read-Optimized for Common Questions)

For the 20% of questions that constitute 80% of agent query volume, pre-materialize results keyed by semantic hash. This is different from prompt caching — it caches the data result, not the model's prompt prefix.

```python
import hashlib
import json
import redis

cache = redis.Redis(host="redis.internal", port=6379, decode_responses=True)

def get_or_compute_metric(metric: str, dimensions: dict, ttl_seconds: int = 300) -> dict:
    # Stable cache key from metric + dimensions
    key = hashlib.sha256(
        json.dumps({"metric": metric, "dimensions": dimensions}, sort_keys=True).encode()
    ).hexdigest()

    cached = cache.get(f"metric:{key}")
    if cached:
        return json.loads(cached)

    # Cache miss: execute against warehouse
    result = execute_warehouse_query(metric, dimensions)
    cache.setex(f"metric:{key}", ttl_seconds, json.dumps(result))
    return result
```

For dimensional data with natural time-based grain (daily KPIs, weekly aggregates), yesterday's figures don't change — TTLs of hours to days are appropriate. This collapses warehouse latency to Redis latency (~1ms) for repeated questions.

### When Not to Use Vector Search

Semantic/vector search is the right tool for unstructured text retrieval (documents, notes, support tickets). It is the wrong tool for structured enterprise data:

| Data Type | Right Storage Pattern |
|---|---|
| Dimensional OLAP data | Columnar lakehouse (Iceberg/Delta) + semantic layer |
| Transactional OLTP data | Materialized snapshots + feature store for scoring |
| Slowly-changing KPIs | Semantic result cache (Redis) with TTL |
| Entity features (ML) | Feature store (Feast, Tecton) |
| Document/text content | Vector store (pgvector, Pinecone, Weaviate) |
| Mixed enterprise queries | Routing layer → appropriate store per query type |

Applying vector search to structured dimensional data produces approximately correct answers at high latency. The model hallucinates metrics it can't retrieve precisely. Structured query execution — even with its roundtrip cost — produces exact answers.

---

## Choosing an Approach

| | Tool-Based NL→Query | Semantic Layer Grounding | Fine-Tuning |
|---|---|---|---|
| Query freshness | Real-time | TTL-bounded | Static (retrain for schema changes) |
| Roundtrip latency | 1–30s | 100ms–2s | 100ms–2s (execution unchanged) |
| SQL/query accuracy | Depends on schema prompt quality | High (pre-defined metrics) | High for trained patterns |
| Schema change handling | Update prompt | Update semantic layer | Retrain |
| Ad hoc question support | Full | Limited to defined metrics | Trained patterns only |
| Transactional data | Yes (with care) | Via snapshots | Risky (schema drift) |
| Dimensional data | Yes | Optimal | Good for stable schemas |

In practice the right production architecture layers all three: fine-tune for common query patterns on the stable dimensional model, expose a semantic layer for defined KPIs, and fall back to tool-based NL→SQL for ad hoc questions. Route by question type before invoking the model.

```python
def route_enterprise_query(question: str, question_type: str) -> str:
    if question_type == "defined_kpi":
        return query_semantic_layer(question)       # sub-second, pre-validated
    elif question_type == "known_pattern":
        return query_with_finetuned_model(question) # fast SQL gen, exact execution
    else:
        return query_with_nl_to_sql(question)       # full roundtrip, ad hoc
```

