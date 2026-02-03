# 資料匯出與分析指南

> **最後更新**：2026-02-03

本指南全面說明如何使用 PromQL 和其他工具匯出、查詢和分析 Claude Code 遙測資料。

## 目錄

- [PromQL 基礎](#promql-基礎)
- [基本查詢](#基本查詢)
- [進階分析](#進階分析)
- [資料匯出方法](#資料匯出方法)
- [分析工作流程](#分析工作流程)
- [視覺化工具](#視覺化工具)
- [報告範本](#報告範本)

## PromQL 基礎

### 基本語法

```promql
# Metric selection
claude_code_session_count

# Label filtering
claude_code_session_count{team="engineering"}

# Regex matching
claude_code_session_count{team=~"eng.*"}

# Negative matching
claude_code_session_count{environment!="production"}
```

### 常用函數

```promql
# Rate of change (per second)
rate(claude_code_tokens_input[5m])

# Increase over time range
increase(claude_code_session_count[1h])

# Sum across all series
sum(claude_code_cost_total)

# Sum by label
sum by (team) (claude_code_cost_total)

# Average
avg(claude_code_tokens_input)

# Max/Min
max(claude_code_cost_total)
min(claude_code_cost_total)

# Top K
topk(5, claude_code_cost_total)
```

### 時間範圍修飾符

```promql
# Last 5 minutes
claude_code_session_count[5m]

# Last 1 hour
claude_code_session_count[1h]

# Last 24 hours
claude_code_session_count[24h]

# Offset (historical data)
claude_code_session_count offset 1d

# At specific time (Unix timestamp)
claude_code_session_count @ 1704067200
```

## 基本查詢

### Session Metrics

```promql
# Total sessions (all time)
sum(claude_code_session_count)

# Sessions per hour
sum(increase(claude_code_session_count[1h]))

# Sessions by team (last 24h)
sum(increase(claude_code_session_count[24h])) by (team)

# Active sessions (if gauge metric exists)
claude_code_active_sessions

# Session duration percentiles
histogram_quantile(0.95, sum(rate(claude_code_session_duration_bucket[5m])) by (le))
```

### Token 使用量

```promql
# Total input tokens (last 24h)
sum(increase(claude_code_tokens_input[24h]))

# Total output tokens (last 24h)
sum(increase(claude_code_tokens_output[24h]))

# Token rate (per second)
sum(rate(claude_code_tokens_input[5m]))

# Input/Output ratio
sum(rate(claude_code_tokens_output[1h])) / sum(rate(claude_code_tokens_input[1h]))

# Tokens per session
(sum(claude_code_tokens_input) + sum(claude_code_tokens_output))
/
sum(claude_code_session_count)

# Token usage by model
sum(increase(claude_code_tokens_input[24h])) by (model)
```

### 成本分析

```promql
# Total cost (last 24h)
sum(increase(claude_code_cost_total[24h]))

# Cost by team (last 7 days)
sum(increase(claude_code_cost_total[7d])) by (team)

# Daily cost trend
sum(increase(claude_code_cost_total[1d]))

# Cost per session
sum(claude_code_cost_total) / sum(claude_code_session_count)

# Hourly burn rate
sum(rate(claude_code_cost_total[1h])) * 3600

# Weekly cost comparison (current vs last week)
sum(increase(claude_code_cost_total[7d]))
-
sum(increase(claude_code_cost_total[7d] offset 7d))

# Month-to-date cost（本月至今成本）
# 使用 day_of_month() 函數計算動態時間範圍
sum(increase(claude_code_cost_total[30d]))
  * (day_of_month(vector(time())) / 30)
```

### API Metrics

```promql
# API request rate
sum(rate(claude_code_api_requests[5m]))

# API error rate
sum(rate(claude_code_api_errors[5m])) / sum(rate(claude_code_api_requests[5m]))

# Requests by status code
sum(increase(claude_code_api_requests[1h])) by (status_code)

# API latency percentiles
histogram_quantile(0.99, sum(rate(claude_code_api_latency_bucket[5m])) by (le))
```

### 工具使用情況

```promql
# Tool usage count
sum(increase(claude_code_tool_decisions[24h])) by (tool_name)

# Most used tools
topk(10, sum(increase(claude_code_tool_decisions[24h])) by (tool_name))

# Tool success rate
sum(increase(claude_code_tool_decisions{status="success"}[24h])) by (tool_name)
/
sum(increase(claude_code_tool_decisions[24h])) by (tool_name)
```

## 進階分析

### 異常偵測

```promql
# Z-score based anomaly (>2 standard deviations)
(
  sum(rate(claude_code_tokens_input[5m]))
  -
  avg_over_time(sum(rate(claude_code_tokens_input[5m]))[7d:1h])
)
/
stddev_over_time(sum(rate(claude_code_tokens_input[5m]))[7d:1h])
> 2

# Sudden cost spike (3x normal)
sum(rate(claude_code_cost_total[5m]))
>
3 * avg_over_time(sum(rate(claude_code_cost_total[5m]))[24h:5m])
```

### 趨勢分析

```promql
# Linear regression slope (cost trend)
deriv(sum(claude_code_cost_total)[1h:5m])

# Predict cost in 1 hour
sum(claude_code_cost_total)
+
deriv(sum(claude_code_cost_total)[1h:5m]) * 3600

# Week-over-week growth rate
(
  sum(increase(claude_code_cost_total[7d]))
  -
  sum(increase(claude_code_cost_total[7d] offset 7d))
)
/
sum(increase(claude_code_cost_total[7d] offset 7d))
* 100
```

### 群組分析

```promql
# Cost by team and environment
sum(increase(claude_code_cost_total[24h])) by (team, environment)

# Heavy users (sessions > 10 per day)
count(
  sum(increase(claude_code_session_count[24h])) by (user) > 10
)

# New vs returning users (if user tracking exists)
count(claude_code_session_count) by (user_type)
```

### 效率指標

```promql
# Cost per token
sum(claude_code_cost_total) / (sum(claude_code_tokens_input) + sum(claude_code_tokens_output))

# Tokens per dollar
(sum(claude_code_tokens_input) + sum(claude_code_tokens_output)) / sum(claude_code_cost_total)

# Output efficiency (output tokens / input tokens)
sum(claude_code_tokens_output) / sum(claude_code_tokens_input)
```

## 資料匯出方法

### Prometheus HTTP API

```bash
# Instant query
curl -s "http://localhost:9090/api/v1/query" \
  --data-urlencode "query=sum(claude_code_cost_total)" \
  | jq '.data.result'

# Range query
curl -s "http://localhost:9090/api/v1/query_range" \
  --data-urlencode "query=sum(rate(claude_code_tokens_input[5m])) by (team)" \
  --data-urlencode "start=$(date -d '24 hours ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode "step=1h" \
  | jq '.data.result'

# Export to CSV
curl -s "http://localhost:9090/api/v1/query_range" \
  --data-urlencode "query=sum(increase(claude_code_cost_total[1h])) by (team)" \
  --data-urlencode "start=$(date -d '7 days ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode "step=1h" \
  | jq -r '.data.result[] | .metric.team as $team | .values[] | [$team, .[0], .[1]] | @csv' \
  > cost_by_team.csv
```

### Python 匯出腳本

```python
#!/usr/bin/env python3
"""export_claude_metrics.py - Export Claude Code metrics to various formats"""

import requests
import pandas as pd
from datetime import datetime, timedelta
import json

PROMETHEUS_URL = "http://localhost:9090"

def query_prometheus(query: str, start: datetime, end: datetime, step: str = "1h"):
    """Query Prometheus range API"""
    response = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query_range",
        params={
            "query": query,
            "start": start.timestamp(),
            "end": end.timestamp(),
            "step": step
        }
    )
    return response.json()["data"]["result"]

def export_to_dataframe(results):
    """Convert Prometheus results to pandas DataFrame"""
    rows = []
    for result in results:
        labels = result["metric"]
        for timestamp, value in result["values"]:
            row = {**labels, "timestamp": datetime.fromtimestamp(timestamp), "value": float(value)}
            rows.append(row)
    return pd.DataFrame(rows)

def main():
    end = datetime.now()
    start = end - timedelta(days=7)

    # Query cost by team
    results = query_prometheus(
        "sum(increase(claude_code_cost_total[1h])) by (team)",
        start, end
    )

    df = export_to_dataframe(results)

    # Export to various formats
    df.to_csv("claude_costs.csv", index=False)
    df.to_json("claude_costs.json", orient="records", date_format="iso")
    df.to_parquet("claude_costs.parquet", index=False)

    # Summary statistics
    print("Cost Summary:")
    print(df.groupby("team")["value"].agg(["sum", "mean", "max"]))

if __name__ == "__main__":
    main()
```

### 匯出到 BigQuery

```python
#!/usr/bin/env python3
"""Export to BigQuery for advanced analysis"""

from google.cloud import bigquery
import pandas as pd

def export_to_bigquery(df: pd.DataFrame, project_id: str, dataset: str, table: str):
    client = bigquery.Client(project=project_id)
    table_id = f"{project_id}.{dataset}.{table}"

    job_config = bigquery.LoadJobConfig(
        write_disposition=bigquery.WriteDisposition.WRITE_APPEND,
        schema=[
            bigquery.SchemaField("team", "STRING"),
            bigquery.SchemaField("timestamp", "TIMESTAMP"),
            bigquery.SchemaField("cost", "FLOAT64"),
            bigquery.SchemaField("tokens_input", "INTEGER"),
            bigquery.SchemaField("tokens_output", "INTEGER"),
        ]
    )

    job = client.load_table_from_dataframe(df, table_id, job_config=job_config)
    job.result()
    print(f"Loaded {job.output_rows} rows to {table_id}")
```

## 分析工作流程

### 每週成本分析

```bash
#!/bin/bash
set -euo pipefail
# weekly-analysis.sh

PROMETHEUS_URL="http://localhost:9090"
OUTPUT_DIR="./reports/$(date +%Y-%W)"
mkdir -p "$OUTPUT_DIR"

# Total cost
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_cost_total[7d]))" \
  | jq -r '.data.result[0].value[1]' > "$OUTPUT_DIR/total_cost.txt"

# Cost by team
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sort_desc(sum(increase(claude_code_cost_total[7d])) by (team))" \
  | jq -r '.data.result[] | "\(.metric.team): $\(.value[1])"' > "$OUTPUT_DIR/cost_by_team.txt"

# Token usage
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_tokens_input[7d])) + sum(increase(claude_code_tokens_output[7d]))" \
  | jq -r '.data.result[0].value[1]' > "$OUTPUT_DIR/total_tokens.txt"

# Generate summary
cat << EOF > "$OUTPUT_DIR/summary.md"
# Weekly Claude Code Report - $(date +%Y-%W)

## Cost Summary
- **Total Cost**: \$$(cat "$OUTPUT_DIR/total_cost.txt")
- **Total Tokens**: $(cat "$OUTPUT_DIR/total_tokens.txt")

## Cost by Team
$(cat "$OUTPUT_DIR/cost_by_team.txt")
EOF

echo "Report generated: $OUTPUT_DIR/summary.md"
```

### Jupyter Notebook 分析

```python
# claude_code_analysis.ipynb

# Cell 1: Setup
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from prometheus_api_client import PrometheusConnect

prom = PrometheusConnect(url="http://localhost:9090")

# Cell 2: Load Data
start_time = pd.Timestamp.now() - pd.Timedelta(days=30)
end_time = pd.Timestamp.now()

cost_data = prom.custom_query_range(
    query="sum(increase(claude_code_cost_total[1d])) by (team)",
    start_time=start_time,
    end_time=end_time,
    step="1d"
)

# Cell 3: Process Data
def prometheus_to_df(data):
    rows = []
    for series in data:
        team = series["metric"].get("team", "unknown")
        for ts, val in series["values"]:
            rows.append({
                "team": team,
                "date": pd.Timestamp.fromtimestamp(ts),
                "cost": float(val)
            })
    return pd.DataFrame(rows)

df = prometheus_to_df(cost_data)

# Cell 4: Visualize
plt.figure(figsize=(12, 6))
pivot = df.pivot(index="date", columns="team", values="cost")
pivot.plot(kind="area", stacked=True)
plt.title("Claude Code Cost by Team (30 Days)")
plt.xlabel("Date")
plt.ylabel("Cost ($)")
plt.legend(title="Team", bbox_to_anchor=(1.05, 1))
plt.tight_layout()
plt.savefig("cost_trend.png")

# Cell 5: Summary Statistics
print("Cost Summary by Team:")
print(df.groupby("team")["cost"].agg(["sum", "mean", "std", "max"]))

# Cell 6: Top Users Analysis
top_users = prom.custom_query(
    "topk(10, sum(increase(claude_code_cost_total[30d])) by (user))"
)
print("\nTop 10 Users by Cost:")
for user in top_users:
    print(f"  {user['metric'].get('user', 'unknown')}: ${float(user['value'][1]):.2f}")
```

## 視覺化工具

### Grafana Explore 查詢

將這些儲存為 Grafana Explore 查詢以進行即時分析：

```yaml
# grafana-explore-queries.yaml
queries:
  - name: "Cost Breakdown"
    expr: "sum(increase(claude_code_cost_total[$__range])) by (team, environment)"

  - name: "Token Trend"
    expr: |
      sum(rate(claude_code_tokens_input[$__rate_interval])) +
      sum(rate(claude_code_tokens_output[$__rate_interval]))

  - name: "Session Heatmap"
    expr: "sum(increase(claude_code_session_count[1h])) by (team)"

  - name: "Cost Anomalies"
    expr: |
      (sum(rate(claude_code_cost_total[5m])) -
       avg_over_time(sum(rate(claude_code_cost_total[5m]))[7d:1h]))
      / stddev_over_time(sum(rate(claude_code_cost_total[5m]))[7d:1h])
```

### Matplotlib Dashboard 腳本

```python
#!/usr/bin/env python3
"""Generate static dashboard image"""

import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import requests
from datetime import datetime, timedelta

def query_prom(query):
    resp = requests.get(
        "http://localhost:9090/api/v1/query",
        params={"query": query}
    ).json()
    return resp["data"]["result"]

def create_dashboard():
    fig = plt.figure(figsize=(16, 12))
    gs = gridspec.GridSpec(3, 3, figure=fig)

    # Total Cost (Big Number)
    ax1 = fig.add_subplot(gs[0, 0])
    cost = query_prom("sum(increase(claude_code_cost_total[24h]))")
    cost_val = float(cost[0]["value"][1]) if cost else 0
    ax1.text(0.5, 0.5, f"${cost_val:.2f}", ha="center", va="center",
             fontsize=48, fontweight="bold")
    ax1.set_title("24h Cost", fontsize=14)
    ax1.axis("off")

    # Sessions (Big Number)
    ax2 = fig.add_subplot(gs[0, 1])
    sessions = query_prom("sum(increase(claude_code_session_count[24h]))")
    sessions_val = float(sessions[0]["value"][1]) if sessions else 0
    ax2.text(0.5, 0.5, f"{sessions_val:.0f}", ha="center", va="center",
             fontsize=48, fontweight="bold", color="blue")
    ax2.set_title("24h Sessions", fontsize=14)
    ax2.axis("off")

    # Tokens (Big Number)
    ax3 = fig.add_subplot(gs[0, 2])
    tokens = query_prom("sum(increase(claude_code_tokens_input[24h])) + sum(increase(claude_code_tokens_output[24h]))")
    tokens_val = float(tokens[0]["value"][1]) if tokens else 0
    ax3.text(0.5, 0.5, f"{tokens_val/1000:.0f}K", ha="center", va="center",
             fontsize=48, fontweight="bold", color="green")
    ax3.set_title("24h Tokens", fontsize=14)
    ax3.axis("off")

    # Cost by Team (Pie)
    ax4 = fig.add_subplot(gs[1, :2])
    team_costs = query_prom("sum(increase(claude_code_cost_total[24h])) by (team)")
    if team_costs:
        teams = [r["metric"].get("team", "unknown") for r in team_costs]
        values = [float(r["value"][1]) for r in team_costs]
        ax4.pie(values, labels=teams, autopct="%1.1f%%")
        ax4.set_title("Cost by Team", fontsize=14)

    # Top Tools (Bar)
    ax5 = fig.add_subplot(gs[1, 2])
    tools = query_prom("topk(5, sum(increase(claude_code_tool_decisions[24h])) by (tool_name))")
    if tools:
        tool_names = [r["metric"].get("tool_name", "unknown") for r in tools]
        tool_counts = [float(r["value"][1]) for r in tools]
        ax5.barh(tool_names, tool_counts)
        ax5.set_title("Top Tools", fontsize=14)

    plt.suptitle(f"Claude Code Dashboard - {datetime.now().strftime('%Y-%m-%d %H:%M')}", fontsize=16)
    plt.tight_layout()
    plt.savefig("claude_dashboard.png", dpi=150, bbox_inches="tight")
    print("Dashboard saved to claude_dashboard.png")

if __name__ == "__main__":
    create_dashboard()
```

## 報告範本

### 主管摘要範本

```markdown
# Claude Code 使用報告

**期間**：{{start_date}} 至 {{end_date}}
**產生時間**：{{generation_date}}

## 執行摘要

| 指標 | 數值 | 變化 |
|--------|-------|--------|
| 總成本 | ${{total_cost}} | {{cost_change}}% |
| 總 Sessions | {{total_sessions}} | {{sessions_change}}% |
| 總 Tokens | {{total_tokens}} | {{tokens_change}}% |
| 活躍使用者 | {{active_users}} | {{users_change}}% |

## 各團隊成本明細

| 團隊 | 成本 | 佔比 | Sessions | 平均成本/Session |
|------|------|------------|----------|------------------|
{{#each teams}}
| {{name}} | ${{cost}} | {{percentage}}% | {{sessions}} | ${{avg_cost}} |
{{/each}}

## 建議

1. {{recommendation_1}}
2. {{recommendation_2}}
3. {{recommendation_3}}

## 趨勢

![Cost Trend](./charts/cost_trend.png)
![Token Usage](./charts/token_trend.png)
```

### 自動化報告產生器

```python
#!/usr/bin/env python3
"""generate_report.py - Generate Claude Code usage report"""

import requests
from datetime import datetime, timedelta
from jinja2 import Template

PROMETHEUS_URL = "http://localhost:9090"

def query(q):
    r = requests.get(f"{PROMETHEUS_URL}/api/v1/query", params={"query": q})
    result = r.json()["data"]["result"]
    return float(result[0]["value"][1]) if result else 0

def generate_report():
    # Collect metrics
    metrics = {
        "total_cost": query("sum(increase(claude_code_cost_total[7d]))"),
        "total_sessions": query("sum(increase(claude_code_session_count[7d]))"),
        "total_tokens": query("sum(increase(claude_code_tokens_input[7d])) + sum(increase(claude_code_tokens_output[7d]))"),
        "prev_cost": query("sum(increase(claude_code_cost_total[7d] offset 7d))"),
    }

    metrics["cost_change"] = ((metrics["total_cost"] - metrics["prev_cost"]) / metrics["prev_cost"] * 100) if metrics["prev_cost"] else 0

    # Generate report
    template = Template(open("report_template.md").read())
    report = template.render(
        start_date=(datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d"),
        end_date=datetime.now().strftime("%Y-%m-%d"),
        generation_date=datetime.now().strftime("%Y-%m-%d %H:%M"),
        **metrics
    )

    with open(f"reports/report_{datetime.now().strftime('%Y%m%d')}.md", "w") as f:
        f.write(report)

if __name__ == "__main__":
    generate_report()
```

## 相關文件

- [成本優化指南](./03-cost-optimization.md)
- [Grafana Dashboard 開發](./06-grafana-dashboards.md)
- [CI/CD 整合模式](./07-cicd-integration.md)
