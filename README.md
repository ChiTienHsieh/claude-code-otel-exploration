# Claude Code OpenTelemetry Exploration

探索如何使用 OpenTelemetry 監控 Claude Code 的完整指南。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-v2.1.1+-blue)](https://docs.anthropic.com/en/docs/claude-code)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-1.0-blueviolet)](https://opentelemetry.io/)
[![Verify OTEL Stack](https://github.com/ChiTienHsieh/claude-code-otel-exploration/actions/workflows/verify.yml/badge.svg)](https://github.com/ChiTienHsieh/claude-code-otel-exploration/actions/workflows/verify.yml)

## Disclaimer

> ⚠️ **This is NOT official Anthropic documentation.**
>
> 這是個人探索和學習筆記，不代表 Anthropic 官方立場。
> 內容可能隨 Claude Code 版本更新而過時。
> 請參考[官方文件](https://docs.anthropic.com/en/docs/claude-code)獲取最新資訊。

## 這是什麼？

這個 repo 包含：

1. **完整教學** - 如何設定 Claude Code 的 OpenTelemetry 輸出
2. **比較研究** - Claude Code vs OpenCode 的 OTEL 支援度
3. **驗證環境** - 一鍵啟動的 Docker 環境，讓你實際測試

## 內容一覽

| 檔案 | 說明 | 語言 |
|------|------|------|
| [TUTORIAL_OTEL_zh-TW.md](./TUTORIAL_OTEL_zh-TW.md) | 完整 OTEL 設定教學 | 繁體中文 |
| [OPENCODE_OTEL_COMPARISON.md](./OPENCODE_OTEL_COMPARISON.md) | Claude Code vs OpenCode 比較 | 繁體中文 |
| [verify/](./verify/) | 可驗證的 Docker 測試環境 | - |

## Quick Start: 驗證環境

想要親手試試？這個 repo 提供一鍵啟動的測試環境：

```bash
# Clone
git clone https://github.com/ChiTienHsieh/claude-code-otel-exploration
cd claude-code-otel-exploration/verify

# 啟動 observability stack
docker compose up -d

# 設定 Claude Code 環境變數
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# 使用 Claude Code
claude "hello world"

# 查看結果
# NOTE: admin/admin is the default credential - CHANGE IN PRODUCTION!
open http://localhost:3000  # Grafana (admin/admin)
open http://localhost:16686 # Jaeger
open http://localhost:9090  # Prometheus
```

詳細說明：[verify/README.md](./verify/README.md)

## 教學大綱

### TUTORIAL_OTEL_zh-TW.md

1. **環境變數完整參考** - 40+ OTEL 相關環境變數
2. **隱私控制** - 如何保護敏感資料
3. **Resource Attributes** - 多團隊環境標籤
4. **進階調校** - Enterprise 級設定
5. **OTEL Collector 架設** - Docker/Docker Compose 範例
6. **完整設定範例** - Local/Enterprise 情境
7. **輸出資料說明** - Metrics/Events/Attributes

### OPENCODE_OTEL_COMPARISON.md

- OpenCode 簡介（80.5k stars 的開源替代方案）
- OTEL 支援度比較（Claude Code 80% vs OpenCode 35%）
- 功能對照表
- 優缺點分析
- 使用情境建議

## 測試環境架構

```
┌─────────────────┐     OTLP      ┌──────────────────┐
│   Claude Code   │ ───────────── │  OTEL Collector  │
└─────────────────┘               └────────┬─────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
            ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
            │  Prometheus  │      │    Jaeger    │      │   Console    │
            │  (Metrics)   │      │   (Traces)   │      │   (Debug)    │
            └──────┬───────┘      └──────────────┘      └──────────────┘
                   │
                   ▼
            ┌──────────────┐
            │   Grafana    │
            │  (Dashboard) │
            └──────────────┘
```

## 環境需求

- **Claude Code** v2.1.1 或更新版本
- **Docker** & **Docker Compose** (測試環境用)
- 有效的 **Anthropic API Key**

## 官方資源

- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code Telemetry Docs](https://docs.anthropic.com/en/docs/claude-code/telemetry)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

## Contributing

發現錯誤或有建議？歡迎開 Issue 或 PR！

特別歡迎：
- 錯誤修正
- 新版本 Claude Code 的更新
- 英文翻譯
- 其他 observability backend 整合範例

## License

MIT License - 詳見 [LICENSE](./LICENSE)
