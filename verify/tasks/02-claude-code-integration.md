# Task 02: Claude Code Integration

驗證 Claude Code 可以成功發送 telemetry 到 OTEL Collector。

## 前置條件

- 完成 [Task 01: Basic Setup](./01-basic-setup.md)
- 已安裝 Claude Code CLI
- 有有效的 Anthropic API Key

## 目標

讓 Claude Code 發送 metrics/traces/logs 到你的 OTEL Collector。

## 步驟

### Step 1: 設定環境變數

在你的 terminal 執行：

```bash
# 必要設定
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Debug 用（可選，會在 terminal 顯示 telemetry）
export OTEL_TRACES_EXPORTER=otlp,console
export OTEL_METRICS_EXPORTER=otlp,console
export OTEL_LOGS_EXPORTER=otlp,console
```

### Step 2: 驗證環境變數

```bash
env | grep -E "(CLAUDE_CODE|OTEL)" | sort
```

預期輸出包含：
```
CLAUDE_CODE_ENABLE_TELEMETRY=1
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

### Step 3: 執行 Claude Code

開始一個簡單的對話：

```bash
# 互動模式
claude

# 或一次性指令
claude "list files in current directory"
```

### Step 4: 觀察 OTEL Collector Logs

開另一個 terminal：

```bash
docker compose logs -f otel-collector
```

預期：看到收到的 traces/metrics/logs 資料。

範例輸出（debug exporter）：
```
otel-collector  | Span #0
otel-collector  |     Trace ID       : abc123...
otel-collector  |     Span ID        : def456...
otel-collector  |     Name           : claude.api.call
```

### Step 5: 檢查 Prometheus Metrics

開啟：http://localhost:9090

在查詢框輸入：
```
{__name__=~"claude_code.*"}
```

預期：看到 `claude_code_*` 開頭的 metrics。

常見 metrics：
- `claude_code_session_count`
- `claude_code_token_usage`
- `claude_code_cost_usage`

### Step 6: 檢查 Jaeger Traces

開啟：http://localhost:16686

1. Service 下拉選 `claude-code`（或類似名稱）
2. 點 "Find Traces"

預期：看到 Claude Code 的 API 呼叫 traces。

### Step 7: 檢查 Grafana Dashboard

開啟：http://localhost:3000

1. Dashboards → Claude Code → Claude Code OTEL Overview
2. 確認有數據顯示

預期：Dashboard 上的 metrics 有數值。

## 驗證清單

- [ ] 環境變數設定正確
- [ ] Claude Code 正常執行
- [ ] OTEL Collector logs 有收到資料
- [ ] Prometheus 有 `claude_code_*` metrics
- [ ] Jaeger 有 traces
- [ ] Grafana dashboard 有數據

## Troubleshooting

### 沒收到任何資料？

1. 確認 `CLAUDE_CODE_ENABLE_TELEMETRY=1`
2. 確認 endpoint 正確：`http://localhost:4317`（注意是 http 不是 https）
3. 檢查 Claude Code 版本：`claude --version`（需要 v2.1.1+）

### Prometheus 有資料但 Grafana 沒有？

1. 檢查 Grafana data source 設定
2. 確認 Prometheus 是預設 data source
3. 手動測試：Explore → Prometheus → 輸入 `claude_code_session_count`

### Console exporter 沒輸出？

確認有設定：
```bash
export OTEL_TRACES_EXPORTER=otlp,console
```

## 下一步

完成後，進行 [Task 03: Privacy Controls](./03-privacy-controls.md)
