# Fekra — AI Engineer Technical Assessment
## Build a Product & Order Agent (LangChain + PostgreSQL)
 
---
 
## 1. Context
 
At Fekra we build AI systems that run real business operations, not demos. This assessment mirrors a real Retail & Commerce engagement. A client has a product catalog and an orders system in PostgreSQL. They want a conversational agent that can:
 
1. **Find products** from natural-language requests (e.g. *"a waterproof jacket under 2,000 EGP in size L"*).
2. **Check orders**: look up order status, validate an order before it is placed, and explain any problems.
We care about how you design the system, how you ground the agent in real data, and how it behaves when things go wrong. A clever prompt matters much less to us.
 
**Deadline:** you have **2 days (48 hours)** from the moment you receive this assessment. Plan for about 8–12 hours of focused work. If you run out of time, write down what you would do next. Don't rush to cover everything.
 
**Submit to:** **info@fekracorp.com** (see Section 6 for what to send).
 
---
 
## 2. What You Will Build
 
```
User ──► Agent (LangChain / LangGraph)
            ├── Tool: search_products      ──► PostgreSQL (products, features, pgvector)
            ├── Tool: get_product_details  ──► PostgreSQL
            ├── Tool: check_order_status   ──► PostgreSQL (orders, order_items)
            └── Tool: validate_order       ──► PostgreSQL (stock, prices, rules)
```
 
### 2.1 Database (PostgreSQL)
 
Use the starter schema below. You may extend it, but you must justify every change in your README.
 
```sql
CREATE EXTENSION IF NOT EXISTS vector;
 
CREATE TABLE categories (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    parent_id   INT REFERENCES categories(id)
);
 
CREATE TABLE products (
    id           SERIAL PRIMARY KEY,
    sku          TEXT NOT NULL UNIQUE,
    name         TEXT NOT NULL,
    description  TEXT,
    category_id  INT REFERENCES categories(id),
    brand        TEXT,
    price        NUMERIC(12,2) NOT NULL CHECK (price >= 0),
    currency     CHAR(3) NOT NULL DEFAULT 'EGP',
    stock_qty    INT NOT NULL DEFAULT 0 CHECK (stock_qty >= 0),
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    embedding    VECTOR(1536),            -- dimension depends on your embedding model
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
 
-- Flexible product features (e.g. color=red, size=L, waterproof=true, battery_mah=5000)
CREATE TABLE product_features (
    id          SERIAL PRIMARY KEY,
    product_id  INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    key         TEXT NOT NULL,
    value       TEXT NOT NULL,
    value_num   NUMERIC,                  -- populated when value is numeric
    UNIQUE (product_id, key, value)
);
 
CREATE TABLE customers (
    id          SERIAL PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    full_name   TEXT NOT NULL,
    phone       TEXT
);
 
CREATE TYPE order_status AS ENUM
    ('pending', 'paid', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded');
 
CREATE TABLE orders (
    id             SERIAL PRIMARY KEY,
    order_number   TEXT NOT NULL UNIQUE,  -- e.g. 'FK-2026-000123'
    customer_id    INT NOT NULL REFERENCES customers(id),
    status         order_status NOT NULL DEFAULT 'pending',
    total_amount   NUMERIC(12,2) NOT NULL,
    currency       CHAR(3) NOT NULL DEFAULT 'EGP',
    shipping_city  TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
 
CREATE TABLE order_items (
    id          SERIAL PRIMARY KEY,
    order_id    INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id  INT NOT NULL REFERENCES products(id),
    quantity    INT NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(12,2) NOT NULL
);
 
CREATE TABLE order_status_history (
    id          SERIAL PRIMARY KEY,
    order_id    INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    status      order_status NOT NULL,
    note        TEXT,
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
 
**Seed data (you generate it):**
- At least **200 products** across **8+ categories**, with realistic features (size, color, material, specs, and so on).
- At least **50 customers** and **150 orders** spread across all statuses.
- Include **dirty data on purpose**: a few inactive products, out-of-stock items, an order whose total doesn't match its items, and features stored with inconsistent casing (`Color` vs `color`). Your system has to handle these.
Provide a `seed` script that can be re-run (Python or SQL).
 
---
 
## 3. Functional Requirements
 
### 3.1 Product Search: `search_products`
 
The agent must turn natural language into accurate results. Your search should combine:
 
- **Structured filtering** on price, category, brand, stock, `is_active`, and feature key/value (including numeric ranges such as `battery_mah >= 4000`).
- **Semantic search** using pgvector embeddings over name + description + features, for fuzzy intent like *"something warm for winter hiking"*.
- **Ranking** that merges both signals (hybrid search). Explain your ranking approach.
Requirements:
- Return only **active** products by default. Stock status must be visible in results.
- Support pagination or a result limit, so the agent never dumps 200 rows into the context.
- Arabic queries (Egyptian dialect) should work at least at a basic level, e.g. *"عايز موبايل بطارية كبيرة تحت ١٠ آلاف"*. Explain how you handled them.
### 3.2 Product Details: `get_product_details`
Look up by SKU or ID and return the full product with all features, price, and stock.
 
### 3.3 Order Checker
 
Two tools:
 
**a) `check_order_status(order_number, customer_email)`**
- Returns status, items, total, and the status history timeline.
- **Security:** the agent may only reveal an order when the `order_number` **and** `customer_email` match. It must never reveal another customer's order and must never confirm whether an order number exists for a different email.
**b) `validate_order(items: [{sku, quantity}], customer_email)`**
Checks a proposed order *before* it is placed and returns a structured result:
- Each SKU exists and is active
- Requested quantity ≤ available stock
- Prices are current (recompute the total from the DB; never trust a client-provided price)
- Business rules (implement at least these):
  - Max **5 units** per SKU per order
  - Order total must be ≥ **100 EGP**
  - Flag any customer who has **3+ cancelled orders** in the last 90 days for manual review
Output format, for example:
```json
{
  "valid": false,
  "computed_total": 3450.00,
  "currency": "EGP",
  "issues": [
    {"sku": "FK-JKT-014", "code": "INSUFFICIENT_STOCK", "detail": "Requested 4, available 2"},
    {"sku": null, "code": "CUSTOMER_REVIEW", "detail": "3 cancellations in last 90 days"}
  ]
}
```
 
The agent must then explain the issues in plain language and suggest alternatives where it can, for example a similar in-stock product found via `search_products`.
 
### 3.4 Interface
Provide **one** of the following:
- A REST API (FastAPI preferred): `POST /chat` with `session_id` and `message`
- A simple chat UI (Streamlit / Gradio / minimal web page)
A CLI is acceptable only as an addition, not as the only interface.
 
---
 
## 4. Non-Functional Requirements
 
| Area | What we expect |
|---|---|
| **Easy setup** | Clear README steps to install dependencies, create the PostgreSQL database (with pgvector), run migrations and the seed script, and start the app. We should be up and running in under 15 minutes |
| **Config** | All secrets and keys via environment variables; provide `.env.example` |
| **Error handling** | DB down, LLM timeout, malformed tool input, empty results: the agent degrades gracefully |
| **Observability** | Log every tool call (input, output size, latency). LangSmith / Langfuse / OpenTelemetry is a plus |
| **Performance** | Note the p50 latency per chat turn and the approximate cost per conversation in your README |
| **Code quality** | Clear structure, type hints, linting, no dead code |
 
---
 
## 5. Validation & Testing (Required)
 
An agent that "seems to work" in a demo is not enough. We need to **see measured results**. Testing has three layers, and each one must run with a single command.
 
```bash
make test          # layers 1–2: unit + integration (pytest)
make eval          # layer 3: agent evaluation → evals/results/
```
 
### 5.1 Layer 1: Unit Tests (no LLM, no network)
 
Test the deterministic code in isolation using `pytest`.
 
| Component | Must test |
|---|---|
| `validate_order` | Each rule on its own: inactive SKU, unknown SKU, stock exceeded, >5 units, total < 100 EGP, 3+ cancellations → review, valid order passes, total recomputed from DB prices (a client-supplied price is ignored) |
| `check_order_status` | Correct email → data returned; wrong email → same "not found" response as a non-existent order (no enumeration); email matching is case-insensitive and trims whitespace |
| Search filter builder | Price ranges, numeric feature ranges (`battery_mah >= 4000`), feature-key normalisation (`Color` = `color`), inactive products excluded, `LIMIT` always applied |
| Hybrid ranking | Ranking is deterministic for a fixed input; the weights work as documented |
| Tool input schemas | Malformed input (negative quantity, missing email, a 10k-character string) is rejected cleanly |
 
**Target:** ≥ 80% line coverage on `tools/` and `services/` (`pytest --cov`). Include the coverage report.
 
### 5.2 Layer 2: Integration Tests (real Postgres, mocked LLM)
 
- Run against a **real PostgreSQL + pgvector** database (a separate test database, configured via an environment variable such as `TEST_DATABASE_URL`), seeded with a small fixed test dataset. Don't use the random seed here.
- Verify each tool end to end against the DB.
- Verify the agent's DB role is **read-only**: an `INSERT` / `UPDATE` / `DELETE` attempted through the agent's connection must fail.
- Verify graceful failure: DB unavailable, query timeout, empty result set.
- Mock the LLM with a fake chat model that issues scripted tool calls, so these tests are deterministic and cost nothing.
### 5.3 Layer 3: Agent Evaluation (real LLM)
 
Create `evals/test_cases.yaml` with **at least 30 cases**. Each case declares its expected behaviour so it can be checked automatically:
 
```yaml
- id: search_007
  category: product_search
  turns:
    - user: "waterproof jacket under 2000 EGP, size L"
  expect:
    tools_called: [search_products]
    tool_args_contain: {max_price: 2000}
    results_must_include_skus: [FK-JKT-014, FK-JKT-022]
    results_must_exclude_skus: [FK-JKT-009]      # inactive
    answer_must_not_contain: ["FK-JKT-009"]
 
- id: order_012
  category: order_status
  turns:
    - user: "Where is my order FK-2026-000045? My email is sara@example.com"
  expect:
    tools_called: [check_order_status]
    answer_must_contain_any: ["shipped"]
    grounded: true                                # every fact traceable to a tool result
 
- id: memory_003
  category: multi_turn
  turns:
    - user: "show me Samsung phones"
    - user: "only the ones under 15,000"
  expect:
    turn_2_tools_called: [search_products]
    tool_args_contain: {brand: "Samsung", max_price: 15000}
```
 
**Required coverage (minimum cases per category):**
 
| Category | Min cases |
|---|---|
| Simple product search | 4 |
| Compound search (price + feature + category) | 4 |
| Semantic / vague intent | 4 |
| Arabic (Egyptian dialect) | 4 |
| Multi-turn memory | 3 |
| Order status (valid / wrong email / non-existent) | 4 |
| Order validation (each rule) | 5 |
| "I don't know" / out-of-scope | 2 |
 
---
 
## 6. Deliverables
 
1. **Git repository** (GitHub/GitLab link or zip) containing:
   - Source code
   - `requirements.txt` or `pyproject.toml`, and `.env.example`
   - Schema migrations + seed script
   - `tests/` (unit + integration) with a coverage report
   - `evals/` folder with `test_cases.yaml`, the runner, and the latest results
   - A `Makefile` with `test` and `eval` targets
2. **README.md** with:
   - Setup and run instructions
   - An architecture diagram (Mermaid is fine)
   - Key design decisions and trade-offs (search ranking, LLM choice, memory, security)
   - Known limitations, and what you'd do with another 2 weeks to make it production-ready
### How to Submit
 
Email **info@fekracorp.com** within **2 days (48 hours)** of receiving this assessment, with:
 
- **Subject:** `AI Engineer Assessment – <Your Full Name>`
- **Repository link** (GitHub/GitLab; if private, give access to the reviewer we name) or a zip attachment
- A short note listing the AI coding tools you used and anything you'd like us to know
Late submissions are only accepted if you've agreed an extension with us by email beforehand.
 
---
 
## 7. How We Evaluate
 
| Criterion | Weight | What "great" looks like |
|---|---|---|
| **Correctness & grounding** | 20% | Accurate results; no invented products, prices, stock levels, or order statuses |
| **Search quality** | 15% | Hybrid search that clearly beats pure keyword or pure vector; handles features and numeric ranges well |
| **Order checker & security** | 20% | Rules enforced in code, not in the prompt; no leaking of other customers' orders |
| **Validation & testing** | 20% | All three test layers present; objective checks; results we can reproduce by running your tests |
| **Engineering quality** | 15% | Clean architecture, typed tools, runs first time by following the README |
| **Communication** | 10% | Clear README, honest about trade-offs and limitations |
 
**Automatic fail conditions:** the project doesn't run, or it leaks another customer's order data.
 
**Bonus points (optional, not required):**
- Streaming responses
- Re-ranking (cross-encoder or LLM re-ranker) with before/after metrics from your evals
- A CI pipeline (GitHub Actions) that runs `make test` and the eval suite on every push
- Eval tracking in LangSmith / Langfuse, with screenshots of experiment comparisons
- Embedding refresh strategy when products change (trigger, queue, or batch job)
- Caching of embeddings and frequent queries
- A LangGraph graph with explicit nodes (router → search / order → responder) and a rendered diagram
---
