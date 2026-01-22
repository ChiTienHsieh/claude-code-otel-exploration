# Claude Code OTEL Verification Environment

這個目錄提供一個完整的 observability stack，讓你可以驗證 Claude Code 的 OpenTelemetry 輸出。

## 架構圖

```
┌─────────────────┐     OTLP      ┌──────────────────┐
│   Claude Code   │ ───────────── │  OTEL Collector  │
│   (你的電腦)     │   gRPC:4317   │   (Container)    │
└─────────────────┘               └────────┬─────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
            ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
            │  Prometheus  │      │    Jaeger    │      │   Console    │
            │  (Metrics)   │      │   (Traces)   │      │   (Debug)    │
            │  :9090       │      │   :16686     │      │              │
            └──────┬───────┘      └──────────────┘      └──────────────┘
                   │
                   ▼
            ┌──────────────┐
            │   Grafana    │
            │  (Dashboard) │
            │   :3000      │
            └──────────────┘
```

## Quick Start

### 1. 啟動 Observability Stack

```bash
# 進入 verify 目錄
cd verify

# 複製環境變數範例
cp .env.example .env

# 啟動所有服務
docker-compose up -d

# 確認服務都起來了
docker-compose ps
```

### 2. 設定 Claude Code 環境變數

在你要執行 Claude Code 的 terminal 設定這些環境變數：

```bash
# 啟用 telemetry
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# 指向本地 OTEL Collector
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# (可選) 同時輸出到 console 方便 debug
export OTEL_TRACES_EXPORTER=otlp,console
export OTEL_METRICS_EXPORTER=otlp,console
```

### 3. 執行 Claude Code

```bash
# 正常使用 Claude Code
claude

# 或執行特定指令
claude "hello, please list files in current directory"
```

### 4. 查看結果

| 服務 | URL | 說明 |
|------|-----|------|
| Grafana | http://localhost:3000 | Dashboard (admin/admin) |
| Prometheus | http://localhost:9090 | Metrics 查詢 |
| Jaeger | http://localhost:16686 | Distributed Traces |
| OTEL Collector Metrics | http://localhost:8888/metrics | Collector 自身狀態 |

## 驗證項目清單

### Basic Verification

- [ ] Docker compose 正常啟動（4 個 container）
- [ ] Grafana 可以登入 (admin/admin)
- [ ] Prometheus targets 都是 UP 狀態
- [ ] OTEL Collector logs 沒有錯誤

### Claude Code Integration

- [ ] 執行 Claude Code 後，Collector logs 有收到資料
- [ ] Prometheus 有 `claude_code_*` metrics
- [ ] Jaeger 有 traces 資料
- [ ] Grafana dashboard 顯示數據

## 常見問題

### Q: Collector 沒收到資料？

1. 確認環境變數有設定：
   ```bash
   echo $CLAUDE_CODE_ENABLE_TELEMETRY
   echo $OTEL_EXPORTER_OTLP_ENDPOINT
   ```

2. 檢查 Collector logs：
   ```bash
   docker-compose logs otel-collector
   ```

3. 確認 port 有開：
   ```bash
   curl -v http://localhost:4318/v1/traces
   # 應該回 405 Method Not Allowed（代表有在聽）
   ```

### Q: Prometheus 沒有 metrics？

1. 檢查 targets 狀態：
   - 開啟 http://localhost:9090/targets
   - 確認 `claude-code` job 是 UP

2. 直接查詢 Collector exporter：
   ```bash
   curl http://localhost:8889/metrics | grep claude
   ```

### Q: 想要更詳細的 logs？

修改 `otel-collector-config.yaml` 的 debug exporter：

```yaml
exporters:
  debug:
    verbosity: detailed  # 改成 detailed
```

然後重啟：
```bash
docker-compose restart otel-collector
```

## 清理

```bash
# 停止並移除 containers
docker-compose down

# 連 volumes 一起刪（清掉所有資料）
docker-compose down -v
```

## 檔案說明

| 檔案 | 說明 |
|------|------|
| `docker-compose.yml` | 定義所有服務 |
| `otel-collector-config.yaml` | OTEL Collector 設定 |
| `prometheus.yml` | Prometheus 抓取設定 |
| `.env.example` | 環境變數範例 |
| `grafana/provisioning/` | Grafana 自動配置 |
| `tasks/` | 驗證任務指南 |
