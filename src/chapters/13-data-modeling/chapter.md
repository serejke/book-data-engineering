---
chapter: 13
title: "Data Modeling for Analytics"
estimated_pages: 40-45
status: draft
last_updated: 2025-01-22
---

# Data Modeling for Analytics

The previous chapters covered how to store, move, and transform data. But transformations without a coherent model produce chaos—a tangle of tables with unclear relationships, redundant data, and inconsistent metrics. Data modeling brings order to this chaos.

Data modeling is the practice of organizing data into structures that serve analytical needs. Unlike operational database design, which optimizes for transaction throughput, analytical modeling optimizes for query patterns, business understanding, and maintainability.

This chapter explores the major modeling paradigms—dimensional modeling, Data Vault, and One Big Table—not as competing religions but as tools with different strengths. Understanding when to apply each paradigm, and how to combine them, distinguishes competent data engineers from exceptional ones.

## Why Data Modeling Matters

### The Cost of Poor Modeling

Without deliberate modeling, analytics environments degrade:

**Inconsistent metrics:** Marketing calculates revenue one way, Finance another. Neither knows which is correct.

**Query complexity:** Analysts write 200-line queries joining 15 tables because no one built sensible abstractions.

**Maintenance burden:** Changing one upstream table breaks dozens of downstream reports—no one knows which ones until dashboards fail.

**Performance problems:** Queries scan terabytes because tables aren't organized for analytical access patterns.

**Trust erosion:** When numbers don't match, stakeholders stop trusting the data platform entirely.

### What Good Modeling Provides

**Consistency:** Single definitions for business metrics, enforced by model structure.

**Understandability:** Business users can navigate the data because it reflects how they think about the business.

**Performance:** Tables structured for common query patterns, with appropriate pre-aggregation.

**Maintainability:** Clear lineage, documented relationships, isolated changes.

**Flexibility:** New questions can be answered without rebuilding the entire model.

### Modeling vs. Schema Design

Don't confuse data modeling with database schema design:

**Schema design:** Technical structure—tables, columns, data types, constraints.

**Data modeling:** Conceptual organization—how entities relate, what facts we track, how dimensions describe them.

Schema design implements a data model, but the model exists independently of any particular database technology.

## Dimensional Modeling

Dimensional modeling, pioneered by Ralph Kimball in the 1990s, remains the dominant paradigm for analytical data warehouses. Its core insight: organize data around business processes and the context that describes them.

### Core Concepts

**Facts:** Measurable events that happened in the business. Sales transactions, website clicks, trade executions.

**Dimensions:** Context that describes facts. Who, what, where, when.

**Grain:** The level of detail in a fact table. One row per transaction? Per day? Per customer per month?

```
┌─────────────────────────────────────────────────────────────────┐
│                    Dimensional Model Structure                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌──────────────┐                           │
│                      │  dim_date    │                           │
│                      │  ──────────  │                           │
│                      │  date_key    │                           │
│                      │  date        │                           │
│                      │  day_of_week │                           │
│                      │  month       │                           │
│                      │  quarter     │                           │
│                      │  year        │                           │
│                      └──────┬───────┘                           │
│                             │                                    │
│  ┌──────────────┐    ┌──────┴───────┐    ┌──────────────┐      │
│  │ dim_symbol   │    │  fct_trades  │    │ dim_trader   │      │
│  │ ──────────── │    │  ──────────  │    │ ──────────── │      │
│  │ symbol_key   │◄───│ symbol_key   │───▶│ trader_key   │      │
│  │ symbol       │    │ date_key     │    │ trader_id    │      │
│  │ name         │    │ trader_key   │    │ name         │      │
│  │ exchange     │    │ quantity     │    │ type         │      │
│  │ base_ccy     │    │ price        │    │ region       │      │
│  └──────────────┘    │ value        │    └──────────────┘      │
│                      └──────────────┘                           │
│                                                                  │
│  Legend:                                                         │
│  ───▶  Foreign key relationship                                 │
│  Facts contain measures (quantity, price, value)                │
│  Dimensions contain descriptive attributes                      │
└─────────────────────────────────────────────────────────────────┘
```

### Fact Tables

Fact tables record business events at a specific grain.

**Transaction facts:** One row per event at the most granular level.
```sql
-- fct_trades: one row per trade execution
CREATE TABLE fct_trades (
    trade_key BIGINT,
    date_key INT,
    symbol_key INT,
    trader_key INT,
    -- Measures
    quantity DECIMAL(18, 8),
    price DECIMAL(18, 8),
    value DECIMAL(18, 2),
    commission DECIMAL(18, 2)
);
```

**Periodic snapshot facts:** One row per entity per time period.
```sql
-- fct_daily_positions: one row per symbol per day
CREATE TABLE fct_daily_positions (
    date_key INT,
    symbol_key INT,
    trader_key INT,
    -- Measures at end of day
    position_quantity DECIMAL(18, 8),
    position_value DECIMAL(18, 2),
    unrealized_pnl DECIMAL(18, 2)
);
```

**Accumulating snapshot facts:** One row per process instance, updated as the process progresses.
```sql
-- fct_order_lifecycle: one row per order, tracks milestones
CREATE TABLE fct_order_lifecycle (
    order_key BIGINT,
    -- Milestone date keys
    created_date_key INT,
    submitted_date_key INT,
    filled_date_key INT,
    settled_date_key INT,
    -- Measures at each stage
    original_quantity DECIMAL(18, 8),
    filled_quantity DECIMAL(18, 8),
    -- Lag metrics
    submit_to_fill_minutes INT,
    fill_to_settle_minutes INT
);
```

### Dimension Tables

Dimensions provide context for facts.

**Flat dimensions:** Simple descriptive attributes.
```sql
CREATE TABLE dim_symbol (
    symbol_key INT PRIMARY KEY,
    symbol VARCHAR(20),
    name VARCHAR(100),
    exchange VARCHAR(50),
    base_currency VARCHAR(10),
    quote_currency VARCHAR(10),
    asset_class VARCHAR(50),
    sector VARCHAR(50),
    is_active BOOLEAN
);
```

**Hierarchical dimensions:** Natural hierarchies within a dimension.
```sql
CREATE TABLE dim_geography (
    geography_key INT PRIMARY KEY,
    country VARCHAR(100),
    region VARCHAR(100),
    continent VARCHAR(50),
    -- Hierarchy levels
    country_code VARCHAR(3),
    region_code VARCHAR(10)
);
```

**Role-playing dimensions:** Same dimension used multiple ways in a fact.
```sql
-- fct_transfers uses dim_date three times
SELECT
    t.transfer_id,
    od.date as order_date,
    sd.date as ship_date,
    dd.date as delivery_date
FROM fct_transfers t
JOIN dim_date od ON t.order_date_key = od.date_key
JOIN dim_date sd ON t.ship_date_key = sd.date_key
JOIN dim_date dd ON t.delivery_date_key = dd.date_key;
```

### Star Schema vs Snowflake Schema

**Star schema:** Denormalized dimensions directly connected to facts.
```
        dim_date
            │
dim_symbol──fct_trades──dim_trader
            │
        dim_exchange
```

- Fewer joins
- Better query performance
- Some data redundancy

**Snowflake schema:** Normalized dimensions with sub-dimensions.
```
                dim_date
                    │
dim_symbol──fct_trades──dim_trader──dim_region
    │                                   │
dim_exchange                        dim_country
```

- Less redundancy
- More complex queries
- Harder for business users to navigate

**Modern recommendation:** Prefer star schemas. Storage is cheap; query complexity is expensive. Denormalization improves performance and usability.

### Slowly Changing Dimensions (SCD)

Dimensions change over time. How do you handle history?

**Type 1 (Overwrite):** Replace old values. No history preserved.
```sql
-- Before: trader moved from NYC to London
UPDATE dim_trader SET region = 'London' WHERE trader_id = 123;
-- History lost: all historical trades now show London
```

**Type 2 (Add Row):** Create new row for each change. Full history preserved.
```sql
CREATE TABLE dim_trader_scd2 (
    trader_key INT PRIMARY KEY,      -- Surrogate key
    trader_id INT,                   -- Natural key
    name VARCHAR(100),
    region VARCHAR(50),
    effective_from DATE,
    effective_to DATE,               -- NULL = current
    is_current BOOLEAN
);

-- Trader moved from NYC to London on 2024-06-01
-- Row 1: trader_id=123, region='NYC', effective_to='2024-05-31', is_current=false
-- Row 2: trader_id=123, region='London', effective_from='2024-06-01', is_current=true
```

Fact tables link to the surrogate key, preserving the dimension state at transaction time.

**Type 3 (Add Column):** Track limited history with additional columns.
```sql
CREATE TABLE dim_trader_scd3 (
    trader_key INT PRIMARY KEY,
    trader_id INT,
    current_region VARCHAR(50),
    previous_region VARCHAR(50),
    region_changed_date DATE
);
-- Only one level of history
```

**When to use which:**
- Type 1: Corrections, attributes where history doesn't matter
- Type 2: Full audit trail required, historical analysis needed
- Type 3: Only need previous value, limited history sufficient

### Conformed Dimensions

A **conformed dimension** is shared across multiple fact tables, enabling cross-process analysis.

```
               ┌──────────────┐
               │  dim_date    │  (conformed)
               └──────┬───────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
    ┌─────┴─────┐ ┌───┴───┐ ┌────┴────┐
    │fct_trades │ │fct_pnl│ │fct_fees │
    └───────────┘ └───────┘ └─────────┘
          │           │           │
          └───────────┼───────────┘
                      │
               ┌──────┴───────┐
               │ dim_symbol   │  (conformed)
               └──────────────┘
```

Conformed dimensions enable:
- Consistent filtering across facts
- Drill-across queries (combining facts)
- Single source of truth for entities

### Implementing in dbt

Dimensional models map naturally to dbt's staging/marts pattern:

```
models/
├── staging/
│   ├── stg_trades.sql           # Clean source data
│   └── stg_symbols.sql
├── intermediate/
│   └── int_trades_enriched.sql  # Business logic
└── marts/
    ├── core/
    │   ├── dim_date.sql         # Conformed dimensions
    │   ├── dim_symbol.sql
    │   └── dim_trader.sql
    └── trading/
        ├── fct_trades.sql       # Fact tables
        └── fct_daily_positions.sql
```

**Dimension model:**
```sql
-- models/marts/core/dim_symbol.sql
{{ config(materialized='table') }}

WITH source AS (
    SELECT * FROM {{ ref('stg_symbols') }}
),

with_surrogate_key AS (
    SELECT
        {{ dbt_utils.generate_surrogate_key(['symbol']) }} as symbol_key,
        symbol,
        name,
        exchange,
        base_currency,
        quote_currency,
        asset_class,
        is_active
    FROM source
)

SELECT * FROM with_surrogate_key
```

**Fact model:**
```sql
-- models/marts/trading/fct_trades.sql
{{ config(
    materialized='incremental',
    unique_key='trade_key'
) }}

WITH trades AS (
    SELECT * FROM {{ ref('int_trades_enriched') }}
    {% if is_incremental() %}
    WHERE traded_at > (SELECT MAX(traded_at) FROM {{ this }})
    {% endif %}
),

with_keys AS (
    SELECT
        {{ dbt_utils.generate_surrogate_key(['trade_id']) }} as trade_key,
        d.date_key,
        s.symbol_key,
        t.trader_key,
        trades.quantity,
        trades.price,
        trades.quantity * trades.price as value,
        trades.commission,
        trades.traded_at
    FROM trades
    JOIN {{ ref('dim_date') }} d ON DATE(trades.traded_at) = d.date
    JOIN {{ ref('dim_symbol') }} s ON trades.symbol = s.symbol
    JOIN {{ ref('dim_trader') }} t ON trades.trader_id = t.trader_id
)

SELECT * FROM with_keys
```

## Data Vault

Data Vault, created by Dan Linstedt, takes a different approach: model the business as it actually exists, separate from how you want to report on it.

### Philosophy

Data Vault separates three concerns:

1. **Business entities (Hubs):** The core things you track
2. **Relationships between entities (Links):** How things connect
3. **Descriptive attributes (Satellites):** Context that changes over time

This separation enables:
- Parallel development (teams work on different satellites)
- Full history by default
- Agility when business definitions change
- Auditability (all changes tracked)

### Core Components

**Hubs:** Business keys—the unique identifiers of entities.
```sql
CREATE TABLE hub_trader (
    hub_trader_key BINARY(16),      -- Hash of business key
    trader_id VARCHAR(50),          -- Business key
    load_date TIMESTAMP,
    record_source VARCHAR(100)
);
```

**Links:** Relationships between hubs.
```sql
CREATE TABLE link_trade (
    link_trade_key BINARY(16),      -- Hash of all hub keys
    hub_trader_key BINARY(16),
    hub_symbol_key BINARY(16),
    hub_exchange_key BINARY(16),
    load_date TIMESTAMP,
    record_source VARCHAR(100)
);
```

**Satellites:** Descriptive attributes, fully historized.
```sql
CREATE TABLE sat_trader_details (
    hub_trader_key BINARY(16),
    load_date TIMESTAMP,            -- Effective from
    load_end_date TIMESTAMP,        -- Effective to
    name VARCHAR(100),
    email VARCHAR(200),
    region VARCHAR(50),
    tier VARCHAR(20),
    record_source VARCHAR(100),
    hash_diff BINARY(16)            -- Hash of attributes for change detection
);
```

### Data Vault Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Data Vault Structure                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐                           ┌────────────┐        │
│  │ sat_trader │                           │ sat_symbol │        │
│  │ _details   │                           │ _details   │        │
│  └─────┬──────┘                           └─────┬──────┘        │
│        │                                        │               │
│        ▼                                        ▼               │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐        │
│  │ hub_trader │◄────▶│ link_trade │◄────▶│ hub_symbol │        │
│  └────────────┘      └─────┬──────┘      └────────────┘        │
│                            │                                    │
│                            ▼                                    │
│                      ┌────────────┐                             │
│                      │ sat_trade  │                             │
│                      │ _details   │                             │
│                      └────────────┘                             │
│                                                                  │
│  Key insight: Hubs are stable. Satellites can change.           │
│  New attributes = new satellite. No hub changes.                │
└─────────────────────────────────────────────────────────────────┘
```

### Point-in-Time Tables

Querying Data Vault directly is complex. **Point-in-Time (PIT)** tables simplify access:

```sql
-- pit_trader: snapshot of trader at any point in time
CREATE TABLE pit_trader AS
SELECT
    h.hub_trader_key,
    h.trader_id,
    snap.snapshot_date,
    sd.sat_trader_details_key,
    sc.sat_trader_contacts_key
FROM hub_trader h
CROSS JOIN date_spine snap
LEFT JOIN sat_trader_details sd
    ON h.hub_trader_key = sd.hub_trader_key
    AND snap.snapshot_date >= sd.load_date
    AND snap.snapshot_date < COALESCE(sd.load_end_date, '9999-12-31')
LEFT JOIN sat_trader_contacts sc
    ON h.hub_trader_key = sc.hub_trader_key
    AND snap.snapshot_date >= sc.load_date
    AND snap.snapshot_date < COALESCE(sc.load_end_date, '9999-12-31');
```

### Business Vault

Raw Data Vault captures data as-is. **Business Vault** adds derived logic:

**Computed satellites:** Derived attributes
```sql
-- sat_trader_derived: computed from raw data
CREATE TABLE sat_trader_derived (
    hub_trader_key BINARY(16),
    load_date TIMESTAMP,
    total_trades_30d INT,
    average_trade_size DECIMAL(18, 2),
    trader_tier VARCHAR(20)  -- Derived from rules
);
```

**Bridge tables:** Pre-joined paths for common access patterns
```sql
-- bridge_trader_trades: common join path
CREATE TABLE bridge_trader_trades AS
SELECT
    h.hub_trader_key,
    l.link_trade_key,
    st.load_date as trade_date
FROM hub_trader h
JOIN link_trade l ON h.hub_trader_key = l.hub_trader_key
JOIN sat_trade_details st ON l.link_trade_key = st.link_trade_key;
```

### When to Use Data Vault

**Data Vault excels when:**
- Multiple source systems with overlapping entities
- Frequent schema changes in sources
- Full audit history is required (regulatory, compliance)
- Large teams need to work in parallel
- Business definitions are unstable or contested

**Data Vault challenges:**
- Query complexity (many joins)
- Performance overhead
- Steeper learning curve
- Overkill for simple use cases

**Common pattern:** Data Vault as raw/staging layer, dimensional models for marts.

```
Sources → Data Vault (raw) → Dimensional Models (marts) → BI Tools
            │                        │
            │ Full history           │ Optimized for queries
            │ Audit trail            │ Business-friendly
            │ Flexible               │ Performant
```

## One Big Table (OBT)

The One Big Table approach challenges traditional normalization: what if you just denormalized everything into a single wide table?

### The Concept

```sql
-- One Big Table: everything in one place
CREATE TABLE obt_trading AS
SELECT
    -- Trade facts
    t.trade_id,
    t.quantity,
    t.price,
    t.value,
    t.traded_at,

    -- Date attributes (denormalized)
    DATE(t.traded_at) as trade_date,
    EXTRACT(DOW FROM t.traded_at) as day_of_week,
    EXTRACT(MONTH FROM t.traded_at) as month,
    EXTRACT(QUARTER FROM t.traded_at) as quarter,
    EXTRACT(YEAR FROM t.traded_at) as year,
    CASE WHEN EXTRACT(DOW FROM t.traded_at) IN (0, 6) THEN 'Weekend' ELSE 'Weekday' END as day_type,

    -- Symbol attributes (denormalized)
    s.symbol,
    s.name as symbol_name,
    s.exchange,
    s.base_currency,
    s.quote_currency,
    s.asset_class,

    -- Trader attributes (denormalized)
    tr.trader_id,
    tr.trader_name,
    tr.trader_type,
    tr.region,
    tr.tier,

    -- Pre-computed metrics
    t.value / NULLIF(daily_totals.total_value, 0) as pct_of_daily_volume

FROM trades t
JOIN symbols s ON t.symbol = s.symbol
JOIN traders tr ON t.trader_id = tr.trader_id
JOIN daily_totals ON DATE(t.traded_at) = daily_totals.date;
```

### Why OBT Works (Sometimes)

**Modern warehouses changed the calculus:**

1. **Columnar storage:** Only requested columns are read, so wide tables don't penalize simple queries
2. **Compression:** Repeated values compress well in columnar formats
3. **Compute power:** Modern warehouses handle complex queries efficiently
4. **Storage costs:** Storage is cheap; query complexity is expensive

**OBT benefits:**
- Simple queries (no joins)
- Self-documenting (all context in one place)
- Fast for BI tools (they love flat tables)
- Easy to understand for non-technical users

**OBT drawbacks:**
- Data redundancy (storage, though cheap, isn't free)
- Update anomalies (change a symbol name → update millions of rows)
- No enforced relationships
- Can become unwieldy (hundreds of columns)

### When to Use OBT

**OBT works well for:**
- Final reporting layer (downstream of dimensional model)
- Self-service analytics (non-technical users)
- Specific use cases with clear query patterns
- Prototyping before building proper models

**OBT is problematic for:**
- Core data warehouse layer (use dimensional or Data Vault)
- Frequently changing dimensions
- Very large tables where redundancy matters
- Complex relationships requiring flexibility

### Hybrid Approach

Many organizations use OBT as a final presentation layer:

```
Sources → Dimensional Model → OBT for specific use cases
               │
               └──→ Direct queries for complex analysis
```

```sql
-- models/marts/analytics/obt_trading_dashboard.sql
-- OBT specifically for the trading dashboard

{{ config(materialized='table') }}

SELECT
    f.trade_key,
    f.traded_at,
    f.quantity,
    f.price,
    f.value,

    -- Date dimension flattened
    d.date,
    d.day_of_week,
    d.month_name,
    d.quarter,
    d.year,
    d.is_weekend,

    -- Symbol dimension flattened
    s.symbol,
    s.symbol_name,
    s.exchange,
    s.asset_class,

    -- Trader dimension flattened
    t.trader_name,
    t.trader_type,
    t.region

FROM {{ ref('fct_trades') }} f
JOIN {{ ref('dim_date') }} d ON f.date_key = d.date_key
JOIN {{ ref('dim_symbol') }} s ON f.symbol_key = s.symbol_key
JOIN {{ ref('dim_trader') }} t ON f.trader_key = t.trader_key
```

## Activity Schema

The Activity Schema, popularized by the Narrator framework, takes a different approach: model all business events as activities with consistent structure.

### The Concept

Every business event becomes an "activity" with standard columns:

```sql
CREATE TABLE activities (
    activity_id BIGINT,
    ts TIMESTAMP,                    -- When it happened
    customer VARCHAR(100),           -- Who did it (entity)
    activity VARCHAR(100),           -- What happened
    anonymous_customer_id VARCHAR,   -- Pre-identification tracking
    feature_1 VARCHAR,               -- Generic attribute slots
    feature_2 VARCHAR,
    feature_3 VARCHAR,
    revenue_impact DECIMAL,
    link VARCHAR                     -- Related entity
);
```

**Example activities:**
```
| ts                  | customer | activity          | feature_1 | revenue_impact |
|---------------------|----------|-------------------|-----------|----------------|
| 2024-01-15 10:30:00 | trader_1 | placed_order      | BTC-USD   | null           |
| 2024-01-15 10:30:05 | trader_1 | order_filled      | BTC-USD   | 50000          |
| 2024-01-15 10:31:00 | trader_1 | withdrew_funds    | null      | -10000         |
```

### Benefits

**Uniform structure:** All events have the same shape, simplifying pipelines.

**Customer-centric:** Every activity ties to a customer, enabling cohort analysis.

**Temporal queries:** Time-based analysis is natural (first activity, last activity, sequences).

**Flexibility:** New event types don't require schema changes.

### Trade-offs

**Generic columns:** `feature_1`, `feature_2` lack semantic meaning.

**Query complexity:** Typed analysis requires filtering and casting.

**Not for all use cases:** Better for customer journey analysis than operational reporting.

## Modeling Patterns for Common Scenarios

### Time-Series Data

Trading data is inherently time-series. Key patterns:

**Grain selection:**
- Transaction grain for detailed analysis
- Periodic snapshots for point-in-time queries
- Pre-aggregated summaries for dashboards

**Date dimension design:**
```sql
-- dim_date with trading-specific attributes
CREATE TABLE dim_date (
    date_key INT,
    date DATE,
    -- Standard attributes
    day_of_week VARCHAR(10),
    month INT,
    quarter INT,
    year INT,
    -- Trading-specific
    is_trading_day BOOLEAN,
    is_market_holiday BOOLEAN,
    trading_day_of_month INT,
    trading_week_of_year INT
);
```

**Time dimension for intraday:**
```sql
-- dim_time for sub-day granularity
CREATE TABLE dim_time (
    time_key INT,
    time TIME,
    hour INT,
    minute INT,
    -- Trading-specific
    market_session VARCHAR(20),  -- Pre-market, Regular, After-hours
    is_high_volume_period BOOLEAN
);
```

### Multi-Currency Handling

Financial data often involves multiple currencies:

```sql
-- fct_trades with currency handling
CREATE TABLE fct_trades (
    trade_key BIGINT,
    date_key INT,
    symbol_key INT,

    -- Native currency
    quantity DECIMAL(18, 8),
    price_native DECIMAL(18, 8),
    currency_code VARCHAR(3),

    -- Converted to reporting currency
    price_usd DECIMAL(18, 8),
    fx_rate_to_usd DECIMAL(18, 8),

    -- Point-in-time exchange rate
    fx_rate_date DATE
);
```

**Exchange rate dimension:**
```sql
-- dim_exchange_rate: SCD2 for historical rates
CREATE TABLE dim_exchange_rate (
    rate_key INT,
    from_currency VARCHAR(3),
    to_currency VARCHAR(3),
    rate DECIMAL(18, 8),
    effective_from DATE,
    effective_to DATE
);
```

### Handling Late-Arriving Data

Facts may arrive after related dimensions:

**Inferred members:** Create placeholder dimension records for unknown keys.

```sql
-- When trade arrives before symbol is known
INSERT INTO dim_symbol (symbol_key, symbol, name, is_inferred)
VALUES (hash('UNKNOWN_XYZ'), 'UNKNOWN_XYZ', 'Inferred Member', TRUE);

-- Later, update when symbol data arrives
UPDATE dim_symbol
SET name = 'Actual Name', is_inferred = FALSE
WHERE symbol = 'UNKNOWN_XYZ';
```

**Late-arriving facts:** Use incremental models with lookback windows.

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='delete+insert',
    unique_key=['date_key', 'symbol_key']
) }}

SELECT ...
FROM {{ ref('stg_trades') }}
{% if is_incremental() %}
-- Reprocess last 3 days to catch late arrivals
WHERE date >= DATEADD(day, -3, CURRENT_DATE)
{% endif %}
```

### Hierarchies

Many dimensions have hierarchies:

**Fixed-depth hierarchy (embedded):**
```sql
-- dim_geography with embedded hierarchy
CREATE TABLE dim_geography (
    geography_key INT,
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100),
    region VARCHAR(50),
    continent VARCHAR(50)
);

-- Query any level
SELECT continent, SUM(value)
FROM fct_sales f
JOIN dim_geography g ON f.geography_key = g.geography_key
GROUP BY continent;
```

**Variable-depth hierarchy (bridge table):**
```sql
-- For org charts, product categories, etc.
CREATE TABLE bridge_org_hierarchy (
    parent_employee_key INT,
    child_employee_key INT,
    depth INT,
    path VARCHAR(500)  -- '/CEO/VP/Director/Manager'
);
```

## Metrics Layer

A relatively new concept: define metrics once, use everywhere.

### The Problem

Without a metrics layer:
- Marketing defines "revenue" differently than Finance
- The same metric is calculated inconsistently across dashboards
- Changes require updating multiple places

### The Solution

Define metrics centrally, consume them anywhere:

```yaml
# metrics/revenue.yml
metrics:
  - name: total_revenue
    description: Total revenue from filled trades
    type: sum
    sql: price * quantity
    timestamp: traded_at
    time_grains: [day, week, month]
    dimensions:
      - symbol
      - trader
      - region
    filters:
      - status = 'FILLED'

  - name: revenue_per_trade
    description: Average revenue per trade
    type: derived
    sql: "{{ metric('total_revenue') }} / {{ metric('trade_count') }}"
```

**Tools implementing metrics layers:**
- dbt Semantic Layer (MetricFlow)
- Cube.js
- Looker (LookML)
- Lightdash

### dbt Semantic Layer Example

```yaml
# models/semantic/trading_metrics.yml
semantic_models:
  - name: trading_metrics
    defaults:
      agg_time_dimension: traded_at
    model: ref('fct_trades')

    entities:
      - name: trade
        type: primary
        expr: trade_key
      - name: symbol
        type: foreign
        expr: symbol_key
      - name: trader
        type: foreign
        expr: trader_key

    measures:
      - name: trade_count
        agg: count
        expr: trade_key
      - name: total_volume
        agg: sum
        expr: quantity
      - name: total_value
        agg: sum
        expr: value
      - name: avg_price
        agg: average
        expr: price

    dimensions:
      - name: traded_at
        type: time
        type_params:
          time_granularity: day
      - name: trade_status
        type: categorical

metrics:
  - name: vwap
    type: derived
    type_params:
      expr: total_value / total_volume
```

Query the semantic layer:
```sql
-- Consumers query metrics, not raw tables
SELECT
    {{ metric('vwap') }},
    {{ metric('trade_count') }}
GROUP BY symbol, date_trunc('day', traded_at)
```

## Decision Framework: Choosing a Modeling Approach

### Assessment Questions

1. **What are the primary use cases?**
   - Ad-hoc analysis → Dimensional modeling
   - Compliance/audit → Data Vault
   - Simple dashboards → OBT (as presentation layer)

2. **How stable are business definitions?**
   - Stable → Dimensional modeling
   - Frequently changing → Data Vault

3. **What are the query patterns?**
   - Star joins → Dimensional modeling
   - Point-in-time queries → Data Vault or SCD2 dimensions
   - Simple filters → OBT

4. **Team expertise?**
   - SQL-comfortable → Dimensional modeling
   - Specialized data team → Data Vault
   - Non-technical consumers → OBT

5. **Scale requirements?**
   - Moderate → Any approach works
   - Very large → Consider query performance (OBT may help at presentation layer)

### Recommended Patterns by Maturity

**Early stage (small team, simple needs):**
```
Sources → Staging → Dimensional Marts (star schema)
```

**Growth stage (multiple teams, complex needs):**
```
Sources → Data Vault (raw) → Dimensional Marts → OBT (presentation)
                              ↓
                        Metrics Layer
```

**Enterprise (strict compliance, many sources):**
```
Sources → Data Vault → Business Vault → Dimensional Marts
                              ↓
                      Metrics Layer → BI Tools
```

## Summary

Data modeling is the discipline that turns raw data into analytical assets:

**Dimensional Modeling:**
- Facts (events) + Dimensions (context)
- Star schema preferred over snowflake
- SCD Type 2 for full history
- Conformed dimensions for consistency

**Data Vault:**
- Hubs (keys) + Links (relationships) + Satellites (attributes)
- Full history by default
- Agile and auditable
- Complex queries require Business Vault layer

**One Big Table:**
- Denormalized for simplicity
- Works well as presentation layer
- Not suitable for core warehouse

**Activity Schema:**
- Uniform event structure
- Customer journey focus
- Good for behavioral analytics

**Metrics Layer:**
- Single source of truth for calculations
- Consistent across tools
- Growing ecosystem support

**Key insight:** There's no single "right" model. The best approach combines patterns:
- Data Vault for raw ingestion (auditability)
- Dimensional models for marts (query optimization)
- OBT for specific use cases (simplicity)
- Metrics layer for consistency (governance)

**Looking ahead:**
- Chapter 14 covers data quality, which validates your models work correctly
- Chapter 15 shows how these models fit into larger architectures

## Further Reading

- Kimball, R. & Ross, M. (2013). *The Data Warehouse Toolkit, 3rd Edition* — Definitive guide to dimensional modeling
- Linstedt, D. & Olschimke, M. (2015). *Building a Scalable Data Warehouse with Data Vault 2.0* — Data Vault methodology
- [dbt Semantic Layer Documentation](https://docs.getdbt.com/docs/build/about-metricflow) — MetricFlow implementation
- [Activity Schema (Narrator)](https://www.narrator.ai/blog/activity-schema/) — Activity schema pattern
- Inmon, W.H. (2005). *Building the Data Warehouse, 4th Edition* — Enterprise data warehouse architecture
- [Kimball Group Design Tips](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) — Practical modeling guidance
