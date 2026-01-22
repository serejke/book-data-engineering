---
chapter: 14
title: "Data Quality and Observability"
estimated_pages: 30-35
status: draft
last_updated: 2025-01-22
---

# Data Quality and Observability

Your pipelines run. Your models build. Your dashboards render. But are the numbers right?

Data quality is the silent killer of data platforms. A pipeline that fails loudly is annoying but manageable. A pipeline that succeeds while producing subtly wrong data is catastrophic. Decisions get made on bad numbers. Trust erodes. By the time someone notices the CEO's report doesn't match Finance's spreadsheet, the damage is done.

This chapter addresses both data quality (ensuring data is correct) and data observability (understanding what's happening in your data systems). These concepts are intertwined: you can't ensure quality without visibility, and observability without quality standards is just watching things go wrong.

## The Data Quality Problem

### Why Quality Is Hard

Data quality seems straightforward: data should be correct. But "correct" is surprisingly complex:

**Correctness depends on context.** Is a NULL value wrong, or does it mean "unknown"? Is a trade with negative quantity an error, or a valid short sale?

**Quality degrades silently.** A schema change in a source system doesn't break your pipeline—it just starts producing wrong results. No errors, no alerts, just bad data flowing downstream.

**Problems compound.** One bad record in staging becomes thousands of bad aggregations. By the time quality issues surface in dashboards, root causes are buried under layers of transformations.

**Quality is a spectrum.** Perfectly clean data is often impossible and always expensive. The question isn't "is data perfect?" but "is data good enough for this use case?"

### The Cost of Poor Quality

Poor data quality imposes real costs:

**Direct business impact:**
- Wrong decisions based on incorrect metrics
- Customer complaints from bad recommendations
- Regulatory fines from compliance failures
- Operational inefficiencies from bad forecasts

**Engineering costs:**
- Debugging time investigating data issues
- Rework when bad data must be corrected
- Technical debt from workarounds and patches
- Trust deficit requiring manual verification

**Organizational costs:**
- Analysts maintaining shadow spreadsheets
- Stakeholders questioning all numbers
- Data team credibility erosion
- Resistance to data-driven initiatives

A study by IBM estimated poor data quality costs organizations $3.1 trillion annually in the US alone. Even if your costs are a tiny fraction of that, they're real and preventable.

## Data Quality Dimensions

Quality isn't a single property. It has multiple dimensions, each requiring different approaches.

### Completeness

**Definition:** Is all expected data present?

**What to check:**
- Row counts match expectations
- Required fields are populated
- All expected partitions exist
- No gaps in time series

**Examples:**
```sql
-- Check for missing dates
SELECT d.date
FROM dim_date d
LEFT JOIN fct_trades t ON d.date = t.trade_date
WHERE d.date BETWEEN '2024-01-01' AND CURRENT_DATE
  AND d.is_trading_day = TRUE
  AND t.trade_date IS NULL;

-- Check for NULL rates in required fields
SELECT COUNT(*) as null_count
FROM fct_trades
WHERE price IS NULL OR quantity IS NULL;
```

### Accuracy

**Definition:** Does data reflect reality?

**What to check:**
- Values within expected ranges
- Calculations produce correct results
- Data matches authoritative sources
- Aggregations reconcile with source systems

**Examples:**
```sql
-- Check prices are within reasonable range
SELECT *
FROM fct_trades
WHERE price <= 0 OR price > 1000000;

-- Reconcile with source
SELECT
    COUNT(*) as our_count,
    (SELECT COUNT(*) FROM source_system.trades WHERE date = '2024-01-15') as source_count
FROM fct_trades
WHERE trade_date = '2024-01-15';
```

### Consistency

**Definition:** Is data coherent across the system?

**What to check:**
- Same entity has same attributes everywhere
- Calculations match across reports
- Historical data doesn't change unexpectedly
- Related tables agree

**Examples:**
```sql
-- Check referential integrity
SELECT f.symbol
FROM fct_trades f
LEFT JOIN dim_symbol s ON f.symbol = s.symbol
WHERE s.symbol IS NULL;

-- Check aggregations match
SELECT
    SUM(t.quantity) as transaction_total,
    d.daily_volume
FROM fct_trades t
JOIN fct_daily_summary d ON t.trade_date = d.date AND t.symbol = d.symbol
GROUP BY d.daily_volume
HAVING SUM(t.quantity) != d.daily_volume;
```

### Timeliness

**Definition:** Is data available when needed?

**What to check:**
- Data arrives within SLA
- Processing completes on schedule
- Freshness meets requirements
- No unexpected delays

**Examples:**
```sql
-- Check data freshness
SELECT
    MAX(loaded_at) as last_load,
    CURRENT_TIMESTAMP - MAX(loaded_at) as lag
FROM stg_trades;

-- Alert if lag exceeds threshold
SELECT CASE
    WHEN MAX(loaded_at) < CURRENT_TIMESTAMP - INTERVAL '2 hours'
    THEN 'STALE'
    ELSE 'FRESH'
END as freshness_status
FROM stg_trades;
```

### Validity

**Definition:** Does data conform to expected formats and rules?

**What to check:**
- Data types are correct
- Values match expected patterns
- Business rules are satisfied
- Constraints are respected

**Examples:**
```sql
-- Check email format
SELECT *
FROM dim_trader
WHERE email NOT LIKE '%@%.%';

-- Check business rule: quantity must be positive for buys
SELECT *
FROM fct_trades
WHERE side = 'BUY' AND quantity <= 0;

-- Check enumerated values
SELECT DISTINCT status
FROM fct_trades
WHERE status NOT IN ('PENDING', 'FILLED', 'CANCELLED', 'REJECTED');
```

### Uniqueness

**Definition:** No unintended duplicates exist.

**What to check:**
- Primary keys are unique
- Business keys don't have duplicates
- Deduplication is working

**Examples:**
```sql
-- Check for duplicate trade IDs
SELECT trade_id, COUNT(*)
FROM fct_trades
GROUP BY trade_id
HAVING COUNT(*) > 1;

-- Check for duplicate natural keys
SELECT trader_id, effective_from, COUNT(*)
FROM dim_trader
GROUP BY trader_id, effective_from
HAVING COUNT(*) > 1;
```

## Testing Strategies

### Test Pyramid for Data

Apply the testing pyramid concept to data quality:

```
           /\
          /  \     Reconciliation Tests
         /    \    (Cross-system validation)
        /──────\
       /        \  Integration Tests
      /          \ (Cross-table consistency)
     /────────────\
    /              \ Unit Tests
   /                \ (Column-level: not_null, unique, range)
```

**Unit tests:** Fast, focused, run frequently
**Integration tests:** Cross-table, catch relationship issues
**Reconciliation tests:** Compare with source systems, catch systemic problems

### dbt Tests

dbt provides a natural framework for data testing (see Chapter 12):

**Schema tests (YAML):**
```yaml
models:
  - name: fct_trades
    columns:
      - name: trade_id
        tests:
          - unique
          - not_null

      - name: price
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true

      - name: symbol
        tests:
          - relationships:
              to: ref('dim_symbol')
              field: symbol
```

**Custom tests:**
```sql
-- tests/assert_daily_volume_positive.sql
SELECT date, symbol, total_volume
FROM {{ ref('fct_daily_summary') }}
WHERE total_volume <= 0
```

**Test configuration:**
```yaml
# Run critical tests on every build
models:
  - name: fct_trades
    config:
      tags: ['critical']
    tests:
      - unique:
          column_name: trade_id
          config:
            severity: error  # Fail the build
      - dbt_utils.recency:
          datepart: hour
          field: loaded_at
          interval: 2
          config:
            severity: warn  # Warn but don't fail
```

### Great Expectations

Great Expectations provides a comprehensive testing framework:

```python
import great_expectations as gx

# Create expectation suite
suite = context.add_expectation_suite("trading_data_suite")

# Define expectations
validator.expect_column_values_to_be_unique("trade_id")
validator.expect_column_values_to_not_be_null("price")
validator.expect_column_values_to_be_between("price", min_value=0, strict_min=True)
validator.expect_column_values_to_be_in_set("status", ["PENDING", "FILLED", "CANCELLED"])
validator.expect_table_row_count_to_be_between(min_value=1000, max_value=10000000)

# Run validation
results = validator.validate()

if not results.success:
    raise DataQualityError(f"Validation failed: {results}")
```

**Integration with pipelines:**
```python
# Airflow integration
from great_expectations_provider.operators.great_expectations import (
    GreatExpectationsOperator
)

validate_trades = GreatExpectationsOperator(
    task_id="validate_trades",
    data_context_root_dir="/path/to/gx",
    dataframe_to_validate=trades_df,
    expectation_suite_name="trading_data_suite",
)
```

### Statistical Testing

Beyond rule-based tests, detect anomalies statistically:

**Distribution checks:**
```sql
-- Check if today's distribution is unusual
WITH daily_stats AS (
    SELECT
        trade_date,
        AVG(price) as avg_price,
        STDDEV(price) as stddev_price
    FROM fct_trades
    WHERE trade_date >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY trade_date
),
historical AS (
    SELECT
        AVG(avg_price) as historical_avg,
        STDDEV(avg_price) as historical_stddev
    FROM daily_stats
    WHERE trade_date < CURRENT_DATE
)
SELECT
    d.trade_date,
    d.avg_price,
    h.historical_avg,
    ABS(d.avg_price - h.historical_avg) / h.historical_stddev as z_score
FROM daily_stats d
CROSS JOIN historical h
WHERE d.trade_date = CURRENT_DATE
  AND ABS(d.avg_price - h.historical_avg) / h.historical_stddev > 3;
```

**Volume anomaly detection:**
```python
from scipy import stats

def detect_volume_anomaly(daily_volumes, threshold=0.01):
    """Detect if latest volume is statistically unusual."""
    historical = daily_volumes[:-1]
    latest = daily_volumes[-1]

    # Z-score test
    z_score = (latest - historical.mean()) / historical.std()

    # More sophisticated: use Grubbs test
    grubbs_stat, p_value = stats.grubbs(daily_volumes)

    return p_value < threshold, z_score, p_value
```

## Observability

Data observability extends beyond testing to continuous monitoring—understanding what's happening in your data systems at all times.

### The Five Pillars of Data Observability

**1. Freshness:** Is data up to date?
```sql
-- Track freshness over time
INSERT INTO data_freshness_log (table_name, max_timestamp, checked_at)
SELECT
    'fct_trades',
    MAX(loaded_at),
    CURRENT_TIMESTAMP
FROM fct_trades;
```

**2. Volume:** Are we getting the expected amount of data?
```sql
-- Track daily row counts
INSERT INTO volume_log (table_name, partition_date, row_count, logged_at)
SELECT
    'fct_trades',
    trade_date,
    COUNT(*),
    CURRENT_TIMESTAMP
FROM fct_trades
WHERE trade_date = CURRENT_DATE - 1
GROUP BY trade_date;
```

**3. Schema:** Has the structure changed unexpectedly?
```sql
-- Track schema changes
SELECT
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_name = 'fct_trades'
ORDER BY ordinal_position;

-- Compare with baseline and alert on changes
```

**4. Distribution:** Are values distributed as expected?
```sql
-- Track distribution statistics
INSERT INTO distribution_log (
    table_name, column_name, date,
    min_val, max_val, avg_val, null_pct, distinct_count
)
SELECT
    'fct_trades', 'price', CURRENT_DATE,
    MIN(price), MAX(price), AVG(price),
    SUM(CASE WHEN price IS NULL THEN 1 ELSE 0 END)::FLOAT / COUNT(*),
    COUNT(DISTINCT price)
FROM fct_trades
WHERE trade_date = CURRENT_DATE - 1;
```

**5. Lineage:** Where did data come from and where does it go?
```
source.trades ──▶ stg_trades ──▶ int_trades_enriched ──▶ fct_trades
                                        │
                                        └──▶ fct_daily_summary
```

### Observability Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   Data Observability Stack                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Sources   │  │  Pipelines  │  │  Warehouse  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                          │
│              │   Metadata Collector   │                          │
│              │   - Schema snapshots   │                          │
│              │   - Volume metrics     │                          │
│              │   - Freshness checks   │                          │
│              │   - Query logs         │                          │
│              └───────────┬───────────┘                          │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                          │
│              │   Observability Store  │                          │
│              │   (Time-series DB)     │                          │
│              └───────────┬───────────┘                          │
│                          │                                      │
│         ┌────────────────┼────────────────┐                     │
│         │                │                │                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Dashboards │  │   Alerts    │  │  Lineage    │             │
│  │             │  │             │  │   Graph     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementing Observability

**Metadata collection:**
```python
# Collect table metadata
def collect_table_metadata(table_name: str) -> dict:
    return {
        "table": table_name,
        "timestamp": datetime.utcnow(),
        "row_count": get_row_count(table_name),
        "freshness": get_max_timestamp(table_name),
        "schema_hash": hash_schema(table_name),
        "column_stats": get_column_statistics(table_name),
    }

# Run on schedule
@dag(schedule="0 * * * *")  # Hourly
def collect_metadata():
    for table in MONITORED_TABLES:
        metadata = collect_table_metadata(table)
        store_metadata(metadata)
        check_anomalies(metadata)
```

**Anomaly detection:**
```python
def check_anomalies(current: dict, historical: list[dict]) -> list[Alert]:
    alerts = []

    # Volume anomaly
    volumes = [h["row_count"] for h in historical]
    if is_anomaly(current["row_count"], volumes):
        alerts.append(Alert(
            type="VOLUME_ANOMALY",
            table=current["table"],
            message=f"Row count {current['row_count']} is unusual"
        ))

    # Freshness anomaly
    if current["freshness"] < datetime.utcnow() - timedelta(hours=2):
        alerts.append(Alert(
            type="STALE_DATA",
            table=current["table"],
            message=f"Data is {datetime.utcnow() - current['freshness']} old"
        ))

    # Schema change
    last_hash = historical[-1]["schema_hash"] if historical else None
    if last_hash and current["schema_hash"] != last_hash:
        alerts.append(Alert(
            type="SCHEMA_CHANGE",
            table=current["table"],
            message="Schema changed unexpectedly"
        ))

    return alerts
```

### Lineage Tracking

**Column-level lineage:**
```
fct_trades.value = stg_trades.price × stg_trades.quantity
                   ↓                    ↓
                   source.trades.price  source.trades.qty

fct_daily_summary.total_value = SUM(fct_trades.value)
```

**dbt lineage:**
dbt automatically tracks model-level lineage through `ref()` and `source()`. For column-level lineage, use dbt's metadata:

```yaml
# Document column origins
models:
  - name: fct_trades
    columns:
      - name: value
        description: "Trade value = price × quantity"
        meta:
          origin_columns:
            - stg_trades.price
            - stg_trades.quantity
```

**Lineage tools:**
- **dbt:** Model-level lineage built-in
- **DataHub:** Open-source metadata platform with lineage
- **Atlan:** Data catalog with automated lineage
- **Monte Carlo:** Data observability with lineage

### Alerting Strategy

Not all quality issues require the same response:

**Critical (page immediately):**
- Production data completely missing
- Key metrics show impossible values
- Schema changes that break downstream

**High (notify team channel):**
- Data significantly delayed
- Volume anomalies beyond threshold
- Failed quality tests in marts

**Medium (daily digest):**
- Minor freshness delays
- Small volume variations
- Warnings in staging tables

**Low (weekly review):**
- Documentation missing
- Minor schema drift
- Edge case test failures

**Alert configuration:**
```yaml
# Alert rules
alerts:
  - name: critical_data_missing
    condition: "row_count = 0"
    tables: ["fct_trades", "fct_positions"]
    severity: critical
    channels: ["pagerduty", "slack-critical"]

  - name: data_stale
    condition: "freshness_hours > 2"
    tables: ["fct_*"]
    severity: high
    channels: ["slack-data-team"]

  - name: volume_anomaly
    condition: "abs(volume_z_score) > 3"
    tables: ["*"]
    severity: medium
    channels: ["slack-data-team"]
```

## Data Contracts

Data contracts formalize agreements between data producers and consumers.

### The Problem Contracts Solve

Without contracts:
- Upstream teams change schemas without warning
- Downstream teams build brittle dependencies
- No one knows who depends on what
- Breaking changes cause cascading failures

### Contract Structure

A data contract specifies:

**Schema:** What fields exist, their types, and constraints
**Semantics:** What fields mean and how they're calculated
**SLAs:** Freshness, availability, quality guarantees
**Ownership:** Who maintains the data, who to contact

**Example contract:**
```yaml
# contracts/trading/fct_trades.yaml
contract:
  name: fct_trades
  version: 2.1.0
  owner: trading-data-team
  contact: trading-data@company.com

schema:
  - name: trade_id
    type: STRING
    description: Unique trade identifier
    constraints:
      - not_null
      - unique

  - name: price
    type: DECIMAL(18, 8)
    description: Execution price in quote currency
    constraints:
      - not_null
      - positive

  - name: quantity
    type: DECIMAL(18, 8)
    description: Trade quantity
    constraints:
      - not_null

  - name: traded_at
    type: TIMESTAMP
    description: Trade execution time (UTC)
    constraints:
      - not_null

semantics:
  grain: One row per trade execution
  update_pattern: Append-only (trades are immutable)
  business_rules:
    - "Only FILLED trades are included"
    - "Prices are in quote currency of the trading pair"

sla:
  freshness:
    target: 15 minutes
    max: 1 hour
  availability:
    target: 99.9%
  quality:
    completeness: 99.99%
    accuracy: 99.9%

consumers:
  - team: analytics
    use_case: Daily trading reports
  - team: risk
    use_case: Real-time position monitoring
    sla_tier: critical

changelog:
  - version: 2.1.0
    date: 2024-06-01
    changes:
      - Added commission field
      - Deprecated old_price field (removed in 3.0)
  - version: 2.0.0
    date: 2024-01-01
    changes:
      - Changed price from FLOAT to DECIMAL
      - BREAKING: Removed legacy_id field
```

### Enforcing Contracts

**Schema validation:**
```python
def validate_schema(df: DataFrame, contract: Contract) -> list[Violation]:
    violations = []

    for field in contract.schema:
        if field.name not in df.columns:
            violations.append(Violation(f"Missing field: {field.name}"))

        actual_type = df.schema[field.name].dataType
        if not types_compatible(actual_type, field.type):
            violations.append(Violation(
                f"Type mismatch for {field.name}: expected {field.type}, got {actual_type}"
            ))

    return violations
```

**Contract testing in CI:**
```yaml
# GitHub Actions
- name: Validate data contracts
  run: |
    python validate_contracts.py \
      --contracts contracts/ \
      --environment staging
```

**Breaking change detection:**
```python
def check_breaking_changes(old: Contract, new: Contract) -> list[BreakingChange]:
    changes = []

    for field in old.schema:
        if field.name not in [f.name for f in new.schema]:
            changes.append(BreakingChange(
                type="FIELD_REMOVED",
                field=field.name,
                severity="breaking"
            ))

        new_field = get_field(new, field.name)
        if new_field and not types_compatible(field.type, new_field.type):
            changes.append(BreakingChange(
                type="TYPE_CHANGED",
                field=field.name,
                old_type=field.type,
                new_type=new_field.type,
                severity="breaking"
            ))

    return changes
```

## Incident Response

When quality issues occur, respond systematically.

### Incident Classification

**Severity levels:**

| Level | Impact | Response Time | Examples |
|-------|--------|---------------|----------|
| SEV1 | Production down, major business impact | 15 min | All dashboards showing wrong data |
| SEV2 | Significant degradation | 1 hour | Key metrics incorrect |
| SEV3 | Minor impact | 4 hours | Single report affected |
| SEV4 | Minimal impact | 1 day | Edge case data issue |

### Response Process

**1. Detection:**
- Automated alerts
- User reports
- Routine checks

**2. Triage:**
- Assess severity
- Identify affected systems
- Assign owner

**3. Investigation:**
- Review recent changes
- Check upstream sources
- Examine logs and lineage

**4. Mitigation:**
- Stop bad data propagation
- Communicate to stakeholders
- Implement temporary fix

**5. Resolution:**
- Fix root cause
- Backfill affected data
- Verify fix

**6. Post-mortem:**
- Document timeline
- Identify contributing factors
- Define preventive measures

### Runbook Example

```markdown
# Data Quality Incident Runbook: Missing Trade Data

## Symptoms
- fct_trades shows 0 rows for current date
- Downstream aggregations are empty
- Dashboard shows "No data available"

## Immediate Actions
1. Check source freshness: `SELECT MAX(loaded_at) FROM stg_trades`
2. Check pipeline status in Airflow/Dagster
3. Check source system status

## Investigation Steps
1. Is data present in source?
   ```sql
   SELECT COUNT(*) FROM source_db.trades WHERE date = CURRENT_DATE
   ```

2. Is data in staging?
   ```sql
   SELECT COUNT(*) FROM stg_trades WHERE trade_date = CURRENT_DATE
   ```

3. Did transformation fail?
   Check dbt logs for errors

## Resolution Steps
1. If source issue: Contact source team, wait for fix
2. If pipeline issue: Restart failed task
3. If transformation issue: Fix and rerun
4. Backfill affected partitions:
   ```bash
   dbt run --select fct_trades --vars '{"date": "2024-01-15"}'
   ```

## Communication Template
> We've identified missing trade data for [DATE]. The root cause is [CAUSE].
> We expect resolution by [TIME]. Affected reports: [LIST].
> Updates will be posted to #data-incidents.
```

## Building a Quality Culture

Technology alone doesn't ensure quality. Culture matters.

### Ownership Model

**Clear ownership:**
- Every table has an owner
- Owners are responsible for quality
- Consumers know who to contact

**Ownership documentation:**
```yaml
# OWNERS.yaml
tables:
  fct_trades:
    owner: trading-data-team
    contact: alice@company.com
    slack: "#trading-data"

  dim_symbol:
    owner: reference-data-team
    contact: bob@company.com
    slack: "#ref-data"
```

### Quality Metrics

Track and report on quality:

**Data quality score:**
```sql
-- Calculate overall quality score
SELECT
    table_name,
    (
        completeness_pct * 0.3 +
        accuracy_pct * 0.3 +
        timeliness_pct * 0.2 +
        validity_pct * 0.2
    ) as quality_score
FROM quality_metrics
WHERE date = CURRENT_DATE;
```

**Quality dashboard:**
- Overall quality score trend
- Quality by domain/team
- Top quality issues
- Test pass/fail rates
- Incident frequency

### Continuous Improvement

**Regular reviews:**
- Weekly: Review open quality issues
- Monthly: Analyze quality trends
- Quarterly: Assess quality investments

**Feedback loops:**
- Make it easy to report issues
- Track issues to resolution
- Share learnings across teams

## Summary

Data quality and observability are prerequisites for trustworthy data:

**Quality dimensions:**
- Completeness, accuracy, consistency
- Timeliness, validity, uniqueness
- Each requires different testing approaches

**Testing strategies:**
- Unit tests: Column-level constraints
- Integration tests: Cross-table consistency
- Reconciliation: Source system comparison
- Statistical: Anomaly detection

**Observability pillars:**
- Freshness: Is data current?
- Volume: Is data complete?
- Schema: Is structure stable?
- Distribution: Are values normal?
- Lineage: Where did data come from?

**Data contracts:**
- Formalize producer-consumer agreements
- Define schema, semantics, SLAs
- Enable breaking change detection
- Build trust between teams

**Incident response:**
- Classify by severity
- Follow systematic process
- Document and learn

**Key insight:** Quality isn't a feature you add at the end—it's a discipline woven throughout your data platform. Testing catches issues early. Observability detects issues in production. Contracts prevent issues by setting expectations. Together, they build the trust that makes data valuable.

**Looking ahead:**
- Chapter 15 shows how quality fits into overall architecture
- Chapter 12 covered dbt testing in detail
- Chapter 10 discussed orchestration monitoring

## Further Reading

- [Great Expectations Documentation](https://docs.greatexpectations.io/) — Comprehensive testing framework
- [dbt Testing](https://docs.getdbt.com/docs/build/tests) — dbt's testing capabilities
- [Monte Carlo Data Observability](https://www.montecarlodata.com/blog-what-is-data-observability/) — Observability concepts
- [Data Contracts](https://datacontract.com/) — Data contract specification
- Redman, T. (2001). *Data Quality: The Field Guide* — Foundational text on data quality
- [soda.io](https://www.soda.io/) — Data quality testing tool
- [Elementary](https://www.elementary-data.com/) — dbt-native observability
