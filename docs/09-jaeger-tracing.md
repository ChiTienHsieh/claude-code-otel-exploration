# Jaeger & Distributed Tracing Guide

> **Last Updated**: 2026-02-03

A comprehensive guide to using Jaeger for distributed tracing with Claude Code OTEL integration.

## Table of Contents

- [Introduction to Jaeger](#introduction-to-jaeger)
- [Setup and Configuration](#setup-and-configuration)
- [Understanding Traces](#understanding-traces)
- [Jaeger UI Guide](#jaeger-ui-guide)
- [Advanced Queries](#advanced-queries)
- [Performance Analysis](#performance-analysis)
- [Alerting on Traces](#alerting-on-traces)
- [Best Practices](#best-practices)

## Introduction to Jaeger

### What is Jaeger?

Jaeger is an open-source distributed tracing platform that helps:

- Monitor and troubleshoot distributed systems
- Track request flows across services
- Identify performance bottlenecks
- Analyze root causes of errors

### Jaeger Architecture

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

## Setup and Configuration

### Docker Compose Setup

```yaml
# docker-compose.yaml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:1.54
    ports:
      - "6831:6831/udp"   # Jaeger Thrift compact
      - "6832:6832/udp"   # Jaeger Thrift binary
      - "5778:5778"       # Config server
      - "16686:16686"     # UI
      - "4317:4317"       # OTLP gRPC
      - "4318:4318"       # OTLP HTTP
      - "14250:14250"     # Model/Collector gRPC
      - "14268:14268"     # Jaeger Thrift HTTP
      - "14269:14269"     # Health check
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
      - "4317:4317"
      - "4318:4318"
      - "8888:8888"
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    depends_on:
      jaeger:
        condition: service_healthy

volumes:
  jaeger-data:
```

### OTEL Collector Configuration for Jaeger

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

### Claude Code Configuration for Tracing

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

## Understanding Traces

### Trace Concepts

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

### Span Attributes Reference

| Attribute | Description | Example |
|-----------|-------------|---------|
| `service.name` | Service identifier | `claude-code` |
| `service.version` | Version | `2.1.1` |
| `session.id` | Session identifier | `abc123def456` |
| `user.id` | User identifier | `user@example.com` |
| `tool.name` | Tool being executed | `Bash`, `Read`, `Write` |
| `api.model` | AI model used | `claude-sonnet-4-20250514` |
| `api.tokens.input` | Input token count | `500` |
| `api.tokens.output` | Output token count | `1200` |
| `error` | Whether span errored | `true`/`false` |
| `error.message` | Error description | `Command failed` |

## Jaeger UI Guide

### Accessing the UI

```bash
# Start Jaeger
docker compose up -d jaeger

# Open UI
open http://localhost:16686
```

### Search Panel

#### Basic Search

1. **Service**: Select `claude-code`
2. **Operation**: Select specific operation or `all`
3. **Tags**: Filter by attributes
4. **Lookback**: Time range (1h, 2h, 1d, custom)
5. **Min/Max Duration**: Filter by latency

#### Tag Queries

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

### Trace View

#### Timeline View

```
[▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] claude_code.session (45s)
  [▓▓]                            claude_code.prompt (100ms)
     [▓▓▓▓▓▓▓▓▓]                  claude_code.api_request (2.5s)
         [▓▓▓▓▓▓▓▓]               anthropic.api.messages.create (2.4s)
                    [▓▓]          claude_code.tool_execution (500ms)
                      [▓]         tool.bash.execute (480ms)
```

#### Span Details

Click on a span to see:
- **Tags**: Key-value attributes
- **Process**: Service information
- **Logs**: Events/annotations within span
- **Warnings**: Detected issues

### Compare Traces

1. Select multiple traces in search results
2. Click "Compare" button
3. View side-by-side timeline comparison
4. Identify differences in timing and structure

## Advanced Queries

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

### Python Query Script

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

## Performance Analysis

### Identifying Bottlenecks

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

### Latency Analysis

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

### Service Dependency Graph

```bash
# Get dependencies
curl "http://localhost:16686/api/dependencies?endTs=$(date +%s)000&lookback=86400000"
```

## Alerting on Traces

### Prometheus Metrics from Jaeger

Jaeger exposes metrics that can be scraped by Prometheus:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'jaeger'
    static_configs:
      - targets: ['jaeger:14269']
```

### Alert Rules

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

### Grafana Alerts on Trace Data

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

## Best Practices

### 1. Meaningful Span Names

```
Good: claude_code.tool.bash.execute
Bad:  execute

Good: anthropic.api.messages.create
Bad:  api_call
```

### 2. Essential Tags

Always include:
- `service.name`
- `service.version`
- `error` (if applicable)
- `error.message` (if error)

### 3. Sampling Strategy

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

### 4. Trace Context Propagation

```bash
# Ensure context flows across services
export OTEL_PROPAGATORS=tracecontext,baggage,b3

# Manual propagation if needed
export TRACEPARENT="00-{trace_id}-{span_id}-01"
```

### 5. Storage Considerations

| Backend | Use Case | Retention |
|---------|----------|-----------|
| Memory | Development | Session only |
| Badger | Single node | Days |
| Elasticsearch | Production | Weeks |
| Cassandra | Large scale | Months |

```yaml
# Elasticsearch backend
environment:
  - SPAN_STORAGE_TYPE=elasticsearch
  - ES_SERVER_URLS=http://elasticsearch:9200
  - ES_INDEX_PREFIX=jaeger
```

### 6. Trace Retention

```yaml
# Automatic cleanup (Elasticsearch)
environment:
  - ES_MAX_SPAN_AGE=168h  # 7 days
```

## Troubleshooting

### No Traces Appearing

```bash
# Check Jaeger health
curl http://localhost:14269/

# Check collector is receiving
curl http://localhost:14269/metrics | grep jaeger_collector

# Verify OTEL exporter
export OTEL_TRACES_EXPORTER=console,otlp
claude "test"  # Should see trace output
```

### Incomplete Traces

- Check all services have same `OTEL_PROPAGATORS`
- Verify trace context headers are forwarded
- Check sampling configuration isn't dropping spans

### High Latency in UI

- Add indexes to storage backend
- Increase Jaeger query resources
- Reduce trace retention period

## Related Documentation

- [Traces Deep Dive](./01-traces-deep-dive.md)
- [Production Deployment Guide](./02-production-deployment.md)
- [Troubleshooting Playbook](./04-troubleshooting.md)
