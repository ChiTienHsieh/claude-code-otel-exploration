# Grafana Dashboard Development Guide

> **Last Updated**: 2026-02-03

A step-by-step guide to creating, customizing, and managing Grafana dashboards for Claude Code monitoring.

## Table of Contents

- [Getting Started](#getting-started)
- [Dashboard Structure](#dashboard-structure)
- [Creating Panels](#creating-panels)
- [PromQL for Claude Code](#promql-for-claude-code)
- [Variables and Templates](#variables-and-templates)
- [Alerting](#alerting)
- [Best Practices](#best-practices)
- [Complete Dashboard JSON](#complete-dashboard-json)

## Getting Started

### Prerequisites

- Grafana 9.0+ installed
- Prometheus datasource configured
- Claude Code metrics being collected

### Access Grafana

```bash
# If using the verify environment
docker compose up -d
open http://localhost:3000

# Default credentials
# Username: admin
# Password: admin
```

### Add Prometheus Datasource

1. Go to Configuration → Data Sources
2. Click "Add data source"
3. Select "Prometheus"
4. Set URL: `http://prometheus:9090`
5. Click "Save & Test"

## Dashboard Structure

### Recommended Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Dashboard Header                              │
│  [Variables: Team ▼] [Environment ▼] [Time Range ▼]                │
├───────────────────────┬───────────────────────┬────────────────────┤
│                       │                       │                    │
│    Total Sessions     │    Total Tokens       │    Total Cost      │
│        (Stat)         │       (Stat)          │      (Stat)        │
│                       │                       │                    │
├───────────────────────┴───────────────────────┴────────────────────┤
│                                                                     │
│                    Token Usage Over Time                            │
│                      (Time Series)                                  │
│                                                                     │
├─────────────────────────────────┬───────────────────────────────────┤
│                                 │                                   │
│       Cost by Team              │      Sessions by Environment      │
│        (Pie Chart)              │           (Bar Chart)             │
│                                 │                                   │
├─────────────────────────────────┴───────────────────────────────────┤
│                                                                     │
│                    Activity Timeline                                │
│                      (Heatmap)                                      │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│              Recent Events / Tool Usage Table                       │
│                       (Table)                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Creating Panels

### Panel 1: Total Sessions (Stat)

```json
{
  "title": "Total Sessions",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(increase(claude_code_session_count{team=~\"$team\", environment=~\"$environment\"}[$__range]))",
      "legendFormat": "Sessions"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "color": {
        "mode": "thresholds"
      },
      "thresholds": {
        "steps": [
          {"color": "green", "value": null},
          {"color": "yellow", "value": 100},
          {"color": "red", "value": 500}
        ]
      }
    }
  },
  "options": {
    "colorMode": "value",
    "graphMode": "area",
    "justifyMode": "auto",
    "textMode": "auto"
  }
}
```

### Panel 2: Total Tokens (Stat)

```json
{
  "title": "Total Tokens",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(increase(claude_code_tokens_input{team=~\"$team\"}[$__range])) + sum(increase(claude_code_tokens_output{team=~\"$team\"}[$__range]))",
      "legendFormat": "Tokens"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "decimals": 0
    }
  }
}
```

### Panel 3: Total Cost (Stat)

```json
{
  "title": "Total Cost",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(increase(claude_code_cost_total{team=~\"$team\"}[$__range]))",
      "legendFormat": "Cost"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "currencyUSD",
      "decimals": 2,
      "thresholds": {
        "steps": [
          {"color": "green", "value": null},
          {"color": "yellow", "value": 100},
          {"color": "red", "value": 500}
        ]
      }
    }
  }
}
```

### Panel 4: Token Usage Over Time (Time Series)

```json
{
  "title": "Token Usage Over Time",
  "type": "timeseries",
  "targets": [
    {
      "expr": "sum(rate(claude_code_tokens_input{team=~\"$team\"}[5m])) by (team)",
      "legendFormat": "Input - {{team}}"
    },
    {
      "expr": "sum(rate(claude_code_tokens_output{team=~\"$team\"}[5m])) by (team)",
      "legendFormat": "Output - {{team}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "custom": {
        "drawStyle": "line",
        "lineInterpolation": "smooth",
        "fillOpacity": 10,
        "gradientMode": "scheme",
        "spanNulls": false,
        "showPoints": "auto",
        "pointSize": 5,
        "stacking": {
          "mode": "none"
        }
      }
    }
  },
  "options": {
    "tooltip": {
      "mode": "multi",
      "sort": "desc"
    },
    "legend": {
      "displayMode": "table",
      "placement": "bottom",
      "calcs": ["sum", "mean", "max"]
    }
  }
}
```

### Panel 5: Cost by Team (Pie Chart)

```json
{
  "title": "Cost by Team",
  "type": "piechart",
  "targets": [
    {
      "expr": "sum(increase(claude_code_cost_total[$__range])) by (team)",
      "legendFormat": "{{team}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "currencyUSD"
    }
  },
  "options": {
    "legend": {
      "displayMode": "table",
      "placement": "right",
      "values": ["value", "percent"]
    },
    "pieType": "donut",
    "tooltip": {
      "mode": "single"
    }
  }
}
```

### Panel 6: Sessions by Environment (Bar Chart)

```json
{
  "title": "Sessions by Environment",
  "type": "barchart",
  "targets": [
    {
      "expr": "sum(increase(claude_code_session_count[$__range])) by (environment)",
      "legendFormat": "{{environment}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "color": {
        "mode": "palette-classic"
      }
    }
  },
  "options": {
    "orientation": "horizontal",
    "showValue": "always",
    "groupWidth": 0.7,
    "barWidth": 0.8
  }
}
```

### Panel 7: Activity Heatmap

```json
{
  "title": "Activity by Hour",
  "type": "heatmap",
  "targets": [
    {
      "expr": "sum(increase(claude_code_session_count[1h])) by (team)",
      "legendFormat": "{{team}}"
    }
  ],
  "options": {
    "calculate": false,
    "cellGap": 2,
    "color": {
      "scheme": "Spectral",
      "mode": "scheme"
    },
    "yAxis": {
      "axisPlacement": "left"
    }
  }
}
```

### Panel 8: Tool Usage Table

```json
{
  "title": "Recent Tool Usage",
  "type": "table",
  "targets": [
    {
      "expr": "topk(20, sum(increase(claude_code_tool_decisions[$__range])) by (tool_name, team))",
      "format": "table",
      "instant": true
    }
  ],
  "transformations": [
    {
      "id": "organize",
      "options": {
        "excludeByName": {"Time": true},
        "renameByName": {
          "tool_name": "Tool",
          "team": "Team",
          "Value": "Usage Count"
        }
      }
    },
    {
      "id": "sortBy",
      "options": {
        "sort": [{"field": "Usage Count", "desc": true}]
      }
    }
  ],
  "fieldConfig": {
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "Usage Count"},
        "properties": [
          {"id": "custom.cellOptions", "value": {"type": "color-background"}},
          {"id": "color", "value": {"mode": "continuous-GrYlRd"}}
        ]
      }
    ]
  }
}
```

## PromQL for Claude Code

### Essential Queries

```promql
# Total sessions in time range
sum(increase(claude_code_session_count[$__range]))

# Token usage rate (per second)
rate(claude_code_tokens_input[5m])

# Cost accumulation
sum(increase(claude_code_cost_total[24h])) by (team)

# API request rate
rate(claude_code_api_requests[5m])

# Average tokens per session
sum(claude_code_tokens_input + claude_code_tokens_output) / sum(claude_code_session_count)

# Cost per token (efficiency metric)
sum(claude_code_cost_total) / (sum(claude_code_tokens_input) + sum(claude_code_tokens_output))
```

### Advanced Queries

```promql
# Token usage anomaly detection (Z-score)
(
  sum(rate(claude_code_tokens_input[5m]))
  -
  avg_over_time(sum(rate(claude_code_tokens_input[5m]))[7d:1h])
)
/
stddev_over_time(sum(rate(claude_code_tokens_input[5m]))[7d:1h])

# Cost trend (week over week comparison)
sum(increase(claude_code_cost_total[7d]))
/
sum(increase(claude_code_cost_total[7d] offset 7d))

# Top 5 most expensive teams
topk(5, sum(increase(claude_code_cost_total[24h])) by (team))

# Session duration percentiles (if available)
histogram_quantile(0.95, sum(rate(claude_code_session_duration_bucket[5m])) by (le))
```

## Variables and Templates

### Team Variable

```json
{
  "name": "team",
  "type": "query",
  "query": "label_values(claude_code_session_count, team)",
  "datasource": "Prometheus",
  "refresh": 1,
  "includeAll": true,
  "allValue": ".*",
  "multi": true
}
```

### Environment Variable

```json
{
  "name": "environment",
  "type": "query",
  "query": "label_values(claude_code_session_count, environment)",
  "datasource": "Prometheus",
  "refresh": 1,
  "includeAll": true,
  "allValue": ".*",
  "multi": true
}
```

### Model Variable

```json
{
  "name": "model",
  "type": "custom",
  "query": "opus,sonnet,haiku",
  "includeAll": true,
  "allValue": ".*",
  "multi": true
}
```

## Alerting

### Alert Rules in Dashboard

```json
{
  "alert": {
    "alertRuleTags": {},
    "conditions": [
      {
        "evaluator": {
          "params": [100],
          "type": "gt"
        },
        "operator": {"type": "and"},
        "query": {"params": ["A", "5m", "now"]},
        "reducer": {"params": [], "type": "sum"},
        "type": "query"
      }
    ],
    "executionErrorState": "alerting",
    "for": "5m",
    "frequency": "1m",
    "handler": 1,
    "name": "High Cost Alert",
    "noDataState": "no_data",
    "notifications": [
      {"uid": "slack-notification-channel"}
    ]
  }
}
```

### Unified Alerting Rule

```yaml
# alert-rules.yaml (Grafana 9+)
apiVersion: 1
groups:
  - name: claude-code-alerts
    folder: Claude Code
    interval: 1m
    rules:
      - uid: high-daily-cost
        title: Daily Cost Exceeded
        condition: A
        data:
          - refId: A
            relativeTimeRange:
              from: 86400
              to: 0
            datasourceUid: prometheus
            model:
              expr: sum(increase(claude_code_cost_total[24h]))
              refId: A
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary: Daily Claude Code cost exceeded ${{ $values.A }}
        labels:
          severity: warning
```

## Best Practices

### 1. Panel Organization

- Group related panels in rows
- Use collapsible rows for detailed views
- Keep overview panels at the top

### 2. Color Consistency

```json
{
  "fieldConfig": {
    "defaults": {
      "color": {
        "mode": "palette-classic"
      }
    },
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "Input"},
        "properties": [
          {"id": "color", "value": {"fixedColor": "blue", "mode": "fixed"}}
        ]
      },
      {
        "matcher": {"id": "byName", "options": "Output"},
        "properties": [
          {"id": "color", "value": {"fixedColor": "green", "mode": "fixed"}}
        ]
      }
    ]
  }
}
```

### 3. Meaningful Thresholds

```json
{
  "thresholds": {
    "mode": "absolute",
    "steps": [
      {"color": "green", "value": null},
      {"color": "yellow", "value": 70},
      {"color": "orange", "value": 85},
      {"color": "red", "value": 95}
    ]
  }
}
```

### 4. Annotations

```json
{
  "annotations": {
    "list": [
      {
        "datasource": "Prometheus",
        "enable": true,
        "expr": "ALERTS{alertname=\"ClaudeCodeHighCost\"}",
        "iconColor": "red",
        "name": "Cost Alerts",
        "step": "1m"
      }
    ]
  }
}
```

## Complete Dashboard JSON

```json
{
  "dashboard": {
    "id": null,
    "uid": "claude-code-overview",
    "title": "Claude Code Overview",
    "tags": ["claude-code", "otel", "monitoring"],
    "timezone": "browser",
    "schemaVersion": 38,
    "version": 1,
    "refresh": "30s",
    "templating": {
      "list": [
        {
          "name": "team",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(claude_code_session_count, team)",
          "refresh": 1,
          "includeAll": true,
          "multi": true,
          "allValue": ".*"
        },
        {
          "name": "environment",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(claude_code_session_count, environment)",
          "refresh": 1,
          "includeAll": true,
          "multi": true,
          "allValue": ".*"
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "Total Sessions",
        "type": "stat",
        "gridPos": {"h": 4, "w": 8, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "sum(increase(claude_code_session_count{team=~\"$team\", environment=~\"$environment\"}[$__range]))",
            "legendFormat": "Sessions"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 100},
                {"color": "red", "value": 500}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Total Tokens",
        "type": "stat",
        "gridPos": {"h": 4, "w": 8, "x": 8, "y": 0},
        "targets": [
          {
            "expr": "sum(increase(claude_code_tokens_input{team=~\"$team\"}[$__range])) + sum(increase(claude_code_tokens_output{team=~\"$team\"}[$__range]))",
            "legendFormat": "Tokens"
          }
        ],
        "fieldConfig": {
          "defaults": {"unit": "short", "decimals": 0}
        }
      },
      {
        "id": 3,
        "title": "Total Cost",
        "type": "stat",
        "gridPos": {"h": 4, "w": 8, "x": 16, "y": 0},
        "targets": [
          {
            "expr": "sum(increase(claude_code_cost_total{team=~\"$team\"}[$__range]))",
            "legendFormat": "Cost"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "currencyUSD",
            "decimals": 2,
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 100},
                {"color": "red", "value": 500}
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "Token Usage Over Time",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 4},
        "targets": [
          {
            "expr": "sum(rate(claude_code_tokens_input{team=~\"$team\"}[5m])) by (team)",
            "legendFormat": "Input - {{team}}"
          },
          {
            "expr": "sum(rate(claude_code_tokens_output{team=~\"$team\"}[5m])) by (team)",
            "legendFormat": "Output - {{team}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "smooth",
              "fillOpacity": 10
            }
          }
        },
        "options": {
          "legend": {
            "displayMode": "table",
            "placement": "bottom",
            "calcs": ["sum", "mean", "max"]
          }
        }
      },
      {
        "id": 5,
        "title": "Cost by Team",
        "type": "piechart",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 12},
        "targets": [
          {
            "expr": "sum(increase(claude_code_cost_total[$__range])) by (team)",
            "legendFormat": "{{team}}"
          }
        ],
        "fieldConfig": {"defaults": {"unit": "currencyUSD"}},
        "options": {
          "legend": {
            "displayMode": "table",
            "placement": "right",
            "values": ["value", "percent"]
          },
          "pieType": "donut"
        }
      },
      {
        "id": 6,
        "title": "Sessions by Environment",
        "type": "barchart",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 12},
        "targets": [
          {
            "expr": "sum(increase(claude_code_session_count[$__range])) by (environment)",
            "legendFormat": "{{environment}}"
          }
        ],
        "options": {
          "orientation": "horizontal",
          "showValue": "always"
        }
      }
    ],
    "time": {"from": "now-24h", "to": "now"}
  }
}
```

## Provisioning Dashboards

### Directory Structure

```
grafana/
├── provisioning/
│   ├── dashboards/
│   │   ├── dashboard.yml
│   │   └── json/
│   │       └── claude-code-overview.json
│   └── datasources/
│       └── datasources.yml
```

### Dashboard Provisioning Config

```yaml
# dashboard.yml
apiVersion: 1
providers:
  - name: 'Claude Code'
    orgId: 1
    folder: 'Claude Code'
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /etc/grafana/provisioning/dashboards/json
```

## Related Documentation

- [Data Export & Analysis Guide](./08-data-export-analysis.md)
- [Cost Optimization Guide](./03-cost-optimization.md)
- [Troubleshooting Playbook](./04-troubleshooting.md)
