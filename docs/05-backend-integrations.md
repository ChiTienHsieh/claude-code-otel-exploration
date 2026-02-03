# Alternative Backend Integrations

> **Last Updated**: 2026-02-03

This guide provides detailed integration instructions for connecting Claude Code OTEL telemetry to various observability backends.

## Table of Contents

- [Overview](#overview)
- [Datadog](#datadog)
- [Grafana Cloud](#grafana-cloud)
- [AWS CloudWatch](#aws-cloudwatch)
- [Google Cloud Operations](#google-cloud-operations)
- [New Relic](#new-relic)
- [Honeycomb](#honeycomb)
- [Splunk](#splunk)
- [Elastic APM](#elastic-apm)
- [SigNoz](#signoz)
- [Comparison Matrix](#comparison-matrix)

## Overview

Claude Code exports telemetry via OTLP (OpenTelemetry Protocol). Most modern observability platforms support OTLP natively or through an OTEL Collector.

### Integration Patterns

```
Pattern 1: Direct OTLP
Claude Code → Backend (native OTLP support)

Pattern 2: Via Collector
Claude Code → OTEL Collector → Backend (any protocol)

Pattern 3: Vendor Agent
Claude Code → OTEL Collector → Vendor Agent → Backend
```

## Datadog

### Direct OTLP Integration

```bash
# Environment variables
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.datadoghq.com:4317
export OTEL_EXPORTER_OTLP_HEADERS="DD-API-KEY=<your-api-key>"
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Resource attributes for Datadog tagging
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code,env=production,team=engineering"
```

### Via OTEL Collector

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

  # Transform to Datadog format
  transform:
    metric_statements:
      - context: datapoint
        statements:
          - set(attributes["host"], resource.attributes["host.name"])

exporters:
  datadog:
    api:
      key: ${DD_API_KEY}
      site: datadoghq.com  # or datadoghq.eu, us3.datadoghq.com, etc.
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

### Direct OTLP to Grafana Cloud

```bash
# Get credentials from Grafana Cloud Portal
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp-gateway-prod-us-central-0.grafana.net/otlp
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64-encoded-instance-id:token>"
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

### Via OTEL Collector

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

### Via OTEL Collector with AWS Exporter

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

### IAM Policy Required

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

### Via OTEL Collector

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

### GCP Service Account Setup

```bash
# Create service account
gcloud iam service-accounts create otel-collector \
    --display-name="OTEL Collector Service Account"

# Grant permissions
gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:otel-collector@your-project-id.iam.gserviceaccount.com" \
    --role="roles/monitoring.metricWriter"

gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:otel-collector@your-project-id.iam.gserviceaccount.com" \
    --role="roles/cloudtrace.agent"

gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:otel-collector@your-project-id.iam.gserviceaccount.com" \
    --role="roles/logging.logWriter"

# Create key
gcloud iam service-accounts keys create key.json \
    --iam-account=otel-collector@your-project-id.iam.gserviceaccount.com

# Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
```

## New Relic

### Direct OTLP Integration

```bash
# Environment variables
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.nr-data.net:4317
export OTEL_EXPORTER_OTLP_HEADERS="api-key=<your-license-key>"
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Resource attributes
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code"
```

### Via OTEL Collector

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

### New Relic NRQL Queries

```sql
-- Token usage over time
SELECT sum(claude_code.tokens.input), sum(claude_code.tokens.output)
FROM Metric
WHERE service.name = 'claude-code'
TIMESERIES AUTO

-- Cost by team
SELECT sum(claude_code.cost.total)
FROM Metric
FACET team
SINCE 1 week ago

-- Session count by environment
SELECT count(*)
FROM Metric
WHERE metricName = 'claude_code.session.count'
FACET environment
```

## Honeycomb

### Direct OTLP Integration

```bash
# Environment variables
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io:443
export OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=<your-api-key>"
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Set dataset
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code"
```

### Via OTEL Collector

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

### Via OTEL Collector

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

  # For metrics, use Prometheus remote write to Elasticsearch
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

SigNoz is an open-source alternative that natively supports OTLP.

### Direct OTLP Integration

```bash
# Environment variables (self-hosted SigNoz)
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://signoz-otel-collector:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# For SigNoz Cloud
export OTEL_EXPORTER_OTLP_ENDPOINT=https://ingest.signoz.cloud:443
export OTEL_EXPORTER_OTLP_HEADERS="signoz-access-token=<your-token>"
```

### Docker Compose with SigNoz

```yaml
version: '3.8'
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

## Comparison Matrix

| Backend | OTLP Native | Metrics | Traces | Logs | Cost | Ease |
|---------|-------------|---------|--------|------|------|------|
| Datadog | Yes | Full | Full | Full | $$$ | Easy |
| Grafana Cloud | Yes | Full | Full | Full | $$ | Easy |
| AWS CloudWatch | Collector | Full | Via X-Ray | Full | $$ | Medium |
| GCP Operations | Collector | Full | Full | Full | $$ | Medium |
| New Relic | Yes | Full | Full | Full | $$$ | Easy |
| Honeycomb | Yes | Limited | Full | Full | $$ | Easy |
| Splunk | Collector | Full | Full | Full | $$$ | Medium |
| Elastic | Collector | Full | Full | Full | $-$$$ | Hard |
| SigNoz | Yes | Full | Full | Full | $ (OSS) | Easy |

### Legend
- **Cost**: $ = Low, $$ = Medium, $$$ = High
- **Ease**: How easy to set up for Claude Code specifically

## Best Practices

1. **Start with Collector**: Always use OTEL Collector for production
2. **Use Native OTLP**: Prefer backends with native OTLP support
3. **Set Resource Attributes**: Always set `service.name` and `team`
4. **Configure Retry**: Enable retry on failure for reliability
5. **Monitor the Pipeline**: Export collector metrics for observability

## Related Documentation

- [Production Deployment Guide](./02-production-deployment.md)
- [Troubleshooting Playbook](./04-troubleshooting.md)
- [Security Hardening Guide](./11-security-hardening.md)
