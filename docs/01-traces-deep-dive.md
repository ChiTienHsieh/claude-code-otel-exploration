# OpenTelemetry Traces 深度探索 - Claude Code

> **狀態**：實驗性/支援狀態不明確
> **最後更新**：2026-02-03

## 概述

本文件探討 Claude Code 的 OpenTelemetry 整合中的 tracing 功能。與 Metrics 和 Logs 已有官方文件記載不同，Traces 的支援仍處於實驗階段。

## Traces 支援現況

### 已知資訊

1. **環境變數存在**：Claude Code 可識別 `OTEL_TRACES_EXPORTER`
2. **官方文件**：僅明確提及 Metrics 和 Logs
3. **觀察到的行為**：Claude Code 可能不會主動產生 trace spans

### 設定方式

```bash
# 啟用 traces exporter（實驗性）
export OTEL_TRACES_EXPORTER=otlp

# 或用於除錯
export OTEL_TRACES_EXPORTER=console

# 多個 exporters
export OTEL_TRACES_EXPORTER=console,otlp
```

### Trace Sampling 設定

```bash
# Sampling 策略
export OTEL_TRACES_SAMPLER=parentbased_traceidratio

# Sampling 比例（0.0 到 1.0）
export OTEL_TRACES_SAMPLER_ARG=0.1  # 取樣 10% 的 traces

# 全部取樣（用於除錯）
export OTEL_TRACES_SAMPLER=always_on

# 不取樣
export OTEL_TRACES_SAMPLER=always_off
```

## 預期的 Trace 結構

若 traces 完整支援，預期會看到以下 spans：

### 假設的 Span 階層

```
claude_code.session (root span)
├── claude_code.prompt
│   ├── claude_code.api_request
│   │   └── anthropic.api.messages.create
│   └── claude_code.response_processing
├── claude_code.tool_execution
│   ├── claude_code.tool.bash
│   ├── claude_code.tool.read
│   └── claude_code.tool.write
└── claude_code.session_complete
```

### 標準 Span Attributes

根據 OTEL semantic conventions，traces 會包含：

| Attribute | 說明 | 範例 |
|-----------|------|------|
| `service.name` | 服務識別碼 | `claude-code` |
| `service.version` | Claude Code 版本 | `2.1.1` |
| `session.id` | 唯一 session ID | `abc123` |
| `user.id` | 使用者識別碼 | （若啟用） |
| `tool.name` | 執行中的工具 | `Bash` |
| `api.model` | 使用的模型 | `claude-sonnet-4-20250514` |

## 測試 Traces 支援

### 步驟 1：設定 OTEL Collector 接收 Traces

在 `otel-collector-config.yaml` 中加入：

```yaml
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

exporters:
  debug:
    verbosity: detailed

  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, jaeger]
```

### 步驟 2：在 Claude Code 中啟用 Traces

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

### 步驟 3：驗證 Trace 輸出

```bash
# 執行 Claude Code
claude "hello world"

# 檢查 Jaeger UI
open http://localhost:16686

# 或檢查 collector debug logs
docker logs otel-collector 2>&1 | grep -i "trace"
```

## 處理 Traces 資料

### Jaeger 查詢

一旦 traces 出現在 Jaeger 中：

1. **尋找慢速操作**：
   - 依 duration 排序
   - 篩選 `error=true`

2. **服務相依性**：
   - 檢視服務拓撲
   - 識別瓶頸

3. **比較 Traces**：
   - 比較相似操作
   - 識別效能退化

### 關聯 Traces 與 Logs

使用 trace context propagation：

```bash
# Logs 中的 trace context
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp

# Logs 將包含 trace_id 和 span_id 用於關聯
```

## 實作自訂 Traces

若 Claude Code 原生不產生 traces，可以使用 wrapper scripts 進行 instrumentation：

### Bash Wrapper 搭配 Trace Context

```bash
#!/bin/bash
set -euo pipefail
# claude-traced.sh - 加入 trace context 的 wrapper

# 產生 trace ID（32 個十六進位字元）
TRACE_ID=$(openssl rand -hex 16)
# 產生 span ID（16 個十六進位字元）
SPAN_ID=$(openssl rand -hex 8)

# 設定 trace parent header
export TRACEPARENT="00-${TRACE_ID}-${SPAN_ID}-01"

# 記錄開始時間
START_TIME=$(date +%s%N)

# 執行 Claude Code
claude "$@"
EXIT_CODE=$?

# 記錄結束時間
END_TIME=$(date +%s%N)
DURATION=$((($END_TIME - $START_TIME) / 1000000))

echo "Trace ID: $TRACE_ID, Duration: ${DURATION}ms"
exit $EXIT_CODE
```

### Python Wrapper 搭配 OpenTelemetry SDK

```python
#!/usr/bin/env python3
"""claude-traced.py - 具有完整 OTEL tracing 的 Python wrapper"""

import subprocess
import sys
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# 設定 tracer
resource = Resource.create({"service.name": "claude-code-wrapper"})
provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

def run_claude(args):
    with tracer.start_as_current_span("claude_code.session") as span:
        span.set_attribute("claude.args", " ".join(args))

        with tracer.start_as_current_span("claude_code.execution"):
            result = subprocess.run(
                ["claude"] + args,
                capture_output=True,
                text=True
            )

            span.set_attribute("claude.exit_code", result.returncode)
            span.set_attribute("claude.stdout_length", len(result.stdout))

            if result.returncode != 0:
                span.set_status(trace.Status(trace.StatusCode.ERROR))
                span.record_exception(Exception(result.stderr))

        return result

if __name__ == "__main__":
    result = run_claude(sys.argv[1:])
    print(result.stdout)
    sys.exit(result.returncode)
```

## Trace Context Propagation

### W3C Trace Context

Claude Code 應遵循標準 trace context headers：

```bash
# 設定 parent trace context
export TRACEPARENT="00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01"

# 可選：trace state
export TRACESTATE="vendor1=value1,vendor2=value2"
```

### Baggage Propagation

```bash
# 設定 baggage 供下游使用
export OTEL_PROPAGATORS=tracecontext,baggage
```

## 建議

### 開發環境

```bash
# 使用 console exporter 查看 trace 輸出
export OTEL_TRACES_EXPORTER=console
export OTEL_TRACES_SAMPLER=always_on
```

### 正式環境

```bash
# 使用 OTLP 搭配 sampling
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1  # 10% sampling
```

### 除錯問題時

```bash
# 啟用所有 signals 搭配 debug 輸出
export OTEL_TRACES_EXPORTER=console,otlp
export OTEL_METRICS_EXPORTER=console,otlp
export OTEL_LOGS_EXPORTER=console,otlp
```

## 已知限制

1. **無原生 Span 產生**：Claude Code 內部可能不會建立 spans
2. **有限的 Instrumentation**：工具執行可能不會被追蹤
3. **無分散式追蹤**：無法跨多個 Claude Code 實例追蹤

## 未來展望

根據 OpenTelemetry 發展路線和 Claude Code 開發：

1. **原生 Span 支援**：預期在未來版本中實現
2. **工具層級追蹤**：每個工具執行的詳細 spans
3. **API 呼叫追蹤**：Anthropic API 互動的 spans
4. **Semantic Conventions**：採用 LLM 專用的 semantic conventions

## 監控 Trace Pipeline 健康狀態

```yaml
# Collector metrics 用於 trace pipeline
service:
  telemetry:
    metrics:
      level: detailed
      address: 0.0.0.0:8888
```

監控的關鍵指標：
- `otelcol_exporter_sent_spans` - 成功匯出的 spans
- `otelcol_exporter_send_failed_spans` - 失敗的 span 匯出
- `otelcol_processor_batch_batch_send_size` - Batch 大小

## 相關文件

- [Jaeger/分散式追蹤指南](./09-jaeger-tracing.md)
- [OTEL Collector 設定](../TUTORIAL_OTEL_zh-TW.md#第三章otel-collector-設定)
- [官方 OpenTelemetry Traces 文件](https://opentelemetry.io/docs/concepts/signals/traces/)
