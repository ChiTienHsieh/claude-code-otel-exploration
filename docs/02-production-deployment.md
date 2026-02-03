# Production Deployment Guide

> **Last Updated**: 2026-02-03

This guide covers deploying Claude Code OTEL monitoring infrastructure in production environments, including Kubernetes, high availability, and scaling patterns.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Kubernetes Deployment](#kubernetes-deployment)
- [High Availability Setup](#high-availability-setup)
- [Scaling Patterns](#scaling-patterns)
- [Load Balancing](#load-balancing)
- [Failure Recovery](#failure-recovery)
- [Resource Planning](#resource-planning)

## Architecture Overview

### Production Architecture

```
                                    ┌─────────────────────────────────────────┐
                                    │           Kubernetes Cluster             │
                                    │                                          │
┌──────────────┐                    │  ┌────────────────────────────────────┐ │
│ Claude Code  │─────OTLP──────────▶│  │     OTEL Collector (DaemonSet)     │ │
│  Workstation │                    │  │  ┌──────┐ ┌──────┐ ┌──────┐       │ │
└──────────────┘                    │  │  │Node 1│ │Node 2│ │Node 3│       │ │
                                    │  │  └──┬───┘ └──┬───┘ └──┬───┘       │ │
┌──────────────┐                    │  └─────┼────────┼────────┼───────────┘ │
│ Claude Code  │─────OTLP──────────▶│        │        │        │             │
│   CI/CD      │                    │        ▼        ▼        ▼             │
└──────────────┘                    │  ┌────────────────────────────────────┐ │
                                    │  │   OTEL Collector Gateway (HA)      │ │
┌──────────────┐                    │  │      (Deployment - 3 replicas)     │ │
│ Claude Code  │─────OTLP──────────▶│  └─────────────────┬──────────────────┘ │
│   Server     │                    │                    │                    │
└──────────────┘                    │     ┌──────────────┼──────────────┐     │
                                    │     ▼              ▼              ▼     │
                                    │ ┌────────┐   ┌──────────┐   ┌────────┐ │
                                    │ │Prometheus│  │  Jaeger  │   │  Loki  │ │
                                    │ │  (HA)   │   │   (HA)   │   │  (HA)  │ │
                                    │ └────┬────┘   └──────────┘   └────────┘ │
                                    │      │                                  │
                                    │      ▼                                  │
                                    │ ┌──────────┐                            │
                                    │ │ Grafana  │                            │
                                    │ │   (HA)   │                            │
                                    │ └──────────┘                            │
                                    └─────────────────────────────────────────┘
```

## Kubernetes Deployment

### Namespace Setup

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: claude-observability
  labels:
    name: claude-observability
    istio-injection: disabled
```

### OTEL Collector - Gateway Mode

```yaml
# otel-collector-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector-gateway
  namespace: claude-observability
spec:
  replicas: 3
  selector:
    matchLabels:
      app: otel-collector-gateway
  template:
    metadata:
      labels:
        app: otel-collector-gateway
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: otel-collector-gateway
                topologyKey: kubernetes.io/hostname
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.96.0
          args:
            - "--config=/etc/otel-collector-config.yaml"
          ports:
            - containerPort: 4317  # OTLP gRPC
              name: otlp-grpc
            - containerPort: 4318  # OTLP HTTP
              name: otlp-http
            - containerPort: 8888  # Metrics
              name: metrics
            - containerPort: 13133 # Health check
              name: health
          resources:
            requests:
              cpu: 200m
              memory: 400Mi
            limits:
              cpu: 1000m
              memory: 2Gi
          livenessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: config
              mountPath: /etc/otel-collector-config.yaml
              subPath: otel-collector-config.yaml
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: claude-observability
spec:
  type: ClusterIP
  selector:
    app: otel-collector-gateway
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
    - name: metrics
      port: 8888
      targetPort: 8888
```

### OTEL Collector ConfigMap

```yaml
# otel-collector-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: claude-observability
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
            max_recv_msg_size_mib: 16
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 5s
        send_batch_size: 8192
        send_batch_max_size: 16384

      memory_limiter:
        check_interval: 1s
        limit_percentage: 80
        spike_limit_percentage: 25

      resource:
        attributes:
          - key: k8s.cluster.name
            value: production
            action: upsert
          - key: deployment.environment
            value: production
            action: upsert

      filter/health:
        error_mode: ignore
        metrics:
          metric:
            - 'name == "otelcol_process_uptime"'

    exporters:
      prometheusremotewrite:
        endpoint: http://prometheus:9090/api/v1/write
        tls:
          insecure: true
        retry_on_failure:
          enabled: true
          initial_interval: 5s
          max_interval: 30s
          max_elapsed_time: 300s

      otlp/jaeger:
        endpoint: jaeger-collector:4317
        tls:
          insecure: true

      loki:
        endpoint: http://loki:3100/loki/api/v1/push

      debug:
        verbosity: basic

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      pprof:
        endpoint: 0.0.0.0:1777
      zpages:
        endpoint: 0.0.0.0:55679

    service:
      extensions: [health_check, pprof, zpages]
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [prometheusremotewrite]
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [otlp/jaeger]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [loki]
      telemetry:
        logs:
          level: info
        metrics:
          level: detailed
          address: 0.0.0.0:8888
```

### Horizontal Pod Autoscaler

```yaml
# otel-collector-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: otel-collector-gateway
  namespace: claude-observability
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: otel-collector-gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: otelcol_receiver_accepted_spans
        target:
          type: AverageValue
          averageValue: "10000"
```

### PodDisruptionBudget

```yaml
# otel-collector-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: otel-collector-gateway
  namespace: claude-observability
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: otel-collector-gateway
```

## High Availability Setup

### Prometheus HA with Thanos

```yaml
# prometheus-ha.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: claude-observability
spec:
  replicas: 2
  serviceName: prometheus
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
        thanos-store-api: "true"
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: prometheus
              topologyKey: kubernetes.io/hostname
      containers:
        - name: prometheus
          image: prom/prometheus:v2.50.0
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus"
            - "--storage.tsdb.retention.time=6h"
            - "--storage.tsdb.min-block-duration=2h"
            - "--storage.tsdb.max-block-duration=2h"
            - "--web.enable-lifecycle"
            - "--web.enable-admin-api"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-storage
              mountPath: /prometheus
            - name: prometheus-config
              mountPath: /etc/prometheus
          resources:
            requests:
              cpu: 500m
              memory: 2Gi
            limits:
              cpu: 2000m
              memory: 8Gi
        - name: thanos-sidecar
          image: quay.io/thanos/thanos:v0.34.0
          args:
            - sidecar
            - --prometheus.url=http://localhost:9090
            - --tsdb.path=/prometheus
            - --grpc-address=0.0.0.0:10901
            - --http-address=0.0.0.0:10902
            - --objstore.config-file=/etc/thanos/objstore.yml
          ports:
            - containerPort: 10901
              name: grpc
            - containerPort: 10902
              name: http
          volumeMounts:
            - name: prometheus-storage
              mountPath: /prometheus
            - name: thanos-config
              mountPath: /etc/thanos
  volumeClaimTemplates:
    - metadata:
        name: prometheus-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

### Thanos Query

```yaml
# thanos-query.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: claude-observability
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
        - name: thanos-query
          image: quay.io/thanos/thanos:v0.34.0
          args:
            - query
            - --http-address=0.0.0.0:9090
            - --grpc-address=0.0.0.0:10901
            - --query.replica-label=prometheus_replica
            - --store=dnssrv+_grpc._tcp.prometheus.claude-observability.svc.cluster.local
          ports:
            - containerPort: 9090
              name: http
            - containerPort: 10901
              name: grpc
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 2Gi
```

### Grafana HA

```yaml
# grafana-ha.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: claude-observability
spec:
  replicas: 2
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: grafana
                topologyKey: kubernetes.io/hostname
      containers:
        - name: grafana
          image: grafana/grafana:10.3.1
          env:
            - name: GF_DATABASE_TYPE
              value: postgres
            - name: GF_DATABASE_HOST
              value: postgres:5432
            - name: GF_DATABASE_NAME
              value: grafana
            - name: GF_DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: grafana-db-credentials
                  key: username
            - name: GF_DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-db-credentials
                  key: password
            - name: GF_SESSION_PROVIDER
              value: redis
            - name: GF_SESSION_PROVIDER_CONFIG
              value: addr=redis:6379,pool_size=100
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - name: grafana-provisioning
              mountPath: /etc/grafana/provisioning
      volumes:
        - name: grafana-provisioning
          configMap:
            name: grafana-provisioning
```

## Scaling Patterns

### Pattern 1: Agent-Gateway Architecture

For large-scale deployments, use a two-tier collector architecture:

```yaml
# OTEL Collector Agent (DaemonSet - one per node)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector-agent
  namespace: claude-observability
spec:
  selector:
    matchLabels:
      app: otel-collector-agent
  template:
    metadata:
      labels:
        app: otel-collector-agent
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.96.0
          args:
            - "--config=/etc/otel-collector-config.yaml"
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
          volumeMounts:
            - name: config
              mountPath: /etc/otel-collector-config.yaml
              subPath: agent-config.yaml
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
```

Agent configuration (lightweight, forwards to gateway):

```yaml
# agent-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 400

exporters:
  loadbalancing:
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      dns:
        hostname: otel-collector-gateway.claude-observability.svc.cluster.local
        port: 4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loadbalancing]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loadbalancing]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loadbalancing]
```

### Pattern 2: Sharding by Team/Service

```yaml
# Shard metrics by team using routing
processors:
  routing:
    from_attribute: resource.team
    table:
      - value: platform
        exporters: [prometheusremotewrite/platform]
      - value: ml
        exporters: [prometheusremotewrite/ml]
      - value: infra
        exporters: [prometheusremotewrite/infra]
    default_exporters: [prometheusremotewrite/default]
```

## Load Balancing

### External Load Balancer (AWS NLB)

```yaml
# otel-collector-nlb.yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-collector-external
  namespace: claude-observability
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: otel-collector-gateway
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      targetPort: 4318
      protocol: TCP
```

### Client-Side Load Balancing with DNS

Configure Claude Code clients to use DNS-based load balancing:

```bash
# Use headless service for DNS round-robin
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.claude-observability.svc.cluster.local:4317
```

## Failure Recovery

### Circuit Breaker Pattern

```yaml
# otel-collector-config.yaml with retry and circuit breaker
exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 60s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
      storage: file_storage
```

### Persistent Queue Storage

```yaml
extensions:
  file_storage:
    directory: /var/lib/otelcol/file_storage
    timeout: 10s
    compaction:
      directory: /tmp/otelcol/compaction
      on_rebound: true

exporters:
  prometheusremotewrite:
    sending_queue:
      enabled: true
      storage: file_storage
```

### Backup Exporter (使用多 Pipeline 方式)

```yaml
# 設定備援 exporter - 使用雙 pipeline 寫入
exporters:
  prometheusremotewrite/primary:
    endpoint: http://prometheus-primary:9090/api/v1/write
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s

  prometheusremotewrite/backup:
    endpoint: http://prometheus-backup:9090/api/v1/write
    retry_on_failure:
      enabled: true

service:
  pipelines:
    metrics/primary:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite/primary]
    metrics/backup:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite/backup]
```

## Resource Planning

### Sizing Guidelines

| Scale | Claude Code Users | Collector Replicas | Memory per Replica | CPU per Replica |
|-------|-------------------|--------------------|--------------------|-----------------|
| Small | 1-10 | 2 | 512Mi | 200m |
| Medium | 10-50 | 3 | 1Gi | 500m |
| Large | 50-200 | 5 | 2Gi | 1000m |
| Enterprise | 200+ | 10+ | 4Gi | 2000m |

### Prometheus Storage Planning

```
Storage Required = Ingestion Rate × Retention Period × 2 (for overhead)

Example:
- 1000 samples/second
- 30 days retention
- Storage = 1000 × 86400 × 30 × 2 bytes ≈ 5.2 GB (with compression)
```

### Network Bandwidth

```
Bandwidth = (Metrics + Traces + Logs) × Replication Factor

Example:
- Metrics: 100 KB/s
- Traces: 500 KB/s
- Logs: 200 KB/s
- Replication: 2x
- Total: 1.6 MB/s
```

## Helm Chart

For simplified deployment, use the official OpenTelemetry Helm chart:

```bash
# Add Helm repository
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Install OTEL Collector
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace claude-observability \
  --create-namespace \
  --values otel-collector-values.yaml
```

Example `otel-collector-values.yaml`:

```yaml
mode: deployment
replicaCount: 3

config:
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
    memory_limiter:
      check_interval: 1s
      limit_percentage: 80

  exporters:
    prometheusremotewrite:
      endpoint: http://prometheus:9090/api/v1/write

  service:
    pipelines:
      metrics:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [prometheusremotewrite]

resources:
  requests:
    cpu: 200m
    memory: 400Mi
  limits:
    cpu: 1000m
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: opentelemetry-collector
          topologyKey: kubernetes.io/hostname
```

## Related Documentation

- [Security Hardening Guide](./11-security-hardening.md)
- [Troubleshooting Playbook](./04-troubleshooting.md)
- [Cost Optimization Guide](./03-cost-optimization.md)
