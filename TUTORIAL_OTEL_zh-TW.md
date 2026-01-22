# Claude Code + OpenTelemetry 完全攻略 ヽ(>∀<☆)☆

> 實驗日期：2026-01-20
> 環境：Claude Code v2.1.1 (Cloud)
> 作者：一個好奇的 Opus 4.5 ( ˘•ω•˘ )

---

## 前言：為什麼要搞這個？

```
OpenTelemetry = Open + Telemetry
               開放   遙測

簡單說就是：讓你看到 Claude Code 在幹嘛的透明儀表板！
```

想像一下：
- 你老闆問：「Claude Code 這個月花了多少錢？」
- 你：「呃... (´・ω・`)」

有了 OTEL，你可以自信地說：
- 「報告！本月用了 1,234,567 tokens，成本 $12.34！」ヽ(>∀<)ノ

---

## 第一章：環境變數大全

### 1.1 開關總開關 (最重要！)

```bash
# 沒設這個，其他都白搭 (╯°□°）╯︵ ┻━┻
export CLAUDE_CODE_ENABLE_TELEMETRY=1
```

這就像電源開關，不打開什麼都不會發生 ( ˘•ω•˘ )

### 1.2 Exporter 設定（資料要送去哪？）

```bash
# 指標 (Metrics) - 數字類的資料 ✅ 官方確認支援
export OTEL_METRICS_EXPORTER=console    # 直接印在 terminal
export OTEL_METRICS_EXPORTER=otlp       # 送到 OTLP collector
export OTEL_METRICS_EXPORTER=prometheus # 送到 Prometheus

# 日誌 (Logs) - 事件類的資料 ✅ 官方確認支援
export OTEL_LOGS_EXPORTER=console       # 直接印在 terminal
export OTEL_LOGS_EXPORTER=otlp          # 送到 OTLP collector

# Traces - 追蹤呼叫鏈 ⚠️ 支援度待確認
export OTEL_TRACES_EXPORTER=console
export OTEL_TRACES_EXPORTER=otlp

# 可以同時送多個地方！用逗號分隔
export OTEL_METRICS_EXPORTER=console,otlp
```

> ⚠️ **Traces 支援度說明**
>
> Claude Code [官方文件](https://code.claude.com/docs/en/monitoring-usage)
> 目前只明確提到 **Metrics** 和 **Logs** exporter。
>
> `OTEL_TRACES_EXPORTER` 在 source code 中存在，但：
> - 官方文件未提及 traces 支援
> - 實測觀察：Claude Code 可能**不主動產生 traces**
> - 此設定可能僅影響 OTEL SDK 的初始化，實際 span 產出待確認
>
> **建議：** 先以 Metrics 和 Logs 為主，Traces 視為實驗性功能。

**小技巧：** Debug 的時候用 `console`，正式環境用 `otlp` (´▽`)

### 1.3 OTLP Endpoint 設定

```bash
# 基本設定
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc  # 或 http/json, http/protobuf

# 加認證 header（送到雲端服務時）
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer your-token"

# TLS/mTLS 設定（企業安全需求）
export OTEL_EXPORTER_OTLP_CERTIFICATE=/path/to/ca.crt
export OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE=/path/to/client.crt
export OTEL_EXPORTER_OTLP_CLIENT_KEY=/path/to/client.key

# 如果是 HTTP（不安全，僅限測試）
export OTEL_EXPORTER_OTLP_INSECURE=true
```

### 1.4 匯出間隔（多久送一次？）

```bash
# 指標匯出間隔（毫秒）
export OTEL_METRIC_EXPORT_INTERVAL=60000    # 60 秒（預設）
export OTEL_METRIC_EXPORT_INTERVAL=10000    # 10 秒（debug 用，看反應快）

# 日誌匯出間隔 - 使用 BLRP 設定
export OTEL_BLRP_SCHEDULE_DELAY=5000        # 5 秒（標準 OTEL 變數）

# Trace 匯出間隔 - 使用 BSP 設定
export OTEL_BSP_SCHEDULE_DELAY=5000         # 5 秒（標準 OTEL 變數）
```

> ⚠️ **重要說明：非標準環境變數**
>
> **`OTEL_LOGS_EXPORT_INTERVAL`** 和 **`OTEL_TRACES_EXPORT_INTERVAL`** 可能**不是**標準 OpenTelemetry 環境變數！
>
> 根據 [OTEL 規範](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)：
> - **Logs** 匯出間隔應使用 **`OTEL_BLRP_SCHEDULE_DELAY`**（Batch Log Record Processor 的排程延遲）
> - **Trace** 匯出間隔應使用 **`OTEL_BSP_SCHEDULE_DELAY`**（Batch Span Processor 的排程延遲）
>
> `OTEL_LOGS_EXPORT_INTERVAL` 和 `OTEL_TRACES_EXPORT_INTERVAL` 在 Claude Code source code 中被發現，
> 但可能是內部實作或已被標準變數覆蓋。**建議優先使用標準的 `OTEL_BLRP_SCHEDULE_DELAY` 和 `OTEL_BSP_SCHEDULE_DELAY`。**

---

## 第二章：隱私控制 (很重要！)

Claude Code 對隱私超小心的 (´・ω・`)

### 2.1 預設行為（安全模式）

| 資料類型 | 預設行為 | 來源 |
|---------|---------|------|
| API Key | ❌ 永遠不記錄 | 官方文件 |
| 檔案內容 | ❌ 永遠不記錄 | 官方文件 |
| User Prompt | ⚠️ 只記錄長度，內容被 redact | 官方文件 + 實測 |
| Tool 輸出 | ⚠️ 預設不記錄 | Source code 分析 |
| Session ID | ✅ 記錄 | 官方文件 + 實測 |
| Token 用量 | ✅ 記錄 | 官方文件 + 實測 |

> 📋 **來源說明**
> - **官方文件**: [Anthropic Docs - Monitoring](https://code.claude.com/docs/en/monitoring-usage)
> - **實測**: 使用 `console` exporter 觀察實際輸出
> - **Source code 分析**: 從 Claude Code v2.1.1 cli.js 分析得出

### 2.2 可選的隱私設定

```bash
# 要記錄 user prompt 內容嗎？（謹慎！）
export OTEL_LOG_USER_PROMPTS=1              # 開啟（可能包含敏感資料）
export OTEL_LOG_USER_PROMPTS=0              # 關閉（預設，只記錄長度）

# 要記錄 tool 輸出內容嗎？
export OTEL_LOG_TOOL_CONTENT=1              # 開啟
export OTEL_LOG_TOOL_CONTENT=0              # 關閉（預設）

# Metrics 包含哪些識別資訊？
export OTEL_METRICS_INCLUDE_SESSION_ID=true     # 包含 session ID（預設）
export OTEL_METRICS_INCLUDE_ACCOUNT_UUID=true   # 包含帳號 UUID（預設）
export OTEL_METRICS_INCLUDE_VERSION=false       # 包含版本資訊（預設關）
```

> 📋 **變數來源說明**
>
> | 變數 | 來源 | 備註 |
> |------|------|------|
> | `OTEL_LOG_USER_PROMPTS` | 官方文件 | [Anthropic Docs](https://code.claude.com/docs/en/monitoring-usage) |
> | `OTEL_LOG_TOOL_CONTENT` | Source code 分析 | 從 cli.js 發現，待官方文件確認 |
> | `OTEL_METRICS_INCLUDE_*` | Source code 分析 | 從 cli.js 發現，待官方文件確認 |
>
> 標註 `Source code 分析` 的變數已在 Claude Code v2.1.1 實測確認可用，
> 但官方文件尚未涵蓋，未來版本可能變更。

**企業用戶請注意：**
- 在開啟 `OTEL_LOG_USER_PROMPTS` 前，請確認符合公司資安政策！
- 用戶的 prompt 可能包含機密資訊！Σ(°△°|||)

---

## 第三章：Resource Attributes（標記你的資料）

```bash
# 自訂屬性，用來區分團隊/專案/環境
export OTEL_RESOURCE_ATTRIBUTES="department=engineering,team=platform,env=production"

# 設定服務名稱
export OTEL_SERVICE_NAME="claude-code-dev-team"
```

**用途範例：**
- `team=frontend` vs `team=backend` → 看哪個團隊用比較多
- `env=dev` vs `env=prod` → 分開統計開發和生產環境
- `project=awesome-app` → 按專案追蹤成本

---

## 第四章：進階調校參數

### 4.0 Claude Code 專屬設定（企業重要！）

```bash
# Headers Helper Debounce（防抖設定）
export CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS=1000  # 毫秒
```

**這是什麼？**

當你使用 `OTEL_EXPORTER_OTLP_HEADERS` 指向一個 helper script 來動態取得認證 token 時
（例如：`OTEL_EXPORTER_OTLP_HEADERS="$(~/.claude/get-otel-token.sh)"`），
這個 debounce 設定可以避免頻繁呼叫 helper script。

**使用情境：**
- 企業環境使用 SSO / OAuth token
- Token 需要定期更新
- 避免每次送資料都執行一次 helper

```bash
# 範例：動態取得 Bearer token
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer $(get-token.sh)"
export CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS=30000  # 30 秒內不重複呼叫
```

> 📋 **來源：** [官方文件](https://code.claude.com/docs/en/monitoring-usage)

### 4.1 Batch 處理設定

```bash
# Batch Span Processor (BSP) - 處理 traces
export OTEL_BSP_MAX_QUEUE_SIZE=2048          # 佇列大小
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512    # 每批次大小
export OTEL_BSP_EXPORT_TIMEOUT=30000         # 匯出超時（毫秒）
export OTEL_BSP_SCHEDULE_DELAY=5000          # 排程延遲

# Batch Log Record Processor (BLRP) - 處理 logs
export OTEL_BLRP_MAX_QUEUE_SIZE=2048
export OTEL_BLRP_MAX_EXPORT_BATCH_SIZE=512
export OTEL_BLRP_EXPORT_TIMEOUT=30000
export OTEL_BLRP_SCHEDULE_DELAY=5000
```

### 4.2 屬性限制

```bash
# 避免爆記憶體
export OTEL_ATTRIBUTE_COUNT_LIMIT=128        # 每個 span 最多幾個屬性
export OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT=4096 # 屬性值最長幾個字元

# Span 專用
export OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT=128
export OTEL_SPAN_EVENT_COUNT_LIMIT=128
export OTEL_SPAN_LINK_COUNT_LIMIT=128

# Log 專用
export OTEL_LOGRECORD_ATTRIBUTE_COUNT_LIMIT=128
export OTEL_LOGRECORD_ATTRIBUTE_VALUE_LENGTH_LIMIT=4096
```

### 4.3 Sampling（取樣）

```bash
# Trace 取樣器
export OTEL_TRACES_SAMPLER=parentbased_always_on  # 全部記錄
export OTEL_TRACES_SAMPLER=parentbased_always_off # 全部不記錄
export OTEL_TRACES_SAMPLER=parentbased_traceidratio

# 取樣比例（配合 traceidratio 使用）
export OTEL_TRACES_SAMPLER_ARG=0.1  # 只記錄 10%
```

---

## 第 4.5 章：OTEL Collector 快速啟動 (´▽`)

> 教學一直提到送資料到 `localhost:4317`，但那是什麼東西？
> 就是 **OpenTelemetry Collector**！讓我來教你怎麼跑起來 ヽ(>∀<☆)☆

### 什麼是 OTEL Collector？

```
OTEL Collector = OpenTelemetry Collector
                 接收、處理、轉發遙測資料的中繼站

Claude Code  ──→  OTEL Collector  ──→  Grafana / Jaeger / Prometheus
                      ↓
                 可以做 filtering、sampling、轉換格式
```

### 方法一：Docker 一鍵啟動（最簡單！）

```bash
# 用官方 OTEL Collector 映像
docker run -d --name otel-collector \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 55679:55679 \
  otel/opentelemetry-collector-contrib:latest
```

**Port 說明：**
- `4317` - gRPC endpoint（預設）
- `4318` - HTTP endpoint
- `55679` - zPages（debug 用）

### 方法二：Docker Compose（推薦！可自訂配置）

建立 `docker-compose.yml`：

```yaml
# 注意：`version` 欄位在 Docker Compose v2+ 已棄用，可省略
# 保留此行是為了相容舊版 docker-compose
version: '3.8'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      # ⚠️ 安全建議：生產環境請改用 "127.0.0.1:port:port" 限制本機存取
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics (collector 自己的)
      - "55679:55679" # zPages
    restart: unless-stopped
```

建立 `otel-collector-config.yaml`：

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
    timeout: 10s
    send_batch_size: 1024

exporters:
  # 先印到 console 確認有收到資料
  # ⚠️ 注意：`logging` exporter 已在 OTEL Collector v0.111.0 移除！
  # 舊寫法 `logging: { loglevel: debug }` 會報錯
  debug:
    verbosity: detailed

  # 也可以轉發到其他地方（例如 Jaeger）
  # jaeger:
  #   endpoint: jaeger:14250
  #   tls:
  #     insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

啟動：

```bash
docker-compose up -d
```

### 方法三：搭配 Grafana Stack（完整 Observability）

如果你想要完整的視覺化體驗，可以用這個擴充版：

```yaml
# 注意：`version` 欄位在 Docker Compose v2+ 已棄用，可省略
version: '3.8'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      # ⚠️ 生產環境建議：使用 "127.0.0.1:port:port" 限制本機存取
      - "4317:4317"
      - "4318:4318"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin  # ⚠️ 生產環境請改用強密碼！
```

### 驗證 Collector 有在跑

```bash
# 檢查 container 狀態
docker ps | grep otel

# 看 collector 的 logs
docker logs otel-collector

# 用 curl 測試 HTTP endpoint
curl -v http://localhost:4318/v1/metrics
```

### 常見問題

**Q: 為什麼 Claude Code 的資料沒送過來？**

A: 檢查這些：
1. `CLAUDE_CODE_ENABLE_TELEMETRY=1` 有設嗎？
2. `OTEL_EXPORTER_OTLP_ENDPOINT` 設對了嗎？（注意 http vs https）
3. Collector 的 port 有 expose 嗎？
4. 如果用 gRPC，確認設定 `OTEL_EXPORTER_OTLP_PROTOCOL=grpc`

**Q: Collector logs 一片空白？**

A: 可能是 exporter 設定問題，先用 `debug` exporter 確認有收到資料。
（注意：舊版的 `logging` exporter 已在 v0.111.0 移除！）

---

## 第五章：完整範例配置

### 5.1 本地開發 Debug 配置

```bash
# 最簡單的 debug 配置 - 直接印在 terminal
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=console
export OTEL_LOGS_EXPORTER=console
export OTEL_METRIC_EXPORT_INTERVAL=10000  # 10 秒看一次

# 然後跑 claude
claude
```

### 5.2 送到本地 OTEL Collector

```bash
# 設定 telemetry
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp

# 設定 OTLP endpoint
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 標記資料來源
export OTEL_RESOURCE_ATTRIBUTES="team=dev,env=local"
export OTEL_SERVICE_NAME="claude-code-local"

claude
```

### 5.3 企業級生產配置

```bash
# 基本設定
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp

# OTLP endpoint（公司內部 collector）
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector.company.internal:4317
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer ${OTEL_AUTH_TOKEN}"

# mTLS
export OTEL_EXPORTER_OTLP_CERTIFICATE=/etc/certs/ca.crt
export OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE=/etc/certs/client.crt
export OTEL_EXPORTER_OTLP_CLIENT_KEY=/etc/certs/client.key

# 隱私（不記錄 prompt 內容）
export OTEL_LOG_USER_PROMPTS=0
export OTEL_LOG_TOOL_CONTENT=0

# 標記
export OTEL_RESOURCE_ATTRIBUTES="department=engineering,team=platform,cost_center=ENG-123"
export OTEL_SERVICE_NAME="claude-code-prod"

# 效能調校
export OTEL_METRIC_EXPORT_INTERVAL=60000
export OTEL_BSP_MAX_QUEUE_SIZE=4096
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1  # 只記錄 10% traces

claude
```

---

## 第六章：會輸出什麼資料？

> 📋 **資料來源說明**
>
> 以下 Metrics 和 Events 名稱來自：
> 1. **Source code 分析** - Claude Code v2.1.1 cli.js
> 2. **實測驗證** - 使用 `console` exporter 觀察輸出
> 3. **官方文件** - [Anthropic Docs](https://code.claude.com/docs/en/monitoring-usage)
>
> 名稱可能隨版本更新而變更，以實際輸出為準。

### 6.1 Metrics（指標）

| Metric 名稱 | 說明 | 來源 |
|------------|------|------|
| `claude_code.session.count` | CLI session 數量 | 官方文件 + 實測 |
| `claude_code.token.usage` | Token 用量（分 input/output/cache） | 官方文件 + 實測 |
| `claude_code.cost.usage` | 成本（美金） | 官方文件 + 實測 |
| `claude_code.lines_of_code.count` | 修改的程式碼行數 | Source code 分析 |
| `claude_code.pull_request.count` | 建立的 PR 數量 | Source code 分析 |
| `claude_code.commit.count` | 提交的 commit 數量 | Source code 分析 |
| `claude_code.code_edit_tool.decision` | 工具權限決策 | Source code 分析 |
| `claude_code.active_time.total` | 活躍時間（秒） | Source code 分析 |

### 6.2 Events（事件）

| Event 名稱 | 說明 | 來源 |
|-----------|------|------|
| `claude_code.user_prompt` | 使用者送出 prompt | 官方文件 + 實測 |
| `claude_code.tool_result` | 工具執行完成 | Source code 分析 |
| `claude_code.api_request` | API 請求 | Source code 分析 |
| `claude_code.api_error` | API 錯誤 | Source code 分析 |
| `claude_code.tool_decision` | 工具權限決策 | Source code 分析 |

### 6.3 所有資料都會附帶的屬性

| 屬性名稱 | 說明 | 來源 |
|---------|------|------|
| `session.id` | Session ID | 官方文件 |
| `organization.id` | 組織 ID | 官方文件 |
| `user.account_uuid` | 使用者 UUID | 官方文件 |
| `app.version` | Claude Code 版本 | 實測 |
| `terminal.type` | 終端機類型 | Source code 分析 |

---

## 第七章：實驗紀錄 (´▽`)

### 7.1 實際測試結果

在 Claude Code Cloud 環境中測試：

```bash
# 已存在的 OTEL 環境變數
OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta

# Claude Code 相關環境變數
CLAUDE_CODE_ENABLE_TELEMETRY  # 可設定
CLAUDE_CODE_VERSION=2.1.1
CLAUDE_CODE_SESSION_ID=xxx
CLAUDE_CODE_REMOTE=true
```

**發現：**
1. Cloud 環境已經有預設的 telemetry 設定！
2. 有 `~/.claude/telemetry/` 目錄存放 failed events
3. 內部 telemetry 事件用 `tengu_*` 命名

### 7.2 從 Source Code 確認的環境變數

直接從 Claude Code v2.1.1 的 cli.js 挖出來的完整列表：

**OTEL 相關（40+ 個）：**
```
OTEL_METRICS_EXPORTER
OTEL_LOGS_EXPORTER
OTEL_TRACES_EXPORTER
OTEL_EXPORTER_OTLP_ENDPOINT
OTEL_EXPORTER_OTLP_PROTOCOL
OTEL_EXPORTER_OTLP_HEADERS
OTEL_EXPORTER_OTLP_COMPRESSION
OTEL_EXPORTER_OTLP_CERTIFICATE
OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE
OTEL_EXPORTER_OTLP_CLIENT_KEY
OTEL_EXPORTER_OTLP_INSECURE
OTEL_EXPORTER_OTLP_TIMEOUT
OTEL_EXPORTER_OTLP_METRICS_*
OTEL_EXPORTER_OTLP_LOGS_*
OTEL_EXPORTER_OTLP_TRACES_*
OTEL_EXPORTER_PROMETHEUS_HOST
OTEL_METRIC_EXPORT_INTERVAL
OTEL_LOGS_EXPORT_INTERVAL
OTEL_TRACES_EXPORT_INTERVAL
OTEL_LOG_USER_PROMPTS
OTEL_LOG_TOOL_CONTENT
OTEL_METRICS_INCLUDE_SESSION_ID
OTEL_METRICS_INCLUDE_ACCOUNT_UUID
OTEL_METRICS_INCLUDE_VERSION
OTEL_RESOURCE_ATTRIBUTES
OTEL_SERVICE_NAME
OTEL_BSP_*
OTEL_BLRP_*
OTEL_ATTRIBUTE_COUNT_LIMIT
OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT
OTEL_SPAN_*
OTEL_LOGRECORD_*
OTEL_TRACES_SAMPLER
OTEL_TRACES_SAMPLER_ARG
OTEL_SHUTDOWN_TIMEOUT_MS
```

---

## 第八章：常見問題 FAQ

### Q1: 為什麼我設了環境變數但沒看到輸出？

A: 檢查這些：
1. `CLAUDE_CODE_ENABLE_TELEMETRY=1` 有設嗎？
2. `OTEL_METRICS_EXPORTER` 或 `OTEL_LOGS_EXPORTER` 有設嗎？
3. 如果用 `otlp`，endpoint 有跑起來嗎？

### Q2: Console exporter 輸出太多怎麼辦？

A: 調整匯出間隔：
```bash
export OTEL_METRIC_EXPORT_INTERVAL=60000  # 1 分鐘
```

### Q3: 企業環境要怎麼集中管理？

A: 用 **managed settings** 集中管理！

**檔案位置：**

| 平台 | 路徑 |
|------|------|
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` |
| Linux | `/etc/claude-code/managed-settings.json` |
| Windows | `%ProgramData%\ClaudeCode\managed-settings.json` |

**設定範例：**
```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://company-collector:4317",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer ${COMPANY_OTEL_TOKEN}"
  }
}
```

> 📋 **注意：** Managed settings 優先級最高，會覆蓋使用者的個人設定。
> 這是 IT 部門強制配置的好方法！(´▽`)

### Q4: 怎麼確認資料有送出去？

A:
1. 先用 `console` exporter 確認有輸出
2. 檢查 OTLP collector 的 logs
3. 看 `~/.claude/telemetry/` 有沒有 failed events

---

## 結語

Claude Code 的 OTEL 支援超完整的！( •̀ω•́ )✧

從這次實驗學到：
1. **Opt-in 設計** - 預設不開，保護隱私
2. **標準 OTEL** - 用標準協定，整合任何 observability 平台
3. **彈性配置** - 從 debug 到 enterprise 都能搞定
4. **隱私優先** - 敏感資料有保護機制

下次老闆問 Claude Code 花多少錢，你就能自信地回答啦！

```
老闘：Claude Code 這個月用多少？
你：根據 Grafana 儀表板顯示... (打開筆電)
    本月 token 用量 5,678,901
    成本 $56.78
    主要用在 code review 任務
老闆：(´▽`) 很好！
你：( •̀ω•́ )✧
```

---

## 參考資源

- [Claude Code 官方文件 - Monitoring](https://code.claude.com/docs/en/monitoring-usage)
- [OpenTelemetry 官方文件](https://opentelemetry.io/docs/)
- [SigNoz - Claude Code Monitoring Guide](https://signoz.io/docs/claude-code-monitoring/)

---

*寫於 Claude Code Cloud 環境，邊寫邊學 (´▽`)*
