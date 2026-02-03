# Security Hardening 指南

> **最後更新**：2026 年 2 月 3 日

本指南完整說明如何在生產環境中保護 Claude Code OTEL telemetry 基礎設施的安全。

## 目錄

- [Security 概覽](#security-概覽)
- [傳輸安全（TLS/mTLS）](#傳輸安全tlsmtls)
- [Authentication 與 Authorization](#authentication-與-authorization)
- [資料隱私](#資料隱私)
- [網路安全](#網路安全)
- [Secret 管理](#secret-管理)
- [稽核日誌](#稽核日誌)
- [Compliance 考量](#compliance-考量)
- [Security 檢查清單](#security-檢查清單)

## Security 概覽

### 威脅模型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Threat Landscape                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐            │
│  │ Data in      │     │ Data in      │     │ Data at      │            │
│  │ Transit      │     │ Processing   │     │ Rest         │            │
│  ├──────────────┤     ├──────────────┤     ├──────────────┤            │
│  │ - Eavesdrop  │     │ - Injection  │     │ - Theft      │            │
│  │ - MITM       │     │ - Tampering  │     │ - Exposure   │            │
│  │ - Replay     │     │ - DoS        │     │ - Retention  │            │
│  └──────────────┘     └──────────────┘     └──────────────┘            │
│                                                                          │
│  Mitigations:         Mitigations:         Mitigations:                 │
│  - TLS/mTLS           - Input validation   - Encryption                 │
│  - Certificate pin    - Rate limiting      - Access control             │
│  - Network isolation  - Memory limits      - Data retention             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Security 層級

1. **傳輸層**：TLS 加密、mTLS authentication
2. **應用層**：API authentication、authorization
3. **資料層**：靜態加密、資料遮罩
4. **網路層**：防火牆、網路隔離
5. **操作層**：稽核日誌、監控

## 傳輸安全（TLS/mTLS）

### Claude Code 的 TLS 配置

```bash
# Basic TLS
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector.company.com:4317
export OTEL_EXPORTER_OTLP_CERTIFICATE=/etc/ssl/certs/ca-certificates.crt

# Custom CA certificate
export OTEL_EXPORTER_OTLP_CERTIFICATE=/path/to/company-ca.crt
```

### mTLS（Mutual TLS）配置

```bash
# mTLS with client certificates
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector.company.com:4317
export OTEL_EXPORTER_OTLP_CERTIFICATE=/path/to/ca.crt
export OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE=/path/to/client.crt
export OTEL_EXPORTER_OTLP_CLIENT_KEY=/path/to/client.key
```

### OTEL Collector TLS 配置

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        tls:
          cert_file: /etc/otel/certs/server.crt
          key_file: /etc/otel/certs/server.key
          client_ca_file: /etc/otel/certs/ca.crt
          # Require client certificates (mTLS)
          client_auth: RequireAndVerifyClientCert

      http:
        endpoint: 0.0.0.0:4318
        tls:
          cert_file: /etc/otel/certs/server.crt
          key_file: /etc/otel/certs/server.key

exporters:
  prometheusremotewrite:
    endpoint: https://prometheus:9090/api/v1/write
    tls:
      cert_file: /etc/otel/certs/client.crt
      key_file: /etc/otel/certs/client.key
      ca_file: /etc/otel/certs/ca.crt
```

### 憑證產生

```bash
#!/bin/bash
set -euo pipefail
# generate-certs.sh - Generate certificates for mTLS

# Set variables
DAYS=365
ORG="MyCompany"
CN_CA="OTEL CA"
CN_SERVER="otel-collector.company.com"
CN_CLIENT="claude-code-client"

# Create directory
mkdir -p certs && cd certs

# Generate CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days $DAYS -key ca.key -out ca.crt \
  -subj "/O=$ORG/CN=$CN_CA"

# Generate server certificate
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/O=$ORG/CN=$CN_SERVER"

cat > server.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = $CN_SERVER
DNS.2 = localhost
IP.1 = 127.0.0.1
EOF

openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days $DAYS -extfile server.ext

# Generate client certificate
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr \
  -subj "/O=$ORG/CN=$CN_CLIENT"

cat > client.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
EOF

openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days $DAYS -extfile client.ext

# Verify certificates
echo "Verifying certificates..."
openssl verify -CAfile ca.crt server.crt
openssl verify -CAfile ca.crt client.crt

# Set permissions
chmod 600 *.key
chmod 644 *.crt

echo "Certificates generated in $(pwd)"
```

## Authentication 與 Authorization

### API Key Authentication

```bash
# Bearer token authentication
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer ${OTEL_API_KEY}"
```

### OTEL Collector 搭配 Auth Extension

```yaml
# otel-collector-config.yaml
extensions:
  bearertokenauth:
    token: ${OTEL_AUTH_TOKEN}

  basicauth:
    client_auth:
      username: ${OTEL_USERNAME}
      password: ${OTEL_PASSWORD}

  oauth2client:
    client_id: ${OAUTH_CLIENT_ID}
    client_secret: ${OAUTH_CLIENT_SECRET}
    token_url: https://auth.company.com/oauth/token
    scopes: ["telemetry:write"]

receivers:
  otlp:
    protocols:
      grpc:
        auth:
          authenticator: bearertokenauth

exporters:
  prometheusremotewrite:
    endpoint: https://prometheus:9090/api/v1/write
    auth:
      authenticator: basicauth
```

### 動態 Token 刷新

```bash
#!/bin/bash
set -euo pipefail
# token-refresh.sh - Dynamic bearer token refresh

# Path for token file
TOKEN_FILE="/tmp/otel-token"
TOKEN_URL="https://auth.company.com/oauth/token"
CLIENT_ID="${OAUTH_CLIENT_ID}"
CLIENT_SECRET="${OAUTH_CLIENT_SECRET}"

refresh_token() {
    TOKEN=$(curl -s -X POST "$TOKEN_URL" \
        -d "grant_type=client_credentials" \
        -d "client_id=$CLIENT_ID" \
        -d "client_secret=$CLIENT_SECRET" \
        | jq -r '.access_token')

    echo "$TOKEN" > "$TOKEN_FILE"
    echo "Token refreshed: $(date)"
}

# Refresh every 50 minutes (token expires in 60)
while true; do
    refresh_token
    sleep 3000
done
```

Claude Code 動態 token 配置：

```bash
export OTEL_EXPORTER_OTLP_HEADERS_FILE=/tmp/otel-headers
```

### Observability Stack 的 RBAC

```yaml
# Kubernetes RBAC for OTEL Collector
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector
    namespace: observability
```

## 資料隱私

### 預設隱私控制

Claude Code 內建隱私保護：

```bash
# These are NEVER logged by default:
# - API keys
# - File contents
# - User prompts
# - Tool outputs

# Explicit opt-in required:
export OTEL_LOG_USER_PROMPTS=false    # Default: false
export OTEL_LOG_TOOL_CONTENT=false    # Default: false
```

### Collector 中的資料遮罩

```yaml
# otel-collector-config.yaml
processors:
  # Redact sensitive data
  redaction:
    # Block entire attributes
    blocked_values:
      - "(?i)password"
      - "(?i)secret"
      - "(?i)api[_-]?key"
      - "(?i)token"
      - "(?i)bearer"

    # Mask patterns in values
    summary: debug
    allow_all_keys: false
    blocked_key_patterns:
      - "(?i).*password.*"
      - "(?i).*secret.*"
      - "(?i).*credential.*"

  # Hash sensitive identifiers
  transform:
    log_statements:
      - context: log
        statements:
          # Hash user emails
          - set(attributes["user.email"], SHA256(attributes["user.email"]))

  # Filter metrics with sensitive labels
  filter:
    metrics:
      exclude:
        match_type: regexp
        metric_names:
          - ".*secret.*"
          - ".*password.*"
```

### Attribute 過濾

```yaml
# Remove sensitive attributes before export
processors:
  attributes:
    actions:
      # Remove specific attributes
      - key: user.email
        action: delete
      - key: api.key
        action: delete

      # Hash instead of remove
      - key: session.id
        action: hash

      # Truncate long values
      - key: user.prompt
        action: truncate
        max_length: 100
```

## 網路安全

### 防火牆規則

```bash
# iptables rules for OTEL infrastructure
# Allow OTLP from internal network only
iptables -A INPUT -p tcp --dport 4317 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 4317 -j DROP

# Allow Prometheus from specific hosts
iptables -A INPUT -p tcp --dport 9090 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 9090 -j DROP

# Allow Grafana from internal only
iptables -A INPUT -p tcp --dport 3000 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 3000 -j DROP
```

### Kubernetes Network Policies

```yaml
# networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: otel-collector-policy
  namespace: observability
spec:
  podSelector:
    matchLabels:
      app: otel-collector
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow OTLP from application namespaces
    - from:
        - namespaceSelector:
            matchLabels:
              otel-enabled: "true"
      ports:
        - protocol: TCP
          port: 4317
        - protocol: TCP
          port: 4318
  egress:
    # Allow to Prometheus
    - to:
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
    # Allow to Jaeger
    - to:
        - podSelector:
            matchLabels:
              app: jaeger
      ports:
        - protocol: TCP
          port: 14250
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Service Mesh 整合（Istio）

```yaml
# istio-peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: otel-mtls
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: otel-collector-policy
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/*/sa/claude-code-sa"]
      to:
        - operation:
            ports: ["4317", "4318"]
```

## Secret 管理

### Kubernetes Secrets

```yaml
# otel-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: otel-secrets
  namespace: observability
type: Opaque
stringData:
  auth-token: "your-secure-token"
  prometheus-password: "prometheus-password"
---
# Reference in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  template:
    spec:
      containers:
        - name: otel-collector
          env:
            - name: OTEL_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: otel-secrets
                  key: auth-token
```

### HashiCorp Vault 整合

```yaml
# vault-agent-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-agent-config
data:
  config.hcl: |
    auto_auth {
      method "kubernetes" {
        mount_path = "auth/kubernetes"
        config = {
          role = "otel-collector"
        }
      }
      sink "file" {
        config = {
          path = "/vault/secrets/token"
        }
      }
    }

    template {
      source = "/vault/templates/otel-secrets.tpl"
      destination = "/vault/secrets/otel-secrets.env"
    }
---
# Template for secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-templates
data:
  otel-secrets.tpl: |
    {{ with secret "secret/data/otel/collector" }}
    OTEL_AUTH_TOKEN={{ .Data.data.auth_token }}
    PROMETHEUS_PASSWORD={{ .Data.data.prometheus_password }}
    {{ end }}
```

### AWS Secrets Manager

```yaml
# otel-collector with AWS Secrets Manager
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  prometheusremotewrite:
    endpoint: https://prometheus:9090/api/v1/write
    headers:
      Authorization: "Bearer ${env:PROMETHEUS_TOKEN}"

# Use AWS Secrets Manager with external-secrets
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: otel-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: ClusterSecretStore
  target:
    name: otel-secrets
  data:
    - secretKey: auth-token
      remoteRef:
        key: /prod/otel/auth-token
```

## 稽核日誌

### OTEL Collector 稽核日誌

```yaml
# otel-collector-config.yaml
service:
  telemetry:
    logs:
      level: info
      development: false
      encoding: json
      output_paths: ["/var/log/otel/collector.log"]
      error_output_paths: ["/var/log/otel/error.log"]

# Add audit processor
processors:
  audit:
    # Log all incoming telemetry metadata
    log_level: info
    include:
      - resource.attributes
      - scope.name
    exclude:
      - log.body  # Don't log actual content
```

### Prometheus 稽核日誌

```yaml
# prometheus.yml
global:
  external_labels:
    cluster: production

# Enable audit logging via flags
# --web.enable-lifecycle
# --log.level=info
```

### Grafana 稽核日誌

```ini
# grafana.ini
[log]
mode = console file
level = info

[log.file]
log_rotate = true
max_lines = 1000000
max_size_shift = 28
daily_rotate = true

[auditing]
enabled = true
log_dashboard_views = true
log_datasource_requests = true
```

### 集中式稽核日誌收集

```yaml
# fluent-bit config for audit logs
[INPUT]
    Name tail
    Path /var/log/otel/*.log
    Tag otel.collector

[INPUT]
    Name tail
    Path /var/log/prometheus/*.log
    Tag prometheus

[INPUT]
    Name tail
    Path /var/log/grafana/*.log
    Tag grafana

[OUTPUT]
    Name es
    Match *
    Host elasticsearch
    Port 9200
    Index audit-logs
```

## Compliance 考量

### GDPR Compliance

```yaml
# Data minimization - only collect necessary data
processors:
  filter:
    logs:
      exclude:
        match_type: strict
        bodies:
          - "user_prompt"  # Don't store prompts
          - "tool_output"  # Don't store outputs

  # Data retention
  retention:
    max_age: 720h  # 30 days maximum

# Right to erasure - configure data deletion
# Prometheus retention
storage:
  tsdb:
    retention.time: 30d
```

### SOC 2 Compliance

```yaml
# Access control logging
service:
  telemetry:
    logs:
      level: info

# Encryption in transit (TLS)
receivers:
  otlp:
    protocols:
      grpc:
        tls:
          cert_file: /certs/server.crt
          key_file: /certs/server.key

# Regular security reviews
# - Certificate rotation every 90 days
# - Access review quarterly
# - Vulnerability scanning weekly
```

### HIPAA 考量

```yaml
# If processing healthcare data
processors:
  # Remove PHI
  redaction:
    blocked_values:
      - "\\d{3}-\\d{2}-\\d{4}"  # SSN
      - "\\d{10}"               # MRN patterns

  # Encryption
  transform:
    log_statements:
      - context: log
        statements:
          - set(attributes["patient.id"], SHA256(attributes["patient.id"]))
```

## Security 檢查清單

### 部署前

- [ ] TLS 憑證已產生並分發
- [ ] 所有元件已配置 mTLS
- [ ] Authentication 機制已配置
- [ ] Network policies 已定義
- [ ] Secrets 安全儲存（Vault/Secrets Manager）
- [ ] 資料隱私控制已配置

### 部署中

- [ ] 所有 endpoints 使用 HTTPS
- [ ] Client certificates 已驗證
- [ ] 防火牆已配置
- [ ] 網路隔離已就位
- [ ] 稽核日誌已啟用
- [ ] Security 事件監控中

### 營運中

- [ ] 憑證輪換已排程（90 天）
- [ ] 存取權審查已排程（每季）
- [ ] 弱點掃描已啟用
- [ ] Incident response 計畫已記錄
- [ ] 備份與復原已測試
- [ ] Security 訓練已完成

### 定期審查

- [ ] 每週：弱點掃描結果
- [ ] 每月：存取日誌審查
- [ ] 每季：完整 security 稽核
- [ ] 每年：滲透測試

## Security 監控

### Security 告警

```yaml
# prometheus-security-alerts.yaml
groups:
  - name: security-alerts
    rules:
      - alert: TLSCertificateExpiringSoon
        expr: |
          probe_ssl_earliest_cert_expiry - time() < 86400 * 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "TLS certificate expiring in less than 30 days"

      - alert: UnauthorizedAccessAttempt
        expr: |
          rate(otel_collector_auth_failures_total[5m]) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High rate of authentication failures detected"

      - alert: AnomalousDataVolume
        expr: |
          rate(otel_collector_received_spans_total[5m])
          > 3 * avg_over_time(rate(otel_collector_received_spans_total[5m])[7d:1h])
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Anomalous telemetry data volume detected"
```

## 相關文件

- [生產環境部署指南](./02-production-deployment.md)
- [疑難排解手冊](./04-troubleshooting.md)
- [Migration 與升級指南](./10-migration-upgrade.md)
