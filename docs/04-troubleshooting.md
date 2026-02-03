# Troubleshooting Playbook

> **Last Updated**: 2026-02-03

A comprehensive guide to diagnosing and resolving common issues with Claude Code OTEL integration.

## Table of Contents

- [Quick Diagnostics](#quick-diagnostics)
- [Common Issues](#common-issues)
- [Debug Techniques](#debug-techniques)
- [Collector Issues](#collector-issues)
- [Backend Issues](#backend-issues)
- [Performance Issues](#performance-issues)
- [Network Issues](#network-issues)

## Quick Diagnostics

### Health Check Script

```bash
#!/bin/bash
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

### One-Line Diagnostic Commands

```bash
# Check if telemetry is enabled
env | grep -E "(CLAUDE|OTEL)" | sort

# Test OTLP endpoint connectivity
curl -v http://localhost:4317 2>&1 | head -20

# Check collector logs for errors
docker logs otel-collector 2>&1 | grep -i error | tail -20

# Verify Prometheus is scraping
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'

# Check for Claude Code metrics in Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=claude_code_session_count' | jq '.data.result'
```

## Common Issues

### Issue 1: No Telemetry Data Appearing

**Symptoms:**
- Grafana dashboards show "No data"
- Prometheus has no `claude_code_*` metrics
- No output in OTEL Collector logs

**Diagnostic Steps:**

```bash
# Step 1: Verify telemetry is enabled
echo $CLAUDE_CODE_ENABLE_TELEMETRY
# Expected: 1

# Step 2: Check exporter configuration
echo $OTEL_METRICS_EXPORTER
echo $OTEL_LOGS_EXPORTER
# Expected: otlp (or console for debugging)

# Step 3: Test with console exporter first
export OTEL_METRICS_EXPORTER=console
export OTEL_LOGS_EXPORTER=console
claude "hello"
# Should see JSON output in terminal
```

**Solutions:**

```bash
# Solution A: Enable telemetry
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# Solution B: Fix endpoint
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Solution C: Use correct protocol
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc  # or http/protobuf
```

### Issue 2: Connection Refused to OTEL Collector

**Symptoms:**
- Error: "connection refused"
- Telemetry enabled but no data in backend

**Diagnostic Steps:**

```bash
# Check if collector is running
docker ps | grep otel-collector

# Check collector port bindings
docker port otel-collector

# Test gRPC endpoint
grpcurl -plaintext localhost:4317 list

# Test HTTP endpoint
curl -v http://localhost:4318/v1/metrics
```

**Solutions:**

```bash
# Solution A: Start collector
docker compose up -d otel-collector

# Solution B: Fix port mapping in docker-compose.yml
# ports:
#   - "4317:4317"
#   - "4318:4318"

# Solution C: Use host.docker.internal for Docker-to-host communication
export OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317
```

### Issue 3: Metrics Missing Attributes

**Symptoms:**
- Team or environment labels not appearing
- Resource attributes not propagating

**Diagnostic Steps:**

```bash
# Check resource attributes
echo $OTEL_RESOURCE_ATTRIBUTES

# Query Prometheus for label presence
curl -s 'http://localhost:9090/api/v1/query?query=claude_code_session_count' | jq '.data.result[].metric'
```

**Solutions:**

```bash
# Solution: Set resource attributes correctly
export OTEL_RESOURCE_ATTRIBUTES="team=engineering,environment=production,service.name=claude-code"

# For managed settings (settings.json):
{
  "env": {
    "OTEL_RESOURCE_ATTRIBUTES": "team=engineering,environment=production"
  }
}
```

### Issue 4: High Memory Usage in Collector

**Symptoms:**
- Collector OOM kills
- High memory consumption
- Slow metric processing

**Diagnostic Steps:**

```bash
# Check collector memory usage
docker stats otel-collector

# Check queue backlog
curl -s http://localhost:8888/metrics | grep queue

# Check batch processor metrics
curl -s http://localhost:8888/metrics | grep batch
```

**Solutions:**

```yaml
# Solution: Add memory limiter processor
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

### Issue 5: SSL/TLS Certificate Errors

**Symptoms:**
- "certificate verify failed"
- "x509: certificate signed by unknown authority"

**Diagnostic Steps:**

```bash
# Check certificate validity
openssl s_client -connect collector.example.com:4317 </dev/null 2>/dev/null | openssl x509 -text | head -20

# Test with insecure flag (debug only)
export OTEL_EXPORTER_OTLP_INSECURE=true
claude "test"
```

**Solutions:**

```bash
# Solution A: Provide CA certificate
export OTEL_EXPORTER_OTLP_CERTIFICATE=/path/to/ca.crt

# Solution B: For mTLS, provide client certs
export OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE=/path/to/client.crt
export OTEL_EXPORTER_OTLP_CLIENT_KEY=/path/to/client.key

# Solution C: Use insecure for internal testing only
export OTEL_EXPORTER_OTLP_INSECURE=true
```

### Issue 6: Duplicate Metrics

**Symptoms:**
- Same metric reported multiple times
- Inflated counts in dashboards

**Diagnostic Steps:**

```bash
# Check for multiple exporters
echo $OTEL_METRICS_EXPORTER
# Should not have duplicates like "otlp,otlp"

# Check Prometheus for duplicate targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets | length'
```

**Solutions:**

```bash
# Solution A: Use single exporter
export OTEL_METRICS_EXPORTER=otlp

# Solution B: Configure deduplication in Prometheus
# prometheus.yml
scrape_configs:
  - job_name: 'otel-collector'
    honor_labels: true
    static_configs:
      - targets: ['otel-collector:8889']
```

## Debug Techniques

### Enable Debug Logging

```bash
# Console output for all signals
export OTEL_METRICS_EXPORTER=console,otlp
export OTEL_LOGS_EXPORTER=console,otlp
export OTEL_TRACES_EXPORTER=console,otlp

# Run Claude Code and observe output
claude "test prompt" 2>&1 | tee claude-debug.log
```

### Collector Debug Mode

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

### Trace OTLP Traffic

```bash
# Use tcpdump to capture OTLP traffic
sudo tcpdump -i any -w otlp-traffic.pcap port 4317

# Analyze with Wireshark or tshark
tshark -r otlp-traffic.pcap -Y "tcp.port == 4317"
```

### Test Exporter Manually

```bash
# Send test metrics via OTLP HTTP
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

## Collector Issues

### Collector Not Starting

```bash
# Check configuration syntax
docker run --rm -v $(pwd)/otel-collector-config.yaml:/etc/otel/config.yaml \
  otel/opentelemetry-collector-contrib:latest validate --config=/etc/otel/config.yaml

# Check for port conflicts
lsof -i :4317
lsof -i :4318

# View startup logs
docker logs otel-collector 2>&1 | head -50
```

### Pipeline Processing Failures

```bash
# Check processor errors
docker logs otel-collector 2>&1 | grep -i "processor"

# Monitor dropped metrics
curl -s http://localhost:8888/metrics | grep -E "(dropped|failed)"
```

## Backend Issues

### Prometheus Not Receiving Data

```bash
# Check remote write status
curl -s http://localhost:9090/api/v1/status/runtimeinfo | jq '.data.reloadConfigSuccess'

# Verify scrape targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastError: .lastError}'

# Check storage status
curl -s http://localhost:9090/api/v1/status/tsdb | jq '.data'
```

### Grafana Dashboard Empty

```bash
# Test Prometheus datasource
curl -s "http://admin:admin@localhost:3000/api/datasources/proxy/1/api/v1/query?query=up" | jq '.status'

# Check dashboard provisioning
docker logs grafana 2>&1 | grep -i "dashboard"

# Verify datasource configuration
curl -s "http://admin:admin@localhost:3000/api/datasources" | jq '.[].name'
```

## Performance Issues

### High Latency in Claude Code

```bash
# Check if telemetry is adding overhead
# Disable and compare
unset CLAUDE_CODE_ENABLE_TELEMETRY
time claude "test"

# Re-enable and compare
export CLAUDE_CODE_ENABLE_TELEMETRY=1
time claude "test"
```

### Collector Backpressure

```yaml
# Add sending queue for backpressure handling
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

## Network Issues

### DNS Resolution Failures

```bash
# Test DNS resolution
nslookup otel-collector.example.com

# Use IP address directly
export OTEL_EXPORTER_OTLP_ENDPOINT=http://10.0.0.50:4317
```

### Firewall Blocking

```bash
# Check if ports are open
nc -zv localhost 4317
nc -zv localhost 4318

# Test through firewall
telnet collector.example.com 4317
```

### Proxy Configuration

```bash
# If behind corporate proxy
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.internal
```

## Error Reference

| Error Message | Cause | Solution |
|--------------|-------|----------|
| `connection refused` | Collector not running | Start collector |
| `deadline exceeded` | Network timeout | Check network/firewall |
| `certificate verify failed` | TLS certificate issue | Configure certificates |
| `resource exhausted` | Collector overloaded | Add memory limits |
| `invalid endpoint` | Malformed URL | Check endpoint format |
| `permission denied` | File/port access | Check permissions |
| `unknown authority` | CA not trusted | Add CA certificate |

## Support Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [OTEL Collector Troubleshooting](https://opentelemetry.io/docs/collector/troubleshooting/)
- [Prometheus Troubleshooting](https://prometheus.io/docs/prometheus/latest/troubleshooting/)
- [Grafana Troubleshooting](https://grafana.com/docs/grafana/latest/troubleshooting/)

## Related Documentation

- [Production Deployment Guide](./02-production-deployment.md)
- [Security Hardening Guide](./11-security-hardening.md)
- [OTEL Collector Setup](../TUTORIAL_OTEL_zh-TW.md)
