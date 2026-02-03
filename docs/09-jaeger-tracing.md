# Jaeger 與 Distributed Tracing 指南

> **最後更新**：2026 年 2 月 3 日

本指南完整說明如何使用 Jaeger 進行 distributed tracing，並與 Claude Code OTEL 整合。

## 目錄

- [Jaeger 簡介](#jaeger-簡介)
- [設定與配置](#設定與配置)
- [理解 Traces](#理解-traces)
- [Jaeger UI 指南](#jaeger-ui-指南)
- [進階查詢](#進階查詢)
- [效能分析](#效能分析)
- [Traces 告警](#traces-告警)
- [最佳實踐](#最佳實踐)

## Jaeger 簡介

### 什麼是 Jaeger？

Jaeger 是一個開源的 distributed tracing 平台，可以幫助您：

- 監控和除錯分散式系統
- 追蹤請求在各服務間的流程
- 識別效能瓶頸
- 分析錯誤的根本原因

### Jaeger 架構

```
                                    ┌─────────────────────────────────────┐
                                    │           Jaeger Backend            │
                                    │                                     │
┌───────────────┐                   │  ┌───────────┐    ┌─────────────┐  │
│  Claude Code  │───OTLP───────────▶│  │ Collector │───▶│   Storage   │  │
└───────────────┘                   │  └───────────┘    │(Cassandra/  │  │
                                    │        │         │Elasticsearch│  │
┌───────────────┐                   │        ▼         │/Memory)     │  │
│ OTEL Collector│───OTLP/Jaeger────▶│  ┌───────────┐    └─────────────┘  │
└───────────────┘                   │  │   Query   │          │         │
                                    │  │  Service  │◀─────────┘         │
                                    │  └─────┬─────┘                    │
                                    │        │                          │
                                    │        ▼                          │
                                    │  ┌───────────┐                    │
                                    │  │  Jaeger   │                    │
                                    │  │    UI     │                    │
                                    │  └───────────┘                    │
                                    └─────────────────────────────────────┘
```

## 設定與配置

### Docker Compose 設定

```yaml
# docker-compose.yaml
# 注意：OTLP ports 只在 otel-collector 上暴露，避免 port 衝突

services:
  jaeger:
    image: jaegertracing/all-in-one:1.54
    ports:
      - "6831:6831/udp"   # Jaeger Thrift compact
      - "6832:6832/udp"   # Jaeger Thrift binary
      - "5778:5778"       # Config server
      - "16686:16686"     # UI
      - "14250:14250"     # Model/Collector gRPC
      - "14268:14268"     # Jaeger Thrift HTTP
      - "14269:14269"     # Health check
      # 注意：4317/4318 由 otel-collector 處理，不在此暴露
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - SPAN_STORAGE_TYPE=badger
      - BADGER_EPHEMERAL=false
      - BADGER_DIRECTORY_VALUE=/badger/data
      - BADGER_DIRECTORY_KEY=/badger/key
    volumes:
      - jaeger-data:/badger
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "localhost:14269"]
      interval: 10s
      timeout: 5s
      retries: 3

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    ports:
      - "4317:4317"   # OTLP gRPC - 唯一入口點
      - "4318:4318"   # OTLP HTTP - 唯一入口點
      - "8888:8888"   # Collector metrics
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    depends_on:
      jaeger:
        condition: service_healthy

volumes:
  jaeger-data:
```

### Jaeger 的 OTEL Collector 配置

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  # Add trace attributes
  attributes:
    actions:
      - key: deployment.environment
        value: production
        action: upsert

  # Tail-based sampling
  tail_sampling:
    decision_wait: 10s
    num_traces: 100
    expected_new_traces_per_sec: 10
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 5000}
      - name: probabilistic
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  # Native OTLP to Jaeger
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

  # Alternative: Jaeger Thrift
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [otlp/jaeger, debug]
```

### Claude Code Tracing 配置

```bash
# Enable telemetry and traces
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Resource attributes for trace identification
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code,service.version=2.1.1,team=engineering"

# Sampling configuration
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=1.0  # 100% for debugging, reduce for production

# Propagation context
export OTEL_PROPAGATORS=tracecontext,baggage
```

## 理解 Traces

### Trace 概念

```
Trace (complete request flow)
│
├── Span A: claude_code.session (root)
│   ├── Duration: 45s
│   ├── Tags: session.id=abc123
│   │
│   ├── Span B: claude_code.prompt
│   │   ├── Duration: 100ms
│   │   └── Tags: prompt.length=150
│   │
│   ├── Span C: claude_code.api_request
│   │   ├── Duration: 2.5s
│   │   ├── Tags: model=sonnet, tokens.input=500
│   │   │
│   │   └── Span D: anthropic.api.messages.create
│   │       ├── Duration: 2.4s
│   │       └── Tags: status=200
│   │
│   └── Span E: claude_code.tool_execution
│       ├── Duration: 500ms
│       ├── Tags: tool.name=Bash
│       │
│       └── Span F: tool.bash.execute
│           ├── Duration: 480ms
│           └── Tags: command="npm test"
```

### Span Attributes 參考

| Attribute | 說明 | 範例 |
|-----------|------|------|
| `service.name` | 服務識別碼 | `claude-code` |
| `service.version` | 版本 | `2.1.1` |
| `session.id` | Session 識別碼 | `abc123def456` |
| `user.id` | 使用者識別碼 | `user@example.com` |
| `tool.name` | 正在執行的工具 | `Bash`, `Read`, `Write` |
| `api.model` | 使用的 AI model | `claude-sonnet-4-20250514` |
| `api.tokens.input` | 輸入 token 數量 | `500` |
| `api.tokens.output` | 輸出 token 數量 | `1200` |
| `error` | Span 是否發生錯誤 | `true`/`false` |
| `error.message` | 錯誤描述 | `Command failed` |

## Jaeger UI 指南

### 存取 UI

```bash
# Start Jaeger
docker compose up -d jaeger

# Open UI
open http://localhost:16686
```

### 搜尋面板

#### 基本搜尋

1. **Service**：選擇 `claude-code`
2. **Operation**：選擇特定操作或 `all`
3. **Tags**：按 attributes 過濾
4. **Lookback**：時間範圍（1h、2h、1d、自訂）
5. **Min/Max Duration**：按延遲過濾

#### Tag 查詢

```
# Find traces with errors
error=true

# Find traces for specific session
session.id=abc123

# Find traces using specific tool
tool.name=Bash

# Find expensive API calls
api.tokens.input>1000

# Combine filters
error=true service.name=claude-code
```

### Trace 檢視

#### Timeline 檢視

```
[▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] claude_code.session (45s)
  [▓▓]                            claude_code.prompt (100ms)
     [▓▓▓▓▓▓▓▓▓]                  claude_code.api_request (2.5s)
         [▓▓▓▓▓▓▓▓]               anthropic.api.messages.create (2.4s)
                    [▓▓]          claude_code.tool_execution (500ms)
                      [▓]         tool.bash.execute (480ms)
```

#### Span 詳細資訊

點擊 span 可以查看：
- **Tags**：鍵值對 attributes
- **Process**：服務資訊
- **Logs**：Span 內的事件/註解
- **Warnings**：偵測到的問題

### 比較 Traces

1. 在搜尋結果中選擇多個 traces
2. 點擊「Compare」按鈕
3. 並排查看 timeline 比較
4. 識別時間和結構的差異

## 進階查詢

### Jaeger Query API

```bash
# Get services
curl "http://localhost:16686/api/services"

# Get operations for a service
curl "http://localhost:16686/api/services/claude-code/operations"

# Search traces
curl "http://localhost:16686/api/traces?service=claude-code&limit=20&lookback=1h"

# Get specific trace
curl "http://localhost:16686/api/traces/{traceID}"

# Search with tags
curl "http://localhost:16686/api/traces?service=claude-code&tags=%7B%22error%22%3A%22true%22%7D"
```

### Python 查詢腳本

```python
#!/usr/bin/env python3
"""jaeger_query.py - Query Jaeger for trace analysis"""

import requests
from datetime import datetime, timedelta
import json

JAEGER_URL = "http://localhost:16686"

def get_traces(service: str, operation: str = None, tags: dict = None,
               lookback: str = "1h", limit: int = 100):
    """Query Jaeger for traces"""
    params = {
        "service": service,
        "limit": limit,
        "lookback": lookback
    }
    if operation:
        params["operation"] = operation
    if tags:
        params["tags"] = json.dumps(tags)

    response = requests.get(f"{JAEGER_URL}/api/traces", params=params)
    return response.json()["data"]

def analyze_traces(traces):
    """Analyze trace data"""
    stats = {
        "total": len(traces),
        "errors": 0,
        "durations": [],
        "operations": {}
    }

    for trace in traces:
        for span in trace["spans"]:
            duration = span["duration"] / 1000  # Convert to ms
            stats["durations"].append(duration)

            op = span["operationName"]
            if op not in stats["operations"]:
                stats["operations"][op] = {"count": 0, "total_duration": 0}
            stats["operations"][op]["count"] += 1
            stats["operations"][op]["total_duration"] += duration

            for tag in span.get("tags", []):
                if tag["key"] == "error" and tag["value"]:
                    stats["errors"] += 1
                    break

    if stats["durations"]:
        stats["avg_duration"] = sum(stats["durations"]) / len(stats["durations"])
        stats["max_duration"] = max(stats["durations"])
        stats["min_duration"] = min(stats["durations"])

    return stats

# Example usage
traces = get_traces("claude-code", tags={"error": "true"}, lookback="24h")
stats = analyze_traces(traces)
print(f"Found {stats['total']} traces, {stats['errors']} with errors")
print(f"Avg duration: {stats['avg_duration']:.2f}ms")
```

## 效能分析

### 識別瓶頸

```python
def find_bottlenecks(traces, threshold_ms=1000):
    """Find slow spans across traces"""
    bottlenecks = []

    for trace in traces:
        for span in trace["spans"]:
            duration_ms = span["duration"] / 1000
            if duration_ms > threshold_ms:
                bottlenecks.append({
                    "trace_id": trace["traceID"],
                    "span_id": span["spanID"],
                    "operation": span["operationName"],
                    "duration_ms": duration_ms,
                    "tags": {t["key"]: t["value"] for t in span.get("tags", [])}
                })

    return sorted(bottlenecks, key=lambda x: x["duration_ms"], reverse=True)
```

### 延遲分析

```python
import numpy as np

def latency_percentiles(traces, operation=None):
    """Calculate latency percentiles for an operation"""
    durations = []

    for trace in traces:
        for span in trace["spans"]:
            if operation is None or span["operationName"] == operation:
                durations.append(span["duration"] / 1000)

    if not durations:
        return None

    return {
        "p50": np.percentile(durations, 50),
        "p90": np.percentile(durations, 90),
        "p95": np.percentile(durations, 95),
        "p99": np.percentile(durations, 99),
        "count": len(durations)
    }
```

### Service 相依性圖

```bash
# Get dependencies
curl "http://localhost:16686/api/dependencies?endTs=$(date +%s)000&lookback=86400000"
```

## Traces 告警

### 從 Jaeger 取得 Prometheus Metrics

Jaeger 會暴露可被 Prometheus 抓取的 metrics：

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'jaeger'
    static_configs:
      - targets: ['jaeger:14269']
```

### 告警規則

```yaml
# prometheus-alerts.yaml
groups:
  - name: jaeger-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(jaeger_collector_spans_received_total{result="err"}[5m]))
          /
          sum(rate(jaeger_collector_spans_received_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High trace error rate detected"

      - alert: SlowSpans
        expr: |
          histogram_quantile(0.95, rate(jaeger_collector_span_latency_bucket[5m]))
          > 5000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "95th percentile span latency > 5s"

      - alert: JaegerCollectorDown
        expr: up{job="jaeger"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Jaeger collector is down"
```

### Grafana 基於 Trace 資料的告警

```json
{
  "alert": {
    "conditions": [
      {
        "evaluator": {"params": [0.1], "type": "gt"},
        "query": {"params": ["A", "5m", "now"]},
        "reducer": {"type": "avg"}
      }
    ],
    "name": "High Error Rate in Traces",
    "frequency": "1m",
    "handler": 1
  },
  "targets": [
    {
      "expr": "sum(rate(jaeger_collector_spans_received_total{result='err'}[5m])) / sum(rate(jaeger_collector_spans_received_total[5m]))"
    }
  ]
}
```

## 最佳實踐

### 1. 有意義的 Span 名稱

```
Good: claude_code.tool.bash.execute
Bad:  execute

Good: anthropic.api.messages.create
Bad:  api_call
```

### 2. 必要的 Tags

務必包含：
- `service.name`
- `service.version`
- `error`（如適用）
- `error.message`（如有錯誤）

### 3. Sampling 策略

```yaml
# Development: Sample everything
OTEL_TRACES_SAMPLER: always_on

# Production: Sample selectively
tail_sampling:
  policies:
    - name: errors      # Always capture errors
      type: status_code
      status_code: {status_codes: [ERROR]}
    - name: slow        # Capture slow traces
      type: latency
      latency: {threshold_ms: 5000}
    - name: sample      # Random sample rest
      type: probabilistic
      probabilistic: {sampling_percentage: 1}
```

### 4. Trace Context 傳播

```bash
# Ensure context flows across services
export OTEL_PROPAGATORS=tracecontext,baggage,b3

# Manual propagation if needed
export TRACEPARENT="00-{trace_id}-{span_id}-01"
```

### 5. Storage 考量

| Backend | 使用場景 | 保留期限 |
|---------|----------|----------|
| Memory | 開發環境 | 僅限 Session |
| Badger | 單節點 | 數天 |
| Elasticsearch | 生產環境 | 數週 |
| Cassandra | 大規模 | 數月 |

```yaml
# Elasticsearch backend
environment:
  - SPAN_STORAGE_TYPE=elasticsearch
  - ES_SERVER_URLS=http://elasticsearch:9200
  - ES_INDEX_PREFIX=jaeger
```

### 6. Trace 保留

```yaml
# Automatic cleanup (Elasticsearch)
environment:
  - ES_MAX_SPAN_AGE=168h  # 7 days
```

## 疑難排解

### 沒有出現 Traces

```bash
# Check Jaeger health
curl http://localhost:14269/

# Check collector is receiving
curl http://localhost:14269/metrics | grep jaeger_collector

# Verify OTEL exporter
export OTEL_TRACES_EXPORTER=console,otlp
claude "test"  # Should see trace output
```

### 不完整的 Traces

- 檢查所有服務是否有相同的 `OTEL_PROPAGATORS`
- 驗證 trace context headers 是否有轉發
- 檢查 sampling 配置是否丟棄了 spans

### UI 高延遲

- 為 storage backend 新增索引
- 增加 Jaeger query 資源
- 減少 trace 保留期限

## 相關文件

- [Traces 深入探討](./01-traces-deep-dive.md)
- [生產環境部署指南](./02-production-deployment.md)
- [疑難排解手冊](./04-troubleshooting.md)
