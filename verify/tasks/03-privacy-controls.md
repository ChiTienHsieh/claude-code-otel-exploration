# Task 03: Privacy Controls Verification

驗證 Claude Code 的隱私控制功能正常運作。

## 前置條件

- 完成 [Task 02: Claude Code Integration](./02-claude-code-integration.md)
- OTEL stack 正在運行
- Claude Code 可以發送 telemetry

## 目標

驗證預設情況下敏感資料不會被記錄，並測試可選的隱私控制。

## 背景知識

Claude Code 的隱私設計原則：

| 預設行為 | 說明 |
|----------|------|
| API Keys | **永不記錄** |
| 檔案內容 | **永不記錄**（除非明確啟用） |
| User prompts | **預設不記錄** |
| Tool outputs | **預設不記錄** |

## 步驟

### Step 1: 確認預設隱私設定

確保沒有設定任何隱私相關的環境變數：

```bash
# 應該都沒有輸出
env | grep -E "OTEL_LOG_(USER_PROMPTS|TOOL_CONTENT)"
```

### Step 2: 執行包含敏感內容的指令

```bash
# 設定一個假的 "secret"
export MY_SECRET="super-secret-password-123"

# 執行 Claude Code，讓它讀取環境變數
claude "what is the value of MY_SECRET environment variable?"
```

### Step 3: 檢查 OTEL Collector Logs

```bash
docker-compose logs otel-collector | grep -i "secret"
```

**預期：不應該找到任何包含 "secret" 的內容**

如果有找到，代表隱私控制可能有問題！

### Step 4: 測試啟用 User Prompts 記錄

⚠️ **警告：這會記錄你的 prompts，僅在測試環境使用！**

```bash
# 啟用 user prompts 記錄
export OTEL_LOG_USER_PROMPTS=true

# 執行測試
claude "this is a test prompt for privacy verification"
```

檢查 logs：
```bash
docker-compose logs otel-collector | grep -i "privacy verification"
```

**預期：應該能找到 "privacy verification" 相關內容**

### Step 5: 測試 Tool Content 記錄

```bash
# 啟用 tool content 記錄
export OTEL_LOG_TOOL_CONTENT=true

# 執行會產生 tool output 的指令
claude "read the first 3 lines of docker-compose.yml"
```

檢查 logs：
```bash
docker-compose logs otel-collector | grep -i "docker-compose"
```

### Step 6: 恢復預設設定

```bash
unset OTEL_LOG_USER_PROMPTS
unset OTEL_LOG_TOOL_CONTENT
```

確認已清除：
```bash
env | grep -E "OTEL_LOG_(USER_PROMPTS|TOOL_CONTENT)"
# 應該沒有輸出
```

## 驗證清單

- [ ] 預設不記錄敏感環境變數內容
- [ ] 預設不記錄 user prompts 內容
- [ ] `OTEL_LOG_USER_PROMPTS=true` 可以啟用 prompt 記錄
- [ ] `OTEL_LOG_TOOL_CONTENT=true` 可以啟用 tool output 記錄
- [ ] 可以正確恢復預設設定

## 進階測試：Metrics 過濾

Claude Code 也支援 metrics 層級的隱私控制：

```bash
# 不包含 session ID 在 metrics 中
export OTEL_METRICS_INCLUDE_SESSION_ID=false

# 不包含 user UUID 在 metrics 中
export OTEL_METRICS_INCLUDE_USER_UUID=false
```

這些設定適合在意 metrics 中 PII (Personally Identifiable Information) 的場景。

## 安全建議

1. **生產環境**：保持預設設定（不記錄敏感內容）
2. **開發/測試**：可以暫時啟用來 debug
3. **CI/CD**：考慮關閉所有敏感內容記錄
4. **共用環境**：絕對不要啟用 prompt 記錄

## 下一步

恭喜！你已經完成所有驗證任務 🎉

可以繼續探索：
- 自訂 Resource Attributes
- 設定 Sampling（減少 trace 量）
- 整合到 CI/CD pipeline
