# OpenTelemetry Traces Deep Dive for Claude Code

> **Status**: Experimental/Unclear Support
> **Last Updated**: 2026-02-03

## Overview

This document explores the tracing capabilities in Claude Code's OpenTelemetry integration. Unlike Metrics and Logs which are officially documented, Traces support remains experimental.

## Current State of Traces Support

### What We Know

1. **Environment Variable Exists**: `OTEL_TRACES_EXPORTER` is recognized by Claude Code
2. **Official Documentation**: Only mentions Metrics and Logs explicitly
3. **Observed Behavior**: Claude Code may not actively generate trace spans

### Configuration

```bash
# Enable traces exporter (experimental)
export OTEL_TRACES_EXPORTER=otlp

# Or for debugging
export OTEL_TRACES_EXPORTER=console

# Multiple exporters
export OTEL_TRACES_EXPORTER=console,otlp
```

### Trace Sampling Configuration

```bash
# Sampling strategy
export OTEL_TRACES_SAMPLER=parentbased_traceidratio

# Sampling ratio (0.0 to 1.0)
export OTEL_TRACES_SAMPLER_ARG=0.1  # Sample 10% of traces

# Always sample (for debugging)
export OTEL_TRACES_SAMPLER=always_on

# Never sample
export OTEL_TRACES_SAMPLER=always_off
```

## Expected Trace Structure

If traces were fully supported, you would expect to see spans like:

### Hypothetical Span Hierarchy

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

### Standard Span Attributes

Based on OTEL semantic conventions, traces would include:

| Attribute | Description | Example |
|-----------|-------------|---------|
| `service.name` | Service identifier | `claude-code` |
| `service.version` | Claude Code version | `2.1.1` |
| `session.id` | Unique session ID | `abc123` |
| `user.id` | User identifier | (if enabled) |
| `tool.name` | Tool being executed | `Bash` |
| `api.model` | Model being used | `claude-sonnet-4-20250514` |

## Testing Traces Support

### Step 1: Configure OTEL Collector for Traces

Add to your `otel-collector-config.yaml`:

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

### Step 2: Enable Traces in Claude Code

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

### Step 3: Verify Trace Output

```bash
# Run Claude Code
claude "hello world"

# Check Jaeger UI
open http://localhost:16686

# Or check collector debug logs
docker logs otel-collector 2>&1 | grep -i "trace"
```

## Working with Traces Data

### Jaeger Queries

Once traces appear in Jaeger:

1. **Find Slow Operations**:
   - Sort by duration
   - Filter by `error=true`

2. **Service Dependencies**:
   - View service topology
   - Identify bottlenecks

3. **Compare Traces**:
   - Compare similar operations
   - Identify regression

### Correlating Traces with Logs

Use trace context propagation:

```bash
# Trace context in logs
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp

# Logs will include trace_id and span_id for correlation
```

## Implementing Custom Traces

If Claude Code doesn't generate traces natively, you can instrument wrapper scripts:

### Bash Wrapper with Trace Context

```bash
#!/bin/bash
# claude-traced.sh - Wrapper that adds trace context

# Generate trace ID (32 hex chars)
TRACE_ID=$(openssl rand -hex 16)
# Generate span ID (16 hex chars)
SPAN_ID=$(openssl rand -hex 8)

# Set trace parent header
export TRACEPARENT="00-${TRACE_ID}-${SPAN_ID}-01"

# Record start time
START_TIME=$(date +%s%N)

# Run Claude Code
claude "$@"
EXIT_CODE=$?

# Record end time
END_TIME=$(date +%s%N)
DURATION=$((($END_TIME - $START_TIME) / 1000000))

echo "Trace ID: $TRACE_ID, Duration: ${DURATION}ms"
exit $EXIT_CODE
```

### Python Wrapper with OpenTelemetry SDK

```python
#!/usr/bin/env python3
"""claude-traced.py - Python wrapper with full OTEL tracing"""

import subprocess
import sys
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# Configure tracer
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

Claude Code should respect standard trace context headers:

```bash
# Set parent trace context
export TRACEPARENT="00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01"

# Optional: trace state
export TRACESTATE="vendor1=value1,vendor2=value2"
```

### Baggage Propagation

```bash
# Set baggage for downstream context
export OTEL_PROPAGATORS=tracecontext,baggage
```

## Recommendations

### For Development

```bash
# Use console exporter to see trace output
export OTEL_TRACES_EXPORTER=console
export OTEL_TRACES_SAMPLER=always_on
```

### For Production

```bash
# Use OTLP with sampling
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1  # 10% sampling
```

### For Debugging Issues

```bash
# Enable all signals with debug output
export OTEL_TRACES_EXPORTER=console,otlp
export OTEL_METRICS_EXPORTER=console,otlp
export OTEL_LOGS_EXPORTER=console,otlp
```

## Known Limitations

1. **No Native Span Generation**: Claude Code may not create spans internally
2. **Limited Instrumentation**: Tool executions may not be traced
3. **No Distributed Tracing**: Cannot trace across multiple Claude Code instances

## Future Expectations

Based on OpenTelemetry roadmap and Claude Code development:

1. **Native Span Support**: Expected in future versions
2. **Tool-Level Tracing**: Detailed spans for each tool execution
3. **API Call Tracing**: Spans for Anthropic API interactions
4. **Semantic Conventions**: Adoption of LLM-specific semantic conventions

## Monitoring Trace Pipeline Health

```yaml
# Collector metrics for trace pipeline
service:
  telemetry:
    metrics:
      level: detailed
      address: 0.0.0.0:8888
```

Key metrics to monitor:
- `otelcol_exporter_sent_spans` - Spans exported successfully
- `otelcol_exporter_send_failed_spans` - Failed span exports
- `otelcol_processor_batch_batch_send_size` - Batch sizes

## Related Documentation

- [Jaeger/Distributed Tracing Guide](./09-jaeger-tracing.md)
- [OTEL Collector Setup](../TUTORIAL_OTEL_zh-TW.md#第三章otel-collector-設定)
- [Official OpenTelemetry Traces Docs](https://opentelemetry.io/docs/concepts/signals/traces/)
