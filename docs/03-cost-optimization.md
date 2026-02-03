# Cost Optimization Guide for Claude Code

> **Last Updated**: 2026-02-03

This guide explains how to use OTEL metrics to analyze, monitor, and optimize Claude Code costs effectively.

## Table of Contents

- [Understanding Cost Metrics](#understanding-cost-metrics)
- [Setting Up Cost Monitoring](#setting-up-cost-monitoring)
- [Cost Analysis Techniques](#cost-analysis-techniques)
- [Optimization Strategies](#optimization-strategies)
- [Alerting on Cost Anomalies](#alerting-on-cost-anomalies)
- [Reporting and Budgeting](#reporting-and-budgeting)

## Understanding Cost Metrics

### Available Metrics

Claude Code exports these cost-related metrics:

| Metric | Description | Unit |
|--------|-------------|------|
| `claude_code.tokens.input` | Input tokens consumed | tokens |
| `claude_code.tokens.output` | Output tokens generated | tokens |
| `claude_code.cost.total` | Total estimated cost | USD |
| `claude_code.session.count` | Number of sessions | count |
| `claude_code.api.requests` | API calls made | count |

### Token Pricing Reference (2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Claude Opus 4.5 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |

## Setting Up Cost Monitoring

### Environment Variables

```bash
# Enable telemetry with session tracking
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Add cost center and team labels
export OTEL_RESOURCE_ATTRIBUTES="team=engineering,cost_center=CC-1234,environment=production"
```

### OTEL Collector Configuration for Cost Metrics

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  # Add computed cost attributes
  transform/cost:
    metric_statements:
      - context: datapoint
        statements:
          # Calculate cost based on token counts
          - set(attributes["estimated_cost_usd"],
              (attributes["input_tokens"] * 0.000003) +
              (attributes["output_tokens"] * 0.000015))

  # Filter to keep only cost-relevant metrics
  filter/cost_metrics:
    metrics:
      include:
        match_type: regexp
        metric_names:
          - claude_code\.tokens.*
          - claude_code\.cost.*
          - claude_code\.session.*

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [transform/cost, filter/cost_metrics]
      exporters: [prometheusremotewrite]
```

### Prometheus Recording Rules

Create recording rules for efficient cost queries:

```yaml
# prometheus-rules.yaml
groups:
  - name: claude_code_cost
    interval: 1m
    rules:
      # Hourly cost rate
      - record: claude_code:cost:rate1h
        expr: |
          sum(increase(claude_code_cost_total[1h])) by (team, environment)

      # Daily cost
      - record: claude_code:cost:daily
        expr: |
          sum(increase(claude_code_cost_total[24h])) by (team, environment)

      # Token usage rate (per minute)
      - record: claude_code:tokens:rate1m
        expr: |
          sum(rate(claude_code_tokens_input[1m]) + rate(claude_code_tokens_output[1m])) by (team)

      # Cost per session
      - record: claude_code:cost:per_session
        expr: |
          sum(claude_code_cost_total) by (team)
          /
          sum(claude_code_session_count) by (team)

      # Average tokens per session
      - record: claude_code:tokens:per_session
        expr: |
          (sum(claude_code_tokens_input) + sum(claude_code_tokens_output)) by (team)
          /
          sum(claude_code_session_count) by (team)
```

## Cost Analysis Techniques

### PromQL Queries for Cost Analysis

#### Total Cost by Team (Last 30 Days)

```promql
sum(increase(claude_code_cost_total[30d])) by (team)
```

#### Daily Cost Trend

```promql
sum(increase(claude_code_cost_total[1d])) by (team)
```

#### Input vs Output Token Ratio

```promql
sum(rate(claude_code_tokens_output[1h]))
/
sum(rate(claude_code_tokens_input[1h]))
```

#### Cost per API Call

```promql
sum(claude_code_cost_total)
/
sum(claude_code_api_requests)
```

#### Most Expensive Sessions

```promql
topk(10,
  sum by (session_id) (claude_code_cost_total)
)
```

#### Cost Breakdown by Environment

```promql
sum(increase(claude_code_cost_total[7d])) by (environment)
```

### Grafana Dashboard Panels

#### Panel 1: Cost Overview

```json
{
  "title": "Total Cost (Last 30 Days)",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(increase(claude_code_cost_total[30d]))",
      "legendFormat": "Total Cost"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "currencyUSD",
      "decimals": 2
    }
  }
}
```

#### Panel 2: Daily Cost Trend

```json
{
  "title": "Daily Cost by Team",
  "type": "timeseries",
  "targets": [
    {
      "expr": "sum(increase(claude_code_cost_total[1d])) by (team)",
      "legendFormat": "{{team}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "currencyUSD"
    }
  }
}
```

#### Panel 3: Token Usage Heatmap

```json
{
  "title": "Token Usage by Hour of Day",
  "type": "heatmap",
  "targets": [
    {
      "expr": "sum(increase(claude_code_tokens_input[1h])) by (team)",
      "legendFormat": "{{team}}"
    }
  ]
}
```

## Optimization Strategies

### 1. Right-Size Model Selection

Use smaller models for simpler tasks:

```bash
# Use Haiku for simple tasks (80% cheaper than Sonnet)
claude --model haiku "format this json"

# Use Sonnet for complex tasks
claude --model sonnet "refactor this complex function"

# Reserve Opus for critical reasoning
claude --model opus "architect this system"
```

### 2. Prompt Optimization

Reduce input tokens with concise prompts:

```bash
# Instead of verbose prompts:
# "Can you please help me write a function that takes a list and returns..."

# Use concise prompts:
# "Write: function to filter even numbers from list"
```

### 3. Context Window Management

```bash
# Clear context periodically
claude --clear

# Avoid unnecessary file reads
# Instead of reading entire large files, use targeted reads
```

### 4. Caching Strategies

Implement response caching for repeated queries:

```python
# Example: Cache wrapper for Claude Code
import hashlib
import json
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_claude_query(prompt_hash):
    # Only call Claude for new unique prompts
    pass
```

### 5. Batch Operations

Combine multiple small operations:

```bash
# Instead of multiple calls:
# claude "fix typo in file1"
# claude "fix typo in file2"
# claude "fix typo in file3"

# Batch into one call:
claude "fix typos in file1, file2, and file3"
```

### 6. Set Usage Quotas

Implement team-level quotas:

```yaml
# Quota enforcement in OTEL Collector
processors:
  filter/quota:
    metrics:
      metric:
        - 'team == "marketing" and daily_cost > 100'
```

## Alerting on Cost Anomalies

### Prometheus Alerting Rules

```yaml
# alert-rules.yaml
groups:
  - name: claude_code_cost_alerts
    rules:
      # Alert when daily cost exceeds budget
      - alert: ClaudeCodeDailyCostExceeded
        expr: |
          sum(increase(claude_code_cost_total[24h])) by (team) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Daily cost exceeded $100 for team {{ $labels.team }}"
          description: "Team {{ $labels.team }} has spent ${{ $value }} in the last 24 hours"

      # Alert on cost spike (2x normal)
      - alert: ClaudeCodeCostSpike
        expr: |
          sum(rate(claude_code_cost_total[1h])) by (team)
          >
          2 * avg_over_time(sum(rate(claude_code_cost_total[1h])) by (team)[7d:1h])
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Cost spike detected for team {{ $labels.team }}"
          description: "Current cost rate is 2x the 7-day average"

      # Alert when approaching monthly budget
      - alert: ClaudeCodeMonthlyBudgetWarning
        expr: |
          sum(increase(claude_code_cost_total[30d])) by (team) > 800
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Monthly budget warning for team {{ $labels.team }}"
          description: "Team {{ $labels.team }} is approaching monthly budget limit"

      # Alert on unusually long sessions (potential runaway)
      - alert: ClaudeCodeLongSession
        expr: |
          claude_code_session_duration_seconds > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Long-running Claude Code session detected"
          description: "Session running for over 1 hour"
```

### Grafana Alerting

```yaml
# grafana-alert.yaml
apiVersion: 1
groups:
  - name: cost_alerts
    folder: Claude Code
    interval: 5m
    rules:
      - uid: daily-cost-alert
        title: Daily Cost Exceeded
        condition: A
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: sum(increase(claude_code_cost_total[24h]))
        for: 5m
        annotations:
          summary: Daily Claude Code cost exceeded threshold
        labels:
          severity: warning
```

## Reporting and Budgeting

### Weekly Cost Report Query

```promql
# Total cost by team for last week
sum(increase(claude_code_cost_total[7d])) by (team)

# Compare to previous week
sum(increase(claude_code_cost_total[7d])) by (team)
-
sum(increase(claude_code_cost_total[7d] offset 7d)) by (team)
```

### Monthly Budget Dashboard

```json
{
  "title": "Monthly Budget Tracker",
  "panels": [
    {
      "title": "Budget Utilization",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(increase(claude_code_cost_total[30d])) / 1000 * 100",
          "legendFormat": "% of $1000 budget"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "max": 100,
          "thresholds": {
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 70},
              {"color": "red", "value": 90}
            ]
          }
        }
      }
    },
    {
      "title": "Projected Monthly Cost",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(claude_code_cost_total[7d])) * 86400 * 30",
          "legendFormat": "Projected"
        }
      ]
    }
  ]
}
```

### Cost Allocation Report Script

```bash
#!/bin/bash
# cost-report.sh - Generate weekly cost report

PROMETHEUS_URL="http://localhost:9090"
START_DATE=$(date -d "7 days ago" +%Y-%m-%dT00:00:00Z)
END_DATE=$(date +%Y-%m-%dT00:00:00Z)

echo "Claude Code Cost Report: $START_DATE to $END_DATE"
echo "================================================"

# Query Prometheus for cost by team
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_cost_total[7d])) by (team)" \
  | jq -r '.data.result[] | "\(.metric.team): $\(.value[1])"'

echo ""
echo "Token Usage:"
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_tokens_input[7d])) by (team)" \
  | jq -r '.data.result[] | "\(.metric.team) input: \(.value[1]) tokens"'

curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_tokens_output[7d])) by (team)" \
  | jq -r '.data.result[] | "\(.metric.team) output: \(.value[1]) tokens"'
```

## Cost Optimization Checklist

- [ ] Enable telemetry with proper resource attributes
- [ ] Set up Prometheus recording rules for cost metrics
- [ ] Create Grafana dashboards for cost visibility
- [ ] Configure alerting for budget thresholds
- [ ] Implement team-level quotas
- [ ] Review model selection for different use cases
- [ ] Optimize prompts for token efficiency
- [ ] Set up weekly cost reports
- [ ] Establish monthly budget review process
- [ ] Train team on cost-efficient usage patterns

## Related Documentation

- [Data Export & Analysis Guide](./08-data-export-analysis.md)
- [Grafana Dashboard Development](./06-grafana-dashboards.md)
- [CI/CD Integration Patterns](./07-cicd-integration.md)
