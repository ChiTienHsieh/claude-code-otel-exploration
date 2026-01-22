# YOLO CC: Containerized Claude Code for OTEL Testing

在隔離的 Docker 容器中執行 Claude Code，完全不影響你的本機環境。

## 為什麼要用 Container？

| 好處 | 說明 |
|------|------|
| **隔離性** | 不會影響本機的 Claude Code 設定 |
| **可重現** | 每次都是乾淨的環境 |
| **安全** | API Key 只在 container 內使用 |
| **一致性** | 確保測試環境相同 |

## Quick Start

### 方法一：使用 docker-compose（推薦）

```bash
# 先設定 API Key
export ANTHROPIC_API_KEY=sk-ant-xxxxx

# 啟動完整 stack（包含 YOLO CC）
docker-compose --profile yolo up -d

# 進入 Claude Code container
docker-compose exec yolo-cc claude
```

### 方法二：手動 Build & Run

```bash
# 1. 確保 OTEL stack 已啟動
docker-compose up -d

# 2. Build image
docker build -t claude-code-otel-verify ./yolo-cc

# 3. Run（連到 OTEL network）
docker run -it --rm \
  --network verify_otel-net \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  claude-code-otel-verify
```

## 環境變數

Container 已預設以下環境變數：

```bash
CLAUDE_CODE_ENABLE_TELEMETRY=1
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_TRACES_EXPORTER=otlp,console
OTEL_METRICS_EXPORTER=otlp,console
OTEL_LOGS_EXPORTER=otlp,console
```

### 覆蓋設定

```bash
docker run -it --rm \
  --network verify_otel-net \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -e OTEL_LOG_USER_PROMPTS=true \
  claude-code-otel-verify
```

## 測試流程

### 1. 啟動環境

```bash
# Terminal 1: 啟動 OTEL stack
cd verify
docker-compose up -d

# 確認都起來了
docker-compose ps
```

### 2. 執行 Claude Code

```bash
# Terminal 2: 進入 YOLO CC
docker run -it --rm \
  --network verify_otel-net \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  claude-code-otel-verify

# 在 Claude Code 裡執行一些操作
> list files in current directory
> create a hello.py that prints hello world
> run the python file
```

### 3. 觀察 Telemetry

```bash
# Terminal 3: 看 Collector logs
docker-compose logs -f otel-collector

# 或開瀏覽器
# Grafana: http://localhost:3000
# Jaeger:  http://localhost:16686
```

## 掛載本地目錄

如果想讓 Claude Code 操作你的本地專案：

```bash
docker run -it --rm \
  --network verify_otel-net \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v $(pwd)/my-project:/workspace/my-project \
  -w /workspace/my-project \
  claude-code-otel-verify
```

## 安全注意事項

1. **API Key**：只透過環境變數傳入，不要寫入 Dockerfile 或 commit
2. **Network**：Container 只能連到 OTEL stack，無法存取其他服務
3. **Volume**：謹慎掛載本地目錄，Container 內的 Claude Code 有完整存取權

## Troubleshooting

### Container 連不到 OTEL Collector？

確認 network 正確：
```bash
# 列出 networks
docker network ls

# 應該看到 verify_otel-net
# 如果沒有，先 docker-compose up -d
```

### Claude Code 沒啟動 telemetry？

檢查環境變數：
```bash
docker run --rm \
  --network verify_otel-net \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  claude-code-otel-verify \
  env | grep -E "(CLAUDE_CODE|OTEL)"
```

### Image build 失敗？

確認 Node.js 和 npm 可以正常運作：
```bash
docker build --no-cache -t claude-code-otel-verify ./yolo-cc
```
