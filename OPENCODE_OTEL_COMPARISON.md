# OpenCode vs Claude Code：OpenTelemetry 支援度比較 ( ˘•ω•˘ )

> 研究日期：2026-01-21
> 作者：一個好奇的 Opus 4.5

---

## 1. OpenCode 簡介

```
OpenCode = Open + Code
           開放   程式碼

「100% 開源的 AI 編程代理」
```

### 基本資訊

| 項目 | 資訊 |
|------|------|
| 專案位置 | [github.com/sst/opencode](https://github.com/sst/opencode) |
| 目前版本 | v1.1.28 (2026-01-20) |
| Star 數 | 80.5k ヽ(>∀<☆)☆（截至 2026-01-21） |
| 貢獻者 | 608+（截至 2026-01-21） |
| 主要語言 | TypeScript (84.5%) |
| 授權 | MIT |

### 主要特色

1. **完全開源** - MIT 授權，可自由使用、修改、商用
2. **多模型支援** - Claude、OpenAI、Google、本地模型通吃
3. **雙代理模式**：
   - `build` 代理：完整存取權限的開發工具
   - `plan` 代理：唯讀模式，更安全的規劃工具
4. **終端機介面** - TUI 設計，類似 Claude Code 的體驗
5. **供應商獨立** - 不綁定特定 AI 提供商

---

## 2. OpenTelemetry 支援現況

### OpenCode：實驗性支援 - 已發布！(✅)

OpenCode 對 OTEL 的支援已在 **v1.0.134** 正式發布為實驗性功能 ヽ(>∀<☆)☆

#### 2.1 里程碑：v1.0.134 Release

根據 [GitHub Release v1.0.134](https://github.com/sst/opencode/releases/tag/v1.0.134)：
> "Added experimental OpenTelemetry config option to enable OTEL spans"

這個功能來自 [PR #4978](https://github.com/sst/opencode/pull/4978)，為 OpenCode 帶來了基礎的 OTEL 支援。

> ⚠️ **注意：** [PR #5245](https://github.com/sst/opencode/pull/5245) 是另一個相關的 OTEL 增強 PR，**目前仍為 Open 狀態，尚未合併**。

#### 2.2 現有功能

**啟用方式：**
```jsonc
// .opencode/opencode.jsonc
{
  "experimental": {
    "openTelemetry": true
  }
}
```

設定 `experimental.openTelemetry` 為 `true` 後，OpenCode 會開始發送 telemetry spans。

#### 2.3 已發布的功能（PR #4978）

[PR #4978](https://github.com/sst/opencode/pull/4978) 帶來的 OTEL 實驗性支援：

- 新增 `experimental.openTelemetry` 配置選項
- 支援發送基本的 OTEL spans
- 支援 OTLP exporter

#### 2.4 尚未合併的增強功能（PR #5245）

[PR #5245](https://github.com/sst/opencode/pull/5245) 是後續的增強 PR，**目前仍為 Open 狀態**，預計帶來：

- 更完整的 session 管理 tracing
- CLI parsing 追蹤
- 增強的錯誤處理和事件記錄

#### 2.5 社群需求

[Issue #2666](https://github.com/sst/opencode/issues/2666) 請求更詳細的 per-interaction telemetry：
- 每次請求/回應的 token 計數
- 輸入組成分析（prompt、system、tool calls 等）
- 可選的 verbose logging 和資料脫敏

這個 issue 有 12 個 thumbs up（截至 2026-01-21），顯示社群確實需要這功能！

#### 2.6 隱私說明

[Issue #459](https://github.com/sst/opencode/issues/459) 曾詢問 OpenCode 的隱私政策。

**維護者明確回覆：** "there is no telemetry collected"

這意味著 OpenCode **預設不收集任何 telemetry 資料**。OTEL 功能是使用者主動啟用後，將資料送到**使用者自己指定的 endpoint**，而非送給 OpenCode 團隊。

這是很好的隱私設計！( •̀ω•́ )✧

---

### Claude Code：正式支援 (✅)

Claude Code 的 OTEL 支援是**內建且正式的**：

#### 主要特點

1. **原生支援** - 不需要 wrapper 或 sidecar
2. **標準 OTEL** - 完全遵循 OpenTelemetry 規範
3. **隱私優先** - 預設不記錄敏感資料
4. **企業級功能** - 支援 managed settings 集中管理

#### 支援的 Telemetry 類型

| 類型 | 支援度 | 說明 |
|------|--------|------|
| Metrics | ✅ 完整 | Token 用量、成本、session 統計 |
| Logs | ✅ 完整 | 事件記錄、錯誤追蹤 |
| Traces | ✅ 透過 Spans | 官方支援，SigNoz/Honeycomb 整合可用 |

**Traces 整合案例：**
- [SigNoz 整合指南](https://signoz.io/docs/claude-code-monitoring/) - 成功追蹤 Claude Code spans
- [Honeycomb 分析](https://www.honeycomb.io/blog/can-claude-code-observe-its-own-code) - 深度分析 Claude Code trace 資料

#### 環境變數

Claude Code 支援標準的 [OTEL 環境變數](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)，以下是常用的配置：

```bash
# 基本
CLAUDE_CODE_ENABLE_TELEMETRY=1
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 隱私控制
OTEL_LOG_USER_PROMPTS=0
OTEL_LOG_TOOL_CONTENT=0

# 企業功能
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer token"
OTEL_EXPORTER_OTLP_CERTIFICATE=/path/to/cert
```

---

## 3. 功能對比表

> 更新日期：2026-01-21（根據 OpenCode v1.0.134 release）

| 功能 | Claude Code | OpenCode |
|------|:-----------:|:--------:|
| **OTEL 支援狀態** | ✅ 正式支援 | ✅ 實驗性（v1.0.134+） |
| **Metrics 匯出** | ✅ 完整 | ⚠️ 尚未實作 |
| **Logs 匯出** | ✅ 完整 | ⚠️ 尚未實作 |
| **Traces 匯出** | ✅ 完整 | ✅ 實驗性支援 |
| **Console Exporter** | ✅ 支援 | ⚠️ 未知 |
| **OTLP Exporter** | ✅ 支援 | ✅ 實驗性支援 |
| **Prometheus Exporter** | ✅ 支援 | ❌ 不支援 |
| **gRPC Protocol** | ✅ 支援 | ⚠️ 未知 |
| **HTTP/protobuf** | ✅ 支援 | ⚠️ 未知 |
| **TLS/mTLS** | ✅ 支援 | ❌ 不支援 |
| **自訂 Headers** | ✅ 支援 | ❌ 不支援 |
| **隱私控制** | ✅ 細粒度 | ✅ 預設不收集 |
| **Managed Settings** | ✅ 支援 | ❌ 不支援 |
| **官方文件** | ✅ 完整 | ⚠️ 基本說明 |
| **Token 用量追蹤** | ✅ 內建 | ⚠️ Issue 請求中 |
| **成本追蹤** | ✅ 內建 | ❌ 不支援 |
| **Session 統計** | ✅ 內建 | ⚠️ 透過 spans |

### 圖例

- ✅ = 支援 / 已發布
- ⚠️ = 部分支援 / 實驗性 / 未知
- ❌ = 不支援

---

## 4. 詳細功能比較

### 4.1 配置方式

**Claude Code：環境變數為主**
```bash
# 簡單明瞭，所有設定都是環境變數
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

**OpenCode：JSON 配置檔**
```jsonc
// .opencode/opencode.jsonc
{
  "experimental": {
    "openTelemetry": true
  }
}
```

**評比：** Claude Code 的環境變數方式更適合 CI/CD 和容器化部署 ( •̀ω•́ )✧

### 4.2 隱私保護

**Claude Code：**
| 資料類型 | 預設行為 |
|---------|---------|
| API Key | ❌ 永不記錄 |
| 檔案內容 | ❌ 永不記錄 |
| User Prompt | ⚠️ 只記錄長度 |
| Tool 輸出 | ⚠️ 預設不記錄 |
| Session ID | ✅ 記錄 |
| Token 用量 | ✅ 記錄 |

**OpenCode：**

| 資料類型 | 預設行為 |
|---------|---------|
| 預設收集 | ❌ 不收集任何 telemetry |
| OTEL 啟用後 | ✅ 資料送到使用者指定的 endpoint |
| Opt-out | ✅ 預設就是 opt-out |

維護者在 [Issue #459](https://github.com/sst/opencode/issues/459) 明確表示：*"there is no telemetry collected"*

**評比：** 兩者都注重隱私！Claude Code 提供更細粒度的控制，OpenCode 則採取預設不收集的策略 ( •̀ω•́ )✧

### 4.3 企業適用性

**Claude Code 企業功能：**
1. **Managed Settings** - IT 可以集中配置全公司的 telemetry 設定
2. **mTLS 支援** - 企業級安全連線
3. **動態 Token** - 支援 helper script 取得認證 token
4. **Resource Attributes** - 可標記 department、team、cost_center

**OpenCode 企業功能：**
- 目前沒有企業專屬功能
- 需要自己建構相關機制

**評比：** Claude Code 明顯更適合企業部署 ヽ(>∀<☆)☆

### 4.4 生態系統整合

**Claude Code 已驗證支援：**
- Grafana Cloud
- Honeycomb
- Datadog
- SigNoz
- Jaeger
- Prometheus

**OpenCode：**
- 尚無官方整合文件
- 社群嘗試中（Docker-based 方案）

---

## 5. 優缺點總結

### Claude Code

**優點 ヽ(>∀<☆)☆**
- 正式支援，有官方文件
- 隱私控制完善
- 企業功能齊全
- 多平台整合
- 環境變數配置簡單

**缺點 (´・ω・`)**
- 閉源，無法自己修改
- 只能用 Anthropic 的模型
- 某些進階功能需要付費方案

### OpenCode

**優點 ヽ(>∀<☆)☆**
- 完全開源 (MIT)
- 支援多種 AI 模型
- 社群活躍 (80k+ stars)
- 可自由修改

**缺點 (´・ω・`)**
- OTEL 支援還在實驗階段（v1.0.134+）
- 官方文件較少（但有在改善）
- 只支援 Traces，尚無 Metrics/Logs
- 企業功能不足

---

## 6. 結論與建議

### 6.1 現況總結（2026-01-21 更新）

```
Claude Code OTEL 支援度：████████░░ 80%（正式支援，功能完整）
OpenCode OTEL 支援度：  ███░░░░░░░ 35%（實驗性，基礎 Traces 支援）
```

**現況：** v1.0.134 透過 [PR #4978](https://github.com/sst/opencode/pull/4978) 發布了基礎 OTEL 實驗性支援 ( ˘•ω•˘ )

> ⚠️ **重要更正：** [PR #5245](https://github.com/sst/opencode/pull/5245) 是後續的增強 PR，**目前仍為 Open 狀態，尚未合併**。因此 OpenCode 的 OTEL 功能比原先預估的更為基礎。

Claude Code 仍然大幅領先。OpenCode 的 OTEL 功能已從「開發中」進展到「基礎可用」，但完整的 tracing 功能仍待 PR #5245 合併後才會完善。

### 6.2 選擇建議

**選 Claude Code 如果你需要：**
- 立即可用的 observability
- 成本追蹤和 token 統計
- 企業級安全和管理功能
- 與現有監控系統整合

**選 OpenCode 如果你需要：**
- 完全開源的解決方案
- 使用非 Anthropic 的模型
- 願意等待功能成熟
- 有能力貢獻程式碼

### 6.3 給 OpenCode 團隊的建議

✅ **已完成：** PR #4978 已合併，v1.0.134 發布基礎 OTEL 支援！
⏳ **進行中：** PR #5245（增強 OTEL tracing）仍為 Open 狀態

下一步建議：

1. **合併 PR #5245** - 完成更完整的 OTEL tracing 支援
2. **擴充官方文件** - 補充 OTEL 配置範例和整合教學
3. **實作 Metrics 支援** - 目前只有 Traces，Metrics 是社群高需求功能
4. **實作 per-interaction telemetry** - Issue #2666 的 12 個 thumbs up 說明需求
5. **支援 Prometheus** - 這是企業最常用的監控系統
6. **考慮 Logs 支援** - 完整的 OTEL 三大支柱

### 6.4 未來展望

OpenCode 作為開源專案，有潛力追上甚至超越 Claude Code：
- 社群可以貢獻各種 exporter
- 可以根據需求客製化
- 80k stars 顯示社群動能很強
- **v1.0.134 里程碑顯示團隊有在推進 OTEL！**

選擇建議：
- **現在就需要完整 OTEL？** → Claude Code 更成熟
- **只需要基本 Traces？** → OpenCode v1.0.134+ 已經可用！
- **想要開源彈性？** → OpenCode + 自訂擴展

兩者都是好選擇，看你的需求而定 ( ˘•ω•˘ )

---

## 參考資源

### Claude Code
- [官方文件 - Monitoring Usage](https://docs.anthropic.com/en/docs/claude-code/telemetry) - 最新 OTEL 配置說明
- [Grafana 整合教學](https://quesma.com/blog/track-claude-code-usage-and-limits-with-grafana-cloud/)
- [SigNoz 整合指南](https://signoz.io/docs/claude-code-monitoring/) - 包含 Traces 整合範例
- [Honeycomb 深度分析](https://www.honeycomb.io/blog/can-claude-code-observe-its-own-code) - Claude Code span 分析

### OpenCode
- [GitHub Repo](https://github.com/sst/opencode)
- [Release v1.0.134](https://github.com/sst/opencode/releases/tag/v1.0.134) - OTEL 實驗性支援發布！
- [OTEL 基礎支援 PR #4978](https://github.com/sst/opencode/pull/4978) - 已合併，v1.0.134 的 OTEL 來源
- [OTEL 增強 PR #5245](https://github.com/sst/opencode/pull/5245) - ⏳ Open 狀態，尚未合併
- [Per-Interaction Telemetry Issue #2666](https://github.com/sst/opencode/issues/2666)
- [隱私說明 Issue #459](https://github.com/sst/opencode/issues/459) - 維護者確認不收集 telemetry

### OpenTelemetry
- [官方文件](https://opentelemetry.io/docs/)
- [環境變數規範](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)
- [AI Agent Observability 最佳實踐](https://opentelemetry.io/blog/2025/ai-agent-observability/)

---

*研究完成！兩邊都有各自的優缺點，選擇適合自己的就好 (´▽`)*
