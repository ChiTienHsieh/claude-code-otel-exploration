# 替代 Backend 整合

> **最後更新**: 2026-02-03

本指南提供將 Claude Code OTEL telemetry 連接到各種可觀測性 backend 的詳細整合說明。

## 目錄

- [概覽](#概覽)
- [Datadog](#datadog)
- [Grafana Cloud](#grafana-cloud)
- [AWS CloudWatch](#aws-cloudwatch)
- [Google Cloud Operations](#google-cloud-operations)
- [New Relic](#new-relic)
- [Honeycomb](#honeycomb)
- [Splunk](#splunk)
- [Elastic APM](#elastic-apm)
- [SigNoz](#signoz)
- [比較表](#比較表)

## 概覽

Claude Code 透過 OTLP（OpenTelemetry Protocol）匯出 telemetry。大多數現代可觀測性平台原生支援 OTLP 或透過 OTEL Collector 支援。

### 整合模式

```
模式 1：直接 OTLP
Claude Code → Backend（原生 OTLP 支援）

模式 2：透過 Collector
Claude Code → OTEL Collector → Backend（任何 protocol）

模式 3：供應商 Agent
Claude Code → OTEL Collector → Vendor Agent → Backend
```

## Datadog

### 直接 OTLP 整合

```bash
# 環境變數
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.datadoghq.com:4317
export OTEL_EXPORTER_OTLP_HEADERS="DD-API-KEY=<your-api-key>"
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# 用於 Datadog tagging 的 resource attributes
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code,env=production,team=engineering"
```

### 透過 OTEL Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 10s

  # 轉換為 Datadog 格式
  transform:
    metric_statements:
      - context: datapoint
        statements:
          - set(attributes["host"], resource.attributes["host.name"])

exporters:
  datadog:
    api:
      key: ${DD_API_KEY}
      site: datadoghq.com  # 或 datadoghq.eu、us3.datadoghq.com 等
    metrics:
      histograms:
        mode: distributions
      sums:
        cumulative_monotonic_mode: to_delta
    traces:
      span_name_as_resource_name: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch, transform]
      exporters: [datadog]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog]
```

### Datadog Dashboard JSON

```json
{
  "title": "Claude Code Monitoring",
  "widgets": [
    {
      "definition": {
        "title": "Token Usage",
        "type": "timeseries",
        "requests": [
          {
            "q": "sum:claude_code.tokens.input{*}.as_count()",
            "display_type": "bars",
            "style": {"palette": "dog_classic"}
          }
        ]
      }
    },
    {
      "definition": {
        "title": "Cost by Team",
        "type": "toplist",
        "requests": [
          {
            "q": "top(sum:claude_code.cost.total{*} by {team}, 10, 'sum', 'desc')"
          }
        ]
      }
    }
  ]
}
```

## Grafana Cloud

### 直接 OTLP 到 Grafana Cloud

```bash
# 從 Grafana Cloud Portal 取得憑證
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp-gateway-prod-us-central-0.grafana.net/otlp
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64-encoded-instance-id:token>"
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

### 透過 OTEL Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  otlphttp/grafana:
    endpoint: https://otlp-gateway-prod-us-central-0.grafana.net/otlp
    headers:
      Authorization: "Basic ${GRAFANA_CLOUD_TOKEN}"

  prometheusremotewrite/grafana:
    endpoint: https://prometheus-prod-us-central-0.grafana.net/api/prom/push
    headers:
      Authorization: "Bearer ${GRAFANA_CLOUD_TOKEN}"

  loki/grafana:
    endpoint: https://logs-prod-us-central-0.grafana.net/loki/api/v1/push
    headers:
      Authorization: "Bearer ${GRAFANA_CLOUD_TOKEN}"

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite/grafana]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/grafana]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki/grafana]
```

## AWS CloudWatch

### 透過 OTEL Collector 與 AWS Exporter

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 60s
    send_batch_size: 1000

exporters:
  awsemf:
    namespace: ClaudeCode
    region: us-east-1
    log_group_name: /aws/claude-code/metrics
    log_stream_name: metrics
    dimension_rollup_option: NoDimensionRollup
    resource_to_telemetry_conversion:
      enabled: true

  awsxray:
    region: us-east-1
    indexed_attributes:
      - service.name
      - team
      - environment

  awscloudwatchlogs:
    log_group_name: /aws/claude-code/logs
    log_stream_name: application
    region: us-east-1

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsemf]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsxray]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [awscloudwatchlogs]
```

### 所需的 IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/aws/claude-code/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "cloudwatch:namespace": "ClaudeCode"
        }
      }
    }
  ]
}
```

### CloudWatch Dashboard

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "title": "Claude Code Token Usage",
        "metrics": [
          ["ClaudeCode", "claude_code.tokens.input", {"stat": "Sum"}],
          ["ClaudeCode", "claude_code.tokens.output", {"stat": "Sum"}]
        ],
        "period": 300,
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "Claude Code Sessions",
        "metrics": [
          ["ClaudeCode", "claude_code.session.count", {"stat": "Sum"}]
        ],
        "period": 300,
        "region": "us-east-1"
      }
    }
  ]
}
```

## Google Cloud Operations

### 透過 OTEL Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  googlecloud:
    project: your-gcp-project-id
    metric:
      prefix: custom.googleapis.com/claude_code
    log:
      default_log_name: claude-code

  googlecloudmonitoring:
    project: your-gcp-project-id

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [googlecloud]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [googlecloud]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [googlecloud]
```

### GCP Service Account 設置

```bash
# 建立 service account
gcloud iam service-accounts create otel-collector \
    --display-name="OTEL Collector Service Account"

# 授予權限
gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:otel-collector@your-project-id.iam.gserviceaccount.com" \
    --role="roles/monitoring.metricWriter"

gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:otel-collector@your-project-id.iam.gserviceaccount.com" \
    --role="roles/cloudtrace.agent"

gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:otel-collector@your-project-id.iam.gserviceaccount.com" \
    --role="roles/logging.logWriter"

# 建立金鑰
gcloud iam service-accounts keys create key.json \
    --iam-account=otel-collector@your-project-id.iam.gserviceaccount.com

# 設定環境變數
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
```

## New Relic

### 直接 OTLP 整合

```bash
# 環境變數
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.nr-data.net:4317
export OTEL_EXPORTER_OTLP_HEADERS="api-key=<your-license-key>"
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Resource attributes
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code"
```

### 透過 OTEL Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  otlp/newrelic:
    endpoint: otlp.nr-data.net:4317
    headers:
      api-key: ${NEW_RELIC_LICENSE_KEY}

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/newrelic]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/newrelic]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/newrelic]
```

### New Relic NRQL 查詢

```sql
-- 隨時間的 token 使用量
SELECT sum(claude_code.tokens.input), sum(claude_code.tokens.output)
FROM Metric
WHERE service.name = 'claude-code'
TIMESERIES AUTO

-- 按團隊的成本
SELECT sum(claude_code.cost.total)
FROM Metric
FACET team
SINCE 1 week ago

-- 按環境的 session 數量
SELECT count(*)
FROM Metric
WHERE metricName = 'claude_code.session.count'
FACET environment
```

## Honeycomb

### 直接 OTLP 整合

```bash
# 環境變數
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io:443
export OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=<your-api-key>"
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# 設定 dataset
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code"
```

### 透過 OTEL Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  otlp/honeycomb:
    endpoint: api.honeycomb.io:443
    headers:
      x-honeycomb-team: ${HONEYCOMB_API_KEY}
      x-honeycomb-dataset: claude-code

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/honeycomb]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/honeycomb]
```

## Splunk

### Splunk Observability Cloud

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  splunk_hec:
    token: ${SPLUNK_HEC_TOKEN}
    endpoint: https://splunk-hec.example.com:8088/services/collector
    source: claude-code
    sourcetype: otel
    index: main

  signalfx:
    access_token: ${SPLUNK_ACCESS_TOKEN}
    realm: us1
    api_url: https://api.us1.signalfx.com
    ingest_url: https://ingest.us1.signalfx.com

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [signalfx]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [signalfx]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [splunk_hec]
```

## Elastic APM

### 透過 OTEL Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  elasticsearch:
    endpoints: ["https://elasticsearch:9200"]
    user: elastic
    password: ${ELASTIC_PASSWORD}
    traces_index: traces-claude-code
    logs_index: logs-claude-code

  # 對於 metrics，使用 Prometheus remote write 到 Elasticsearch
  prometheusremotewrite:
    endpoint: https://elasticsearch:9200/_prometheus/api/v1/write
    auth:
      authenticator: basicauth
    tls:
      insecure_skip_verify: true

extensions:
  basicauth:
    client_auth:
      username: elastic
      password: ${ELASTIC_PASSWORD}

service:
  extensions: [basicauth]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite]
```

## SigNoz

SigNoz 是一個原生支援 OTLP 的開源替代方案。

### 直接 OTLP 整合

```bash
# 環境變數（自託管 SigNoz）
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://signoz-otel-collector:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# 對於 SigNoz Cloud
export OTEL_EXPORTER_OTLP_ENDPOINT=https://ingest.signoz.cloud:443
export OTEL_EXPORTER_OTLP_HEADERS="signoz-access-token=<your-token>"
```

### 使用 SigNoz 的 Docker Compose

```yaml
# 注意：`version` 欄位在 Docker Compose v2+ 已棄用，不再需要
services:
  signoz:
    image: signoz/signoz:latest
    ports:
      - "3301:3301"  # UI
      - "4317:4317"  # OTLP gRPC
      - "4318:4318"  # OTLP HTTP
    volumes:
      - signoz-data:/var/lib/signoz

volumes:
  signoz-data:
```

## 比較表

| Backend | 原生 OTLP | Metrics | Traces | Logs | 成本 | 易用性 |
|---------|-------------|---------|--------|------|------|------|
| Datadog | 是 | 完整 | 完整 | 完整 | $$$ | 簡單 |
| Grafana Cloud | 是 | 完整 | 完整 | 完整 | $$ | 簡單 |
| AWS CloudWatch | Collector | 完整 | 透過 X-Ray | 完整 | $$ | 中等 |
| GCP Operations | Collector | 完整 | 完整 | 完整 | $$ | 中等 |
| New Relic | 是 | 完整 | 完整 | 完整 | $$$ | 簡單 |
| Honeycomb | 是 | 有限 | 完整 | 完整 | $$ | 簡單 |
| Splunk | Collector | 完整 | 完整 | 完整 | $$$ | 中等 |
| Elastic | Collector | 完整 | 完整 | 完整 | $-$$$ | 困難 |
| SigNoz | 是 | 完整 | 完整 | 完整 | $（開源） | 簡單 |

### 圖例說明
- **成本**：$ = 低、$$ = 中、$$$ = 高
- **易用性**：針對 Claude Code 設置的難易程度

## 最佳實踐

1. **從 Collector 開始**：在 production 環境中始終使用 OTEL Collector
2. **使用原生 OTLP**：優先選擇具有原生 OTLP 支援的 backend
3. **設定 Resource Attributes**：始終設定 `service.name` 和 `team`
4. **配置重試**：啟用失敗重試以確保可靠性
5. **監控 Pipeline**：匯出 collector metrics 以實現可觀測性

## 相關文件

- [Production 部署指南](./02-production-deployment.md)
- [疑難排解手冊](./04-troubleshooting.md)
- [安全加固指南](./11-security-hardening.md)
