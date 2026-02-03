# Claude Code 成本優化指南

> **最後更新**：2026-02-03

本指南說明如何使用 OTEL metrics 來分析、監控和優化 Claude Code 使用成本。

## 目錄

- [了解成本指標](#了解成本指標)
- [設定成本監控](#設定成本監控)
- [成本分析技巧](#成本分析技巧)
- [優化策略](#優化策略)
- [成本異常警報](#成本異常警報)
- [報表與預算](#報表與預算)

## 了解成本指標

### 可用指標

Claude Code 匯出以下成本相關指標：

| 指標 | 說明 | 單位 |
|------|------|------|
| `claude_code_tokens_input` | 消耗的輸入 tokens | tokens |
| `claude_code_tokens_output` | 產生的輸出 tokens | tokens |
| `claude_code_cost_total` | 估計總成本 | USD |
| `claude_code_session_count` | Session 數量 | count |
| `claude_code_api_requests` | API 呼叫次數 | count |

### Token 定價參考（2026）

| 模型 | 輸入（每 1M tokens） | 輸出（每 1M tokens） |
|------|---------------------|---------------------|
| Claude Opus 4.5 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |

## 設定成本監控

### 環境變數

```bash
# 啟用 telemetry 搭配 session 追蹤
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 加入成本中心和團隊標籤
export OTEL_RESOURCE_ATTRIBUTES="team=engineering,cost_center=CC-1234,environment=production"
```

### 成本指標用 OTEL Collector 設定

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  # 加入計算後的成本 attributes
  transform/cost:
    metric_statements:
      - context: datapoint
        statements:
          # 根據 token 數量計算成本
          - set(attributes["estimated_cost_usd"],
              (attributes["input_tokens"] * 0.000003) +
              (attributes["output_tokens"] * 0.000015))

  # 篩選只保留成本相關指標
  filter/cost_metrics:
    metrics:
      include:
        match_type: regexp
        metric_names:
          - claude_code_tokens.*
          - claude_code_cost.*
          - claude_code_session.*

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

建立 recording rules 以提升成本查詢效率：

```yaml
# prometheus-rules.yaml
groups:
  - name: claude_code_cost
    interval: 1m
    rules:
      # 每小時成本率
      - record: claude_code:cost:rate1h
        expr: |
          sum(increase(claude_code_cost_total[1h])) by (team, environment)

      # 每日成本
      - record: claude_code:cost:daily
        expr: |
          sum(increase(claude_code_cost_total[24h])) by (team, environment)

      # Token 使用率（每分鐘）
      - record: claude_code:tokens:rate1m
        expr: |
          sum(rate(claude_code_tokens_input[1m]) + rate(claude_code_tokens_output[1m])) by (team)

      # 每 session 成本
      - record: claude_code:cost:per_session
        expr: |
          sum(claude_code_cost_total) by (team)
          /
          sum(claude_code_session_count) by (team)

      # 每 session 平均 tokens
      - record: claude_code:tokens:per_session
        expr: |
          (sum(claude_code_tokens_input) + sum(claude_code_tokens_output)) by (team)
          /
          sum(claude_code_session_count) by (team)
```

## 成本分析技巧

### 成本分析用 PromQL 查詢

#### 依團隊分組的總成本（過去 30 天）

```promql
sum(increase(claude_code_cost_total[30d])) by (team)
```

#### 每日成本趨勢

```promql
sum(increase(claude_code_cost_total[1d])) by (team)
```

#### 輸入與輸出 Token 比例

```promql
sum(rate(claude_code_tokens_output[1h]))
/
sum(rate(claude_code_tokens_input[1h]))
```

#### 每次 API 呼叫成本

```promql
sum(claude_code_cost_total)
/
sum(claude_code_api_requests)
```

#### 最昂貴的 Sessions

```promql
topk(10,
  sum by (session_id) (claude_code_cost_total)
)
```

#### 依環境分組的成本明細

```promql
sum(increase(claude_code_cost_total[7d])) by (environment)
```

### Grafana Dashboard Panels

#### Panel 1：成本概覽

```json
{
  "title": "總成本（過去 30 天）",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(increase(claude_code_cost_total[30d]))",
      "legendFormat": "總成本"
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

#### Panel 2：每日成本趨勢

```json
{
  "title": "依團隊分組的每日成本",
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

#### Panel 3：Token 使用量 Heatmap

```json
{
  "title": "依時段分組的 Token 使用量",
  "type": "heatmap",
  "targets": [
    {
      "expr": "sum(increase(claude_code_tokens_input[1h])) by (team)",
      "legendFormat": "{{team}}"
    }
  ]
}
```

## 優化策略

### 1. 正確選擇模型

針對不同任務使用適當的模型：

```bash
# 簡單任務使用 Haiku（比 Sonnet 便宜 80%）
claude --model haiku "格式化這個 json"

# 複雜任務使用 Sonnet
claude --model sonnet "重構這個複雜的函式"

# 關鍵推理保留給 Opus
claude --model opus "設計這個系統架構"
```

### 2. Prompt 優化

使用簡潔的 prompts 減少輸入 tokens：

```bash
# 避免冗長的 prompts：
# "可以請你幫我寫一個函式，這個函式會接收一個列表然後回傳..."

# 使用簡潔的 prompts：
# "寫：從列表中篩選偶數的函式"
```

### 3. Context Window 管理

```bash
# 定期清除 context
claude --clear

# 避免不必要的檔案讀取
# 不要讀取整個大檔案，使用針對性的讀取
```

### 4. Caching 策略

為重複查詢實作 response caching：

```python
# 範例：Claude Code 的 cache wrapper
import hashlib
import json
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_claude_query(prompt_hash):
    # 只對新的唯一 prompts 呼叫 Claude
    pass
```

### 5. 批次操作

合併多個小操作：

```bash
# 避免多次呼叫：
# claude "修正 file1 的錯字"
# claude "修正 file2 的錯字"
# claude "修正 file3 的錯字"

# 批次成一次呼叫：
claude "修正 file1、file2 和 file3 中的錯字"
```

### 6. 設定使用配額

實作團隊層級配額：

```yaml
# OTEL Collector 中的配額執行
processors:
  filter/quota:
    error_mode: ignore
    metrics:
      datapoint:
        # 當達到配額時記錄警告
        - 'resource.attributes["team"] == "marketing"'
```

## 成本異常警報

### Prometheus Alerting Rules

```yaml
# alert-rules.yaml
groups:
  - name: claude_code_cost_alerts
    rules:
      # 每日成本超過預算時警報
      - alert: ClaudeCodeDailyCostExceeded
        expr: |
          sum(increase(claude_code_cost_total[24h])) by (team) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "團隊 {{ $labels.team }} 每日成本超過 $100"
          description: "團隊 {{ $labels.team }} 過去 24 小時已花費 ${{ $value }}"

      # 成本飆升警報（2 倍正常值）
      - alert: ClaudeCodeCostSpike
        expr: |
          sum(rate(claude_code_cost_total[1h])) by (team)
          >
          2 * avg_over_time(sum(rate(claude_code_cost_total[1h])) by (team)[7d:1h])
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "偵測到團隊 {{ $labels.team }} 成本飆升"
          description: "目前成本率是 7 天平均值的 2 倍"

      # 接近月預算時警報
      - alert: ClaudeCodeMonthlyBudgetWarning
        expr: |
          sum(increase(claude_code_cost_total[30d])) by (team) > 800
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "團隊 {{ $labels.team }} 月預算警告"
          description: "團隊 {{ $labels.team }} 接近月預算上限"

      # 異常長時間 session 警報（可能失控）
      - alert: ClaudeCodeLongSession
        expr: |
          claude_code_session_duration_seconds > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "偵測到長時間執行的 Claude Code session"
          description: "Session 已執行超過 1 小時"
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
        title: 每日成本超標
        condition: A
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: sum(increase(claude_code_cost_total[24h]))
        for: 5m
        annotations:
          summary: 每日 Claude Code 成本超過閾值
        labels:
          severity: warning
```

## 報表與預算

### 每週成本報表查詢

```promql
# 上週依團隊分組的總成本
sum(increase(claude_code_cost_total[7d])) by (team)

# 與前一週比較
sum(increase(claude_code_cost_total[7d])) by (team)
-
sum(increase(claude_code_cost_total[7d] offset 7d)) by (team)
```

### 每月預算 Dashboard

```json
{
  "title": "每月預算追蹤器",
  "panels": [
    {
      "title": "預算使用率",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(increase(claude_code_cost_total[30d])) / 1000 * 100",
          "legendFormat": "% of $1000 預算"
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
      "title": "預估月成本",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(claude_code_cost_total[7d])) * 86400 * 30",
          "legendFormat": "預估值"
        }
      ]
    }
  ]
}
```

### 成本分配報表 Script

```bash
#!/bin/bash
set -euo pipefail
# cost-report.sh - 產生每週成本報表

PROMETHEUS_URL="http://localhost:9090"
START_DATE=$(date -d "7 days ago" +%Y-%m-%dT00:00:00Z)
END_DATE=$(date +%Y-%m-%dT00:00:00Z)

echo "Claude Code 成本報表：$START_DATE 至 $END_DATE"
echo "================================================"

# 向 Prometheus 查詢依團隊分組的成本
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_cost_total[7d])) by (team)" \
  | jq -r '.data.result[] | "\(.metric.team): $\(.value[1])"'

echo ""
echo "Token 使用量："
curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_tokens_input[7d])) by (team)" \
  | jq -r '.data.result[] | "\(.metric.team) 輸入: \(.value[1]) tokens"'

curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=sum(increase(claude_code_tokens_output[7d])) by (team)" \
  | jq -r '.data.result[] | "\(.metric.team) 輸出: \(.value[1]) tokens"'
```

## 成本優化檢查清單

- [ ] 啟用 telemetry 並設定適當的 resource attributes
- [ ] 設定 Prometheus recording rules 用於成本指標
- [ ] 建立 Grafana dashboards 以提供成本可視性
- [ ] 設定預算閾值警報
- [ ] 實作團隊層級配額
- [ ] 針對不同使用案例審視模型選擇
- [ ] 優化 prompts 以提升 token 效率
- [ ] 設定每週成本報表
- [ ] 建立每月預算審查流程
- [ ] 訓練團隊了解高成本效益的使用模式

## 相關文件

- [資料匯出與分析指南](./08-data-export-analysis.md)
- [Grafana Dashboard 開發](./06-grafana-dashboards.md)
- [CI/CD 整合模式](./07-cicd-integration.md)
