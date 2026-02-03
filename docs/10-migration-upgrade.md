# Migration & Upgrade Guide

> **Last Updated**: 2026-02-03

This guide covers migrating to Claude Code OTEL monitoring and upgrading existing deployments.

## Table of Contents

- [Migration Overview](#migration-overview)
- [From No Telemetry to OTEL](#from-no-telemetry-to-otel)
- [Upgrading Claude Code Versions](#upgrading-claude-code-versions)
- [Upgrading OTEL Infrastructure](#upgrading-otel-infrastructure)
- [Data Migration](#data-migration)
- [Rollback Procedures](#rollback-procedures)
- [Compatibility Matrix](#compatibility-matrix)

## Migration Overview

### Migration Paths

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Migration Scenarios                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Scenario 1: Enable OTEL for the first time                         │
│  ─────────────────────────────────────────────                      │
│  No Telemetry → OTEL Enabled                                        │
│                                                                      │
│  Scenario 2: Upgrade Claude Code version                            │
│  ───────────────────────────────────────────                        │
│  v2.0.x → v2.1.x (with OTEL changes)                               │
│                                                                      │
│  Scenario 3: Upgrade OTEL infrastructure                            │
│  ───────────────────────────────────────────                        │
│  Collector v0.90 → v0.96 / Prometheus upgrade                       │
│                                                                      │
│  Scenario 4: Migrate observability backend                          │
│  ─────────────────────────────────────────────                      │
│  Prometheus → Datadog / Self-hosted → Cloud                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## From No Telemetry to OTEL

### Phase 1: Planning (1-2 days)

#### Assessment Checklist

- [ ] Identify all Claude Code deployment locations
- [ ] Review network requirements (firewall rules, proxies)
- [ ] Estimate telemetry data volume
- [ ] Choose observability backend
- [ ] Define resource attribute schema (teams, environments)
- [ ] Review privacy requirements

#### Capacity Planning

```
Estimated Data Volume:
- Per session: ~10-50 KB metrics/logs
- Per user/day: ~1-5 MB (assuming 100 sessions)
- Per team (10 users): ~10-50 MB/day

Infrastructure Requirements:
- OTEL Collector: 200MB RAM, 0.2 CPU per 1000 sessions/hour
- Prometheus: 2GB RAM base + 1GB per 10K active series
- Grafana: 256MB RAM minimum
```

### Phase 2: Infrastructure Setup (1-2 days)

#### Step 1: Deploy OTEL Collector

```bash
# Create directory structure
mkdir -p /opt/otel/{config,data}

# Create collector config
cat > /opt/otel/config/collector.yaml << 'EOF'
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_percentage: 75

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  debug:
    verbosity: basic

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite, debug]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
EOF

# Start collector
docker run -d \
  --name otel-collector \
  -p 4317:4317 \
  -p 4318:4318 \
  -v /opt/otel/config:/etc/otel \
  otel/opentelemetry-collector-contrib:0.96.0 \
  --config=/etc/otel/collector.yaml
```

#### Step 2: Deploy Prometheus

```bash
# Prometheus config
cat > /opt/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

remote_write:
  - url: "http://localhost:9090/api/v1/write"

scrape_configs:
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8888']
EOF

# Start Prometheus
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /opt/prometheus:/etc/prometheus \
  -v prometheus-data:/prometheus \
  prom/prometheus:v2.50.0 \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-remote-write-receiver
```

#### Step 3: Deploy Grafana

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  grafana/grafana:10.3.1
```

### Phase 3: Client Configuration (Gradual Rollout)

#### Stage 1: Development Environment

```bash
# Start with development/staging
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_RESOURCE_ATTRIBUTES="environment=development,team=platform"

# Test
claude "hello world"

# Verify data in Prometheus
curl "http://localhost:9090/api/v1/query?query=claude_code_session_count"
```

#### Stage 2: Pilot Users (10%)

```bash
# Roll out to pilot group
# Add to user profile or managed settings

# Monitor for issues
# - Check collector logs for errors
# - Verify metrics appearing in Prometheus
# - Test Grafana dashboards
```

#### Stage 3: Full Rollout

```bash
# After successful pilot (1-2 weeks)
# Deploy to all users via managed settings

# settings.json (managed)
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://otel.company.com:4317",
    "OTEL_RESOURCE_ATTRIBUTES": "team=${TEAM_NAME},environment=production"
  }
}
```

### Phase 4: Validation

```bash
#!/bin/bash
# validate-migration.sh

echo "=== Migration Validation ==="

# Check collector health
echo "1. OTEL Collector:"
curl -s http://localhost:13133 && echo " ✓ Healthy" || echo " ✗ Unhealthy"

# Check Prometheus
echo "2. Prometheus:"
curl -s http://localhost:9090/-/healthy && echo " ✓ Healthy" || echo " ✗ Unhealthy"

# Check metrics exist
echo "3. Claude Code Metrics:"
METRICS=$(curl -s "http://localhost:9090/api/v1/query?query=claude_code_session_count" | jq '.data.result | length')
[ "$METRICS" -gt 0 ] && echo " ✓ Found $METRICS series" || echo " ✗ No metrics found"

# Check Grafana
echo "4. Grafana:"
curl -s http://localhost:3000/api/health | jq -r '.database' && echo " ✓ Healthy" || echo " ✗ Unhealthy"

echo "=== Validation Complete ==="
```

## Upgrading Claude Code Versions

### Pre-Upgrade Checklist

- [ ] Review release notes for OTEL changes
- [ ] Backup current configuration
- [ ] Test upgrade in non-production environment
- [ ] Plan rollback procedure
- [ ] Notify users of potential telemetry changes

### Version Compatibility

| Claude Code | OTEL SDK | Collector | Notes |
|-------------|----------|-----------|-------|
| 2.0.x | 1.18.x | 0.88+ | Initial OTEL support |
| 2.1.x | 1.21.x | 0.92+ | Added metrics |
| 2.2.x | 1.24.x | 0.96+ | Privacy controls |

### Upgrade Procedure

```bash
# 1. Check current version
claude --version

# 2. Backup configuration
cp ~/.claude/settings.json ~/.claude/settings.json.backup
cp ~/.bashrc ~/.bashrc.backup

# 3. Upgrade Claude Code
npm update -g @anthropic-ai/claude-code

# 4. Verify new version
claude --version

# 5. Test telemetry
export OTEL_METRICS_EXPORTER=console
claude "test"

# 6. Check for new environment variables
claude --help | grep -i otel

# 7. Update configuration if needed
# (Add new env vars, update deprecated ones)

# 8. Verify in observability stack
curl "http://localhost:9090/api/v1/query?query=claude_code_session_count"
```

### Handling Breaking Changes

```bash
# Example: Metric name change in v2.2
# Old: claude_code_token_count
# New: claude_code_tokens_input, claude_code_tokens_output

# Update Prometheus recording rules
# prometheus-rules-upgrade.yaml
groups:
  - name: compatibility
    rules:
      # Create compatibility metric
      - record: claude_code_token_count
        expr: claude_code_tokens_input + claude_code_tokens_output

# Update Grafana dashboards
# Find and replace metric names
sed -i 's/claude_code_token_count/claude_code_tokens_input + claude_code_tokens_output/g' \
  /var/lib/grafana/dashboards/*.json
```

## Upgrading OTEL Infrastructure

### OTEL Collector Upgrade

```bash
# 1. Check current version
docker inspect otel-collector | jq '.[0].Config.Image'

# 2. Review release notes
# https://github.com/open-telemetry/opentelemetry-collector/releases

# 3. Test new version
docker run --rm \
  -v /opt/otel/config:/etc/otel \
  otel/opentelemetry-collector-contrib:0.96.0 \
  validate --config=/etc/otel/collector.yaml

# 4. Backup current config
cp /opt/otel/config/collector.yaml /opt/otel/config/collector.yaml.v0.90

# 5. Update configuration for new version (if needed)
# Check for deprecated options

# 6. Rolling upgrade (for HA deployments)
kubectl set image deployment/otel-collector \
  otel-collector=otel/opentelemetry-collector-contrib:0.96.0

# 7. Single node upgrade
docker stop otel-collector
docker pull otel/opentelemetry-collector-contrib:0.96.0
docker start otel-collector

# 8. Verify
docker logs otel-collector | tail -20
curl http://localhost:13133
```

### Prometheus Upgrade

```bash
# 1. Check version compatibility
# https://prometheus.io/docs/prometheus/latest/migration/

# 2. Backup data
docker exec prometheus tar -czf /tmp/prometheus-backup.tar.gz /prometheus
docker cp prometheus:/tmp/prometheus-backup.tar.gz ./

# 3. Upgrade
docker pull prom/prometheus:v2.50.0
docker stop prometheus
docker start prometheus

# 4. Verify
curl http://localhost:9090/-/healthy
curl "http://localhost:9090/api/v1/query?query=up"
```

### Grafana Upgrade

```bash
# 1. Backup dashboards and config
docker exec grafana grafana-cli admin export > grafana-backup.json

# 2. Upgrade
docker pull grafana/grafana:10.3.1
docker stop grafana
docker start grafana

# 3. Verify
curl http://localhost:3000/api/health
```

## Data Migration

### Migrating Prometheus Data

```bash
# Export data using Prometheus remote read
# Use promtool for data migration

# Option 1: Prometheus Federation (live migration)
# New prometheus scrapes old prometheus
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{__name__=~"claude_code.*"}'
    static_configs:
      - targets: ['old-prometheus:9090']

# Option 2: Snapshot and restore
# Create snapshot on old Prometheus
curl -X POST http://old-prometheus:9090/api/v1/admin/tsdb/snapshot

# Copy snapshot to new location
rsync -av old-server:/prometheus/snapshots/xxx/ /new-prometheus/data/

# Option 3: Remote write backfill
# Use promtool to replay data
promtool tsdb dump /old-prometheus/data | \
  promtool tsdb create-blocks /new-prometheus/data
```

### Migrating to Different Backend

```yaml
# Example: Prometheus to Datadog migration
# Run both exporters during transition

exporters:
  # Keep existing
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

  # Add new
  datadog:
    api:
      key: ${DD_API_KEY}

service:
  pipelines:
    metrics:
      exporters: [prometheusremotewrite, datadog]  # Dual write
```

## Rollback Procedures

### Quick Rollback (Disable Telemetry)

```bash
# Immediate rollback - disable telemetry
unset CLAUDE_CODE_ENABLE_TELEMETRY
unset OTEL_EXPORTER_OTLP_ENDPOINT

# Or set to 0
export CLAUDE_CODE_ENABLE_TELEMETRY=0

# Verify
claude "test"  # Should not send telemetry
```

### Infrastructure Rollback

```bash
# Docker-based rollback
docker stop otel-collector prometheus grafana

# Restore from backup
docker run -d \
  --name otel-collector \
  -v /opt/otel/config/collector.yaml.backup:/etc/otel/collector.yaml \
  otel/opentelemetry-collector-contrib:0.90.0  # Previous version

# Restore Prometheus data
docker cp prometheus-backup.tar.gz prometheus:/tmp/
docker exec prometheus tar -xzf /tmp/prometheus-backup.tar.gz -C /

# Restart services
docker start prometheus grafana
```

### Configuration Rollback

```bash
# Restore user configuration
cp ~/.claude/settings.json.backup ~/.claude/settings.json
cp ~/.bashrc.backup ~/.bashrc
source ~/.bashrc
```

## Compatibility Matrix

### Claude Code ↔ OTEL Components

| Claude Code | OTEL Collector | Prometheus | Grafana | Status |
|-------------|----------------|------------|---------|--------|
| 2.0.x | 0.88-0.92 | 2.45+ | 9.x+ | Supported |
| 2.1.x | 0.90-0.96 | 2.47+ | 10.x+ | Supported |
| 2.2.x | 0.94+ | 2.50+ | 10.2+ | Current |

### Environment Variable Changes

| Version | Added | Deprecated | Removed |
|---------|-------|------------|---------|
| 2.0.0 | `CLAUDE_CODE_ENABLE_TELEMETRY` | - | - |
| 2.1.0 | `OTEL_LOG_USER_PROMPTS` | - | - |
| 2.1.1 | `OTEL_LOG_TOOL_CONTENT` | - | - |
| 2.2.0 | `OTEL_METRICS_SESSION_ID` | `CLAUDE_OTEL_ENABLED` (use `CLAUDE_CODE_ENABLE_TELEMETRY`) | - |

### Breaking Changes by Version

```yaml
# v2.1.0 Breaking Changes
metrics:
  - change: "Renamed claude_code_token_count to claude_code_tokens_input/output"
    migration: "Update dashboards and alerts to use new metric names"

# v2.2.0 Breaking Changes
config:
  - change: "CLAUDE_OTEL_ENABLED deprecated"
    migration: "Use CLAUDE_CODE_ENABLE_TELEMETRY instead"
```

## Migration Support

### Getting Help

- Review [Troubleshooting Playbook](./04-troubleshooting.md)
- Check [Official Documentation](https://docs.anthropic.com/en/docs/claude-code/telemetry)
- Open GitHub issue for migration-specific problems

### Migration Checklist Template

```markdown
## Migration Checklist

### Pre-Migration
- [ ] Review release notes
- [ ] Backup configurations
- [ ] Test in non-production
- [ ] Prepare rollback plan
- [ ] Notify stakeholders

### Infrastructure
- [ ] Deploy OTEL Collector
- [ ] Configure Prometheus
- [ ] Setup Grafana dashboards
- [ ] Configure alerts

### Client Rollout
- [ ] Development environment
- [ ] Pilot users (10%)
- [ ] Extended pilot (50%)
- [ ] Full rollout (100%)

### Validation
- [ ] Metrics flowing
- [ ] Dashboards working
- [ ] Alerts functional
- [ ] Documentation updated

### Post-Migration
- [ ] Monitor for issues (1 week)
- [ ] Collect feedback
- [ ] Clean up old infrastructure
- [ ] Document lessons learned
```

## Related Documentation

- [Troubleshooting Playbook](./04-troubleshooting.md)
- [Production Deployment Guide](./02-production-deployment.md)
- [Security Hardening Guide](./11-security-hardening.md)
