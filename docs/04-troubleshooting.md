# 疑難排解手冊

> **最後更新**: 2026-02-03

這是一份診斷和解決 Claude Code OTEL 整合常見問題的完整指南。

## 目錄

- [快速診斷](#快速診斷)
- [常見問題](#常見問題)
- [除錯技術](#除錯技術)
- [Collector 問題](#collector-問題)
- [Backend 問題](#backend-問題)
- [效能問題](#效能問題)
- [網路問題](#網路問題)

## 快速診斷

### 健康檢查腳本

```bash
#!/bin/bash
set -euo pipefail
# claude-otel-health-check.sh

echo "=== Claude Code OTEL Health Check ==="
echo ""

# 1. Check environment variables
echo "1. Environment Variables:"
echo "   CLAUDE_CODE_ENABLE_TELEMETRY: ${CLAUDE_CODE_ENABLE_TELEMETRY:-NOT SET}"
echo "   OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT:-NOT SET}"
echo "   OTEL_METRICS_EXPORTER: ${OTEL_METRICS_EXPORTER:-NOT SET}"
echo "   OTEL_LOGS_EXPORTER: ${OTEL_LOGS_EXPORTER:-NOT SET}"
echo ""

# 2. Check OTEL Collector connectivity
echo "2. OTEL Collector Connectivity:"
ENDPOINT=${OTEL_EXPORTER_OTLP_ENDPOINT:-http://localhost:4317}
if curl -s --connect-timeout 2 "${ENDPOINT%:*}:13133" > /dev/null 2>&1; then
    echo "   ✓ Collector reachable at $ENDPOINT"
else
    echo "   ✗ Cannot reach collector at $ENDPOINT"
fi
echo ""

# 3. Check collector health endpoint
echo "3. Collector Health:"
HEALTH_RESPONSE=$(curl -s --connect-timeout 2 "${ENDPOINT%:*}:13133" 2>/dev/null)
if [[ $? -eq 0 ]]; then
    echo "   ✓ Health endpoint responding"
else
    echo "   ✗ Health endpoint not responding"
fi
echo ""

# 4. Check Docker containers (if applicable)
echo "4. Docker Containers:"
if command -v docker &> /dev/null; then
    docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "(otel|prometheus|grafana|jaeger)" || echo "   No OTEL containers found"
else
    echo "   Docker not installed"
fi
echo ""

# 5. Check Prometheus targets
echo "5. Prometheus Targets:"
PROM_TARGETS=$(curl -s "http://localhost:9090/api/v1/targets" 2>/dev/null | jq -r '.data.activeTargets[] | "\(.labels.job): \(.health)"' 2>/dev/null)
if [[ -n "$PROM_TARGETS" ]]; then
    echo "$PROM_TARGETS" | sed 's/^/   /'
else
    echo "   Cannot query Prometheus targets"
fi
echo ""

echo "=== Health Check Complete ==="
```

### 單行診斷命令

```bash
# 檢查 telemetry 是否啟用
env | grep -E "(CLAUDE|OTEL)" | sort

# 測試 OTLP endpoint 連線
curl -v http://localhost:4317 2>&1 | head -20

# 檢查 collector logs 中的錯誤
docker logs otel-collector 2>&1 | grep -i error | tail -20

# 驗證 Prometheus 是否正在抓取資料
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'

# 在 Prometheus 中檢查 Claude Code metrics
curl -s 'http://localhost:9090/api/v1/query?query=claude_code_session_count' | jq '.data.result'
```

## 常見問題

### 問題 1：沒有出現 Telemetry 資料

**症狀：**
- Grafana dashboards 顯示「No data」
- Prometheus 沒有 `claude_code_*` metrics
- OTEL Collector logs 中沒有輸出

**診斷步驟：**

```bash
# 步驟 1：驗證 telemetry 是否啟用
echo $CLAUDE_CODE_ENABLE_TELEMETRY
# 預期：1

# 步驟 2：檢查 exporter 配置
echo $OTEL_METRICS_EXPORTER
echo $OTEL_LOGS_EXPORTER
# 預期：otlp（或用於除錯的 console）

# 步驟 3：先用 console exporter 測試
export OTEL_METRICS_EXPORTER=console
export OTEL_LOGS_EXPORTER=console
claude "hello"
# 應該在終端機看到 JSON 輸出
```

**解決方案：**

```bash
# 方案 A：啟用 telemetry
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# 方案 B：修正 endpoint
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 方案 C：使用正確的 protocol
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc  # 或 http/protobuf
```

### 問題 2：連線到 OTEL Collector 被拒絕

**症狀：**
- 錯誤：「connection refused」
- Telemetry 已啟用但 backend 沒有資料

**診斷步驟：**

```bash
# 檢查 collector 是否正在執行
docker ps | grep otel-collector

# 檢查 collector port 綁定
docker port otel-collector

# 測試 gRPC endpoint
grpcurl -plaintext localhost:4317 list

# 測試 HTTP endpoint
curl -v http://localhost:4318/v1/metrics
```

**解決方案：**

```bash
# 方案 A：啟動 collector
docker compose up -d otel-collector

# 方案 B：修正 docker-compose.yml 中的 port mapping
# ports:
#   - "4317:4317"
#   - "4318:4318"

# 方案 C：使用 host.docker.internal 進行 Docker 到主機的通訊
export OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317
```

### 問題 3：Metrics 缺少屬性

**症狀：**
- 團隊或環境標籤沒有出現
- Resource attributes 沒有傳播

**診斷步驟：**

```bash
# 檢查 resource attributes
echo $OTEL_RESOURCE_ATTRIBUTES

# 在 Prometheus 中查詢標籤是否存在
curl -s 'http://localhost:9090/api/v1/query?query=claude_code_session_count' | jq '.data.result[].metric'
```

**解決方案：**

```bash
# 解決方案：正確設定 resource attributes
export OTEL_RESOURCE_ATTRIBUTES="team=engineering,environment=production,service.name=claude-code"

# 對於 managed 設定（settings.json）：
{
  "env": {
    "OTEL_RESOURCE_ATTRIBUTES": "team=engineering,environment=production"
  }
}
```

### 問題 4：Collector 記憶體使用過高

**症狀：**
- Collector OOM kills
- 記憶體消耗過高
- Metric 處理緩慢

**診斷步驟：**

```bash
# 檢查 collector 記憶體使用
docker stats otel-collector

# 檢查佇列積壓
curl -s http://localhost:8888/metrics | grep queue

# 檢查 batch processor metrics
curl -s http://localhost:8888/metrics | grep batch
```

**解決方案：**

```yaml
# 解決方案：新增 memory limiter processor
processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 75
    spike_limit_percentage: 25
  batch:
    timeout: 5s
    send_batch_size: 1024
    send_batch_max_size: 2048

service:
  pipelines:
    metrics:
      processors: [memory_limiter, batch]
```

### 問題 5：SSL/TLS 憑證錯誤

**症狀：**
- 「certificate verify failed」
- 「x509: certificate signed by unknown authority」

**診斷步驟：**

```bash
# 檢查憑證有效性
openssl s_client -connect collector.example.com:4317 </dev/null 2>/dev/null | openssl x509 -text | head -20

# 使用 insecure flag 測試（僅用於除錯）
export OTEL_EXPORTER_OTLP_INSECURE=true
claude "test"
```

**解決方案：**

```bash
# 方案 A：提供 CA 憑證
export OTEL_EXPORTER_OTLP_CERTIFICATE=/path/to/ca.crt

# 方案 B：對於 mTLS，提供客戶端憑證
export OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE=/path/to/client.crt
export OTEL_EXPORTER_OTLP_CLIENT_KEY=/path/to/client.key

# 方案 C：僅用於內部測試時使用 insecure
export OTEL_EXPORTER_OTLP_INSECURE=true
```

### 問題 6：重複的 Metrics

**症狀：**
- 相同的 metric 被報告多次
- Dashboards 中的計數被誇大

**診斷步驟：**

```bash
# 檢查是否有多個 exporters
echo $OTEL_METRICS_EXPORTER
# 不應該有重複，如 "otlp,otlp"

# 在 Prometheus 中檢查重複的 targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets | length'
```

**解決方案：**

```bash
# 方案 A：使用單一 exporter
export OTEL_METRICS_EXPORTER=otlp

# 方案 B：在 Prometheus 中配置去重
# prometheus.yml
scrape_configs:
  - job_name: 'otel-collector'
    honor_labels: true
    static_configs:
      - targets: ['otel-collector:8889']
```

## 除錯技術

### 啟用 Debug Logging

```bash
# 所有信號的 console 輸出
export OTEL_METRICS_EXPORTER=console,otlp
export OTEL_LOGS_EXPORTER=console,otlp
export OTEL_TRACES_EXPORTER=console,otlp

# 執行 Claude Code 並觀察輸出
claude "test prompt" 2>&1 | tee claude-debug.log
```

### Collector Debug 模式

```yaml
# otel-collector-config.yaml
exporters:
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200

service:
  pipelines:
    metrics:
      exporters: [debug, prometheusremotewrite]
  telemetry:
    logs:
      level: debug
```

### 追蹤 OTLP 流量

```bash
# 使用 tcpdump 擷取 OTLP 流量
sudo tcpdump -i any -w otlp-traffic.pcap port 4317

# 使用 Wireshark 或 tshark 分析
tshark -r otlp-traffic.pcap -Y "tcp.port == 4317"
```

### 手動測試 Exporter

```bash
# 透過 OTLP HTTP 發送測試 metrics
curl -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "test"}}]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "test_metric",
          "gauge": {
            "dataPoints": [{"asInt": "42", "timeUnixNano": "'$(date +%s)000000000'"}]
          }
        }]
      }]
    }]
  }'
```

## Collector 問題

### Collector 無法啟動

```bash
# 檢查配置語法
docker run --rm -v $(pwd)/otel-collector-config.yaml:/etc/otel/config.yaml \
  otel/opentelemetry-collector-contrib:latest validate --config=/etc/otel/config.yaml

# 檢查 port 衝突
lsof -i :4317
lsof -i :4318

# 檢視啟動 logs
docker logs otel-collector 2>&1 | head -50
```

### Pipeline 處理失敗

```bash
# 檢查 processor 錯誤
docker logs otel-collector 2>&1 | grep -i "processor"

# 監控丟棄的 metrics
curl -s http://localhost:8888/metrics | grep -E "(dropped|failed)"
```

## Backend 問題

### Prometheus 未接收資料

```bash
# 檢查 remote write 狀態
curl -s http://localhost:9090/api/v1/status/runtimeinfo | jq '.data.reloadConfigSuccess'

# 驗證 scrape targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastError: .lastError}'

# 檢查儲存狀態
curl -s http://localhost:9090/api/v1/status/tsdb | jq '.data'
```

### Grafana Dashboard 空白

> **安全警告**：以下範例使用預設憑證（admin/admin）。
> **在 Production 環境中請務必更改** - 切勿在非開發環境中使用預設憑證。

```bash
# 測試 Prometheus datasource
curl -s "http://admin:admin@localhost:3000/api/datasources/proxy/1/api/v1/query?query=up" | jq '.status'

# 檢查 dashboard provisioning
docker logs grafana 2>&1 | grep -i "dashboard"

# 驗證 datasource 配置
curl -s "http://admin:admin@localhost:3000/api/datasources" | jq '.[].name'
```

## 效能問題

### Claude Code 高延遲

```bash
# 檢查 telemetry 是否增加額外開銷
# 停用並比較
unset CLAUDE_CODE_ENABLE_TELEMETRY
time claude "test"

# 重新啟用並比較
export CLAUDE_CODE_ENABLE_TELEMETRY=1
time claude "test"
```

### Collector 背壓

```yaml
# 新增 sending queue 處理背壓
exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 10000
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
```

## 網路問題

### DNS 解析失敗

```bash
# 測試 DNS 解析
nslookup otel-collector.example.com

# 直接使用 IP 位址
export OTEL_EXPORTER_OTLP_ENDPOINT=http://10.0.0.50:4317
```

### 防火牆阻擋

```bash
# 檢查 ports 是否開放
nc -zv localhost 4317
nc -zv localhost 4318

# 透過防火牆測試
telnet collector.example.com 4317
```

### Proxy 配置

```bash
# 如果在企業 proxy 後面
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.internal
```

## 錯誤參考

| 錯誤訊息 | 原因 | 解決方案 |
|--------------|-------|----------|
| `connection refused` | Collector 未執行 | 啟動 collector |
| `deadline exceeded` | 網路逾時 | 檢查網路/防火牆 |
| `certificate verify failed` | TLS 憑證問題 | 配置憑證 |
| `resource exhausted` | Collector 過載 | 增加記憶體限制 |
| `invalid endpoint` | URL 格式錯誤 | 檢查 endpoint 格式 |
| `permission denied` | 檔案/port 存取權限 | 檢查權限 |
| `unknown authority` | CA 不受信任 | 新增 CA 憑證 |

## 支援資源

- [OpenTelemetry 文件](https://opentelemetry.io/docs/)
- [OTEL Collector 疑難排解](https://opentelemetry.io/docs/collector/troubleshooting/)
- [Prometheus 疑難排解](https://prometheus.io/docs/prometheus/latest/troubleshooting/)
- [Grafana 疑難排解](https://grafana.com/docs/grafana/latest/troubleshooting/)

## 相關文件

- [Production 部署指南](./02-production-deployment.md)
- [安全加固指南](./11-security-hardening.md)
- [OTEL Collector 設置](../TUTORIAL_OTEL_zh-TW.md)
