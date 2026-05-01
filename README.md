# GA4 BigQuery Query Skill

An [Agent Skill](https://agentskills.io/) that gives AI agents the ability to write correct, performant, and cost-efficient BigQuery SQL for Google Analytics 4 data.

## What's Included

The skill covers:

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
```

## Installation

Copy the `ga4-bigquery-query/` folder into a supported skill location for your agent host:

### Project-level (shared with your team via source control)

Copy into any of these directories at your repository root:

```
.github/skills/ga4-bigquery-query/
.agents/skills/ga4-bigquery-query/
.claude/skills/ga4-bigquery-query/
```

Example:

```bash
cp -r ga4-bigquery-query/ /path/to/your-project/.github/skills/ga4-bigquery-query/
```

### Personal (available across all your workspaces)

Copy into any of these directories in your home folder:

```
~/.copilot/skills/ga4-bigquery-query/
~/.agents/skills/ga4-bigquery-query/
~/.claude/skills/ga4-bigquery-query/
```

Example:

```bash
cp -r ga4-bigquery-query/ ~/.copilot/skills/ga4-bigquery-query/
```

## Usage

Once installed, ask your agent GA4 BigQuery questions such as:

```
Write a query to get sessions by source/medium for the last 30 days
```

For runnable SQL, the agent needs your BigQuery project ID and either:

- your GA4 dataset name, or
- your GA4 property ID

If only the property ID is known, the dataset is usually `analytics_<property_id>`. Documentation examples use `{project}` and `{dataset}` as placeholders, but the agent should replace them automatically when enough context is available and ask for the missing identifier if it is not.

## How It Works

Agent Skills use progressive loading:

1. **Discovery** — The agent reads the skill name and description from SKILL.md frontmatter
2. **Instructions** — When relevant, the agent loads the SKILL.md body (~200 lines of core patterns)
3. **References** — As the agent works, it loads specific reference files on demand (e.g., only the ecommerce reference when you ask about revenue queries)

This means the skill stays efficient — it doesn't flood the context window with all 11 reference files at once.

## Compatibility

This skill follows the open [Agent Skills standard](https://agentskills.io/) and works with:

- GitHub Copilot
- GitHub Copilot CLI
- GitHub Copilot cloud agent
- Any agent that supports the agentskills.io specification
