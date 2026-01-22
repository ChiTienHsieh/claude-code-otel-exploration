# Task 01: Basic Setup Verification

驗證 observability stack 正常運作。

## 目標

確認所有 containers 正常啟動，服務之間可以互相通訊。

## 步驟

### Step 1: 啟動 Stack

```bash
cd verify
docker-compose up -d
```

預期輸出：
```
[+] Running 5/5
 ✔ Network verify_otel-net       Created
 ✔ Container otel-collector      Started
 ✔ Container jaeger              Started
 ✔ Container prometheus          Started
 ✔ Container grafana             Started
```

### Step 2: 檢查 Container 狀態

```bash
docker-compose ps
```

預期：所有 4 個 containers 都是 `running` 狀態。

### Step 3: 檢查 OTEL Collector

```bash
# 查看 logs
docker-compose logs otel-collector | head -20

# 檢查 health endpoint
curl http://localhost:8888/metrics | head -5
```

預期：看到 `Everything is ready.` 或類似訊息。

### Step 4: 檢查 Prometheus Targets

開啟瀏覽器：http://localhost:9090/targets

預期：
- `otel-collector` job: UP
- `claude-code` job: UP

### Step 5: 檢查 Grafana

開啟瀏覽器：http://localhost:3000

1. 登入 (admin / admin)
2. 可以跳過改密碼
3. 左側選單 → Dashboards → Claude Code → Claude Code OTEL Overview

預期：Dashboard 載入成功（目前沒資料是正常的）

### Step 6: 檢查 Jaeger

開啟瀏覽器：http://localhost:16686

預期：Jaeger UI 正常顯示（目前沒 traces 是正常的）

## 驗證清單

- [ ] 4 個 containers 都在運行
- [ ] OTEL Collector 沒有錯誤
- [ ] Prometheus targets 都是 UP
- [ ] Grafana 可以登入
- [ ] Jaeger UI 可以訪問

## 下一步

完成後，進行 [Task 02: Claude Code Integration](./02-claude-code-integration.md)
