# GA4 BigQuery Skills

A collection of [Agent Skills](https://agentskills.io/) that give AI agents the ability to write correct, performant, and cost-efficient BigQuery SQL for Google Analytics 4 data — from everyday reporting queries to ML-ready dataset creation.

## Skills

### `ga4-bigquery-query` — Reporting & Analysis

Covers everything needed to query GA4 event data in BigQuery:

- **Schema & table structure** — GA4 export format, all top-level fields, nested/repeated record patterns
- **Event parameter extraction** — 3 UNNEST patterns, value type reference, user property propagation
- **Users & sessions** — Session key construction, all core user/session metric calculations
- **Traffic sources (all 4 scopes)** — `traffic_source`, `session_traffic_source_last_click`, `collected_traffic_source`, event params — plus full default channel grouping CASE logic
- **Ecommerce** — Transaction & item queries, funnel analysis, market basket analysis, revenue by source
- **Page, event & date/time dimensions** — Landing/exit page, page path levels, date formatting, device/geo
- **Attribution models** — Last-touch, first-touch, linear, position-based (40-20-40), time decay (7-day half-life)
- **Advanced patterns** — Cohort revenue, path analysis, checkout abandonment recovery, retention analysis
- **Cost optimization** — `_table_suffix` patterns, column selection, materialization strategy
- **Privacy & consent** — Consent mode impact on data quality, consent-aware query templates
- **BigQuery SQL tips** — SAFE functions, MAX_BY, QUALIFY, PIVOT, ARRAY_AGG, named windows, REGEXP

### `ga4-bigquery-ml-query` — ML Dataset Creation

Covers building ML-ready datasets from GA4 data in BigQuery:

- **Dataset architecture** — Lookback/lookahead temporal windows, entity pools, snapshot dates, leakage prevention
- **Feature engineering** — Cumulative metrics, recency signals, site content features, traffic source and geo encoding
- **Training vs inference modes** — Stored procedure pattern with `MODE` parameter, deterministic train/val/test splits
- **GA4-specific examples** — Full propensity and predictive LTV query templates with 5-layer CTE architecture

## Skill Structure

```
ga4-bigquery-query/
├── SKILL.md                              # Main skill file (loaded by the agent)
└── references/                           # Detailed reference files (loaded on demand)
    ├── schema-and-tables.md
    ├── unnesting-patterns.md
    ├── users-and-sessions.md
    ├── traffic-sources.md
    ├── ecommerce.md
    ├── page-event-dimensions.md
    ├── attribution-models.md
    ├── advanced-patterns.md
    ├── cost-optimization.md
    ├── privacy-and-consent.md
    └── sql-tips.md

ga4-bigquery-ml-query/
├── SKILL.md                              # Main skill file (loaded by the agent)
└── references/                           # Detailed reference files (loaded on demand)
    ├── dataset-architecture.md
    ├── feature-engineering.md
    ├── training-and-inference.md
    └── ga4-examples.md
```

## Installation

Copy one or both skill folders into a supported skill location for your agent host. Install only the skills you need.

### Project-level (shared with your team via source control)

Copy into any of these directories at your repository root:

```
.github/skills/ga4-bigquery-query/
.github/skills/ga4-bigquery-ml-query/
.agents/skills/
.claude/skills/
```

Example:

```bash
cp -r ga4-bigquery-query/ /path/to/your-project/.github/skills/ga4-bigquery-query/
cp -r ga4-bigquery-ml-query/ /path/to/your-project/.github/skills/ga4-bigquery-ml-query/
```

### Personal (available across all your workspaces)

Copy into any of these directories in your home folder:

```
~/.copilot/skills/
~/.agents/skills/
~/.claude/skills/
```

Example:

```bash
cp -r ga4-bigquery-query/ ~/.copilot/skills/ga4-bigquery-query/
cp -r ga4-bigquery-ml-query/ ~/.copilot/skills/ga4-bigquery-ml-query/
```

## Usage

### Reporting queries (`ga4-bigquery-query`)

Once installed, ask your agent GA4 BigQuery questions such as:

```
Write a query to get sessions by source/medium for the last 30 days
```

For runnable SQL, the agent needs your BigQuery project ID and either:

- your GA4 dataset name, or
- your GA4 property ID

If only the property ID is known, the dataset is usually `analytics_<property_id>`. Documentation examples use `{project}` and `{dataset}` as placeholders, but the agent should replace them automatically when enough context is available and ask for the missing identifier if it is not.

### ML dataset creation (`ga4-bigquery-ml-query`)

Ask your agent to build ML-ready datasets, for example:

```
Build a propensity-to-purchase dataset using GA4 data with a 30-day lookback and 14-day label window
```

Before generating a query, the agent will confirm your ML objective, observation grain, label definition, and dataset identifiers. The output is a BigQuery stored procedure that supports both `TRAINING` and `INFERENCE` modes.

## How It Works

Agent Skills use progressive loading:

1. **Discovery** — The agent reads the skill name and description from SKILL.md frontmatter
2. **Instructions** — When relevant, the agent loads the SKILL.md body (~200 lines of core patterns)
3. **References** — As the agent works, it loads specific reference files on demand (e.g., only the ecommerce reference when you ask about revenue queries)

This means the skills stay efficient — they don't flood the context window with all reference files at once.

## Compatibility

This skill follows the open [Agent Skills standard](https://agentskills.io/).
