# 進階文件

> **最後更新**：2026-02-03

本目錄包含主教學中未完整探討的進階主題文件。這些指南填補了初始 repository 涵蓋範圍分析中所識別的空白。

## 文件索引

### 核心概念

| 文件 | 說明 | 優先級 |
|------|------|--------|
| [01 - Traces 深度探索](./01-traces-deep-dive.md) | 了解 traces 支援、設定和測試 | 關鍵 |
| [09 - Jaeger 與分散式追蹤](./09-jaeger-tracing.md) | 使用 Jaeger 進行 trace 視覺化與分析 | 重要 |

### 部署與維運

| 文件 | 說明 | 優先級 |
|------|------|--------|
| [02 - 正式環境部署](./02-production-deployment.md) | Kubernetes、高可用性、擴展模式 | 關鍵 |
| [10 - 遷移與升級](./10-migration-upgrade.md) | 遷移路徑和升級程序 | 重要 |
| [11 - 安全強化](./11-security-hardening.md) | TLS、認證、資料隱私、合規性 | 關鍵 |

### 監控與分析

| 文件 | 說明 | 優先級 |
|------|------|--------|
| [03 - 成本優化](./03-cost-optimization.md) | 使用 metrics 分析和降低成本 | 關鍵 |
| [06 - Grafana Dashboards](./06-grafana-dashboards.md) | 建立和自訂 dashboards | 重要 |
| [08 - 資料匯出與分析](./08-data-export-analysis.md) | PromQL 查詢和資料分析工作流程 | 重要 |

### 整合

| 文件 | 說明 | 優先級 |
|------|------|--------|
| [05 - Backend 整合](./05-backend-integrations.md) | Datadog、AWS、GCP、New Relic 等 | 重要 |
| [07 - CI/CD 整合](./07-cicd-integration.md) | GitHub Actions、GitLab CI、Jenkins 模式 | 重要 |

### 疑難排解

| 文件 | 說明 | 優先級 |
|------|------|--------|
| [04 - 疑難排解手冊](./04-troubleshooting.md) | 常見問題和診斷程序 | 關鍵 |

## 快速導覽

### 依使用案例

**「我想開始使用 OTEL 監控」**
1. [主教學](../TUTORIAL_OTEL_zh-TW.md) - 基本設定
2. [疑難排解手冊](./04-troubleshooting.md) - 遇到問題時

**「我想部署到正式環境」**
1. [正式環境部署](./02-production-deployment.md) - K8s、高可用性、擴展
2. [安全強化](./11-security-hardening.md) - 保護您的部署
3. [遷移指南](./10-migration-upgrade.md) - 規劃您的上線

**「我想了解並降低成本」**
1. [成本優化](./03-cost-optimization.md) - 分析與優化
2. [Grafana Dashboards](./06-grafana-dashboards.md) - 視覺化成本
3. [資料匯出與分析](./08-data-export-analysis.md) - 深度分析

**「我想與現有工具整合」**
1. [Backend 整合](./05-backend-integrations.md) - 連接您的 stack
2. [CI/CD 整合](./07-cicd-integration.md) - 使用 pipelines 自動化

**「我想了解 tracing」**
1. [Traces 深度探索](./01-traces-deep-dive.md) - Trace 基礎
2. [Jaeger Tracing](./09-jaeger-tracing.md) - 視覺化與分析

## 涵蓋範圍摘要

### 先前已記錄（主教學）
- 環境變數設定
- 隱私控制
- Resource attributes
- OTEL Collector 基本設定
- Metrics 和 logs exporters
- 驗證環境

### 現已記錄（本目錄）

| 領域 | 狀態 | 文件 |
|------|------|------|
| Traces 支援 | 已記錄 | [01-traces-deep-dive.md](./01-traces-deep-dive.md) |
| 正式環境部署 | 已記錄 | [02-production-deployment.md](./02-production-deployment.md) |
| 成本優化 | 已記錄 | [03-cost-optimization.md](./03-cost-optimization.md) |
| 疑難排解 | 已記錄 | [04-troubleshooting.md](./04-troubleshooting.md) |
| Backend 整合 | 已記錄 | [05-backend-integrations.md](./05-backend-integrations.md) |
| Grafana Dashboards | 已記錄 | [06-grafana-dashboards.md](./06-grafana-dashboards.md) |
| CI/CD 整合 | 已記錄 | [07-cicd-integration.md](./07-cicd-integration.md) |
| 資料匯出與分析 | 已記錄 | [08-data-export-analysis.md](./08-data-export-analysis.md) |
| Jaeger Tracing | 已記錄 | [09-jaeger-tracing.md](./09-jaeger-tracing.md) |
| 遷移與升級 | 已記錄 | [10-migration-upgrade.md](./10-migration-upgrade.md) |
| 安全強化 | 已記錄 | [11-security-hardening.md](./11-security-hardening.md) |

## 文件關係圖

```
                    ┌──────────────────────────┐
                    │         主教學           │
                    │ (TUTORIAL_OTEL_zh-TW.md) │
                    └────────────┬─────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   核心文件    │    │    維運       │    │    整合      │
├───────────────┤    ├───────────────┤    ├───────────────┤
│ 01-Traces     │    │ 02-部署       │    │ 05-Backends   │
│ 09-Jaeger     │    │ 04-疑難排解   │    │ 07-CI/CD      │
│               │    │ 10-遷移       │    │               │
│               │    │ 11-安全       │    │               │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                             ▼
                    ┌───────────────┐
                    │     分析      │
                    ├───────────────┤
                    │ 03-成本       │
                    │ 06-Grafana    │
                    │ 08-資料匯出   │
                    └───────────────┘
```

## 貢獻

發現錯誤或想要新增更多內容？歡迎貢獻：

1. Fork 這個 repository
2. 建立 feature branch
3. 新增或更新文件
4. 提交 pull request

### 文件標準

- 使用清晰、簡潔的語言
- 為所有設定包含程式碼範例
- 為複雜概念加入圖表
- 發布前測試所有程式碼片段
- 保持目錄更新

## 意見回饋

如果您發現問題或有建議：

- 開啟 GitHub issue
- 加上 `documentation` 標籤
- 包含具體的檔案和章節參考

## 授權

MIT License - 參見 [LICENSE](../LICENSE)
