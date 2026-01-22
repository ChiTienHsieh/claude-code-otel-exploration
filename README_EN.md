# Claude Code OpenTelemetry Exploration

A comprehensive guide to monitoring Claude Code using OpenTelemetry.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-v2.1.1+-blue)](https://docs.anthropic.com/en/docs/claude-code)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-1.0-blueviolet)](https://opentelemetry.io/)
[![Verify OTEL Stack](https://github.com/ChiTienHsieh/claude-code-otel-exploration/actions/workflows/verify.yml/badge.svg)](https://github.com/ChiTienHsieh/claude-code-otel-exploration/actions/workflows/verify.yml)

## Disclaimer

> **This is NOT official Anthropic documentation.**
>
> This repository contains personal exploration and learning notes. Content may become outdated as Claude Code evolves.
> Please refer to the [official documentation](https://docs.anthropic.com/en/docs/claude-code) for the latest information.

## What's This?

This repository contains:

1. **Complete Tutorial** - How to configure Claude Code's OpenTelemetry output (Chinese)
2. **Comparison Study** - Claude Code vs OpenCode OTEL support comparison (Chinese)
3. **Verification Environment** - One-click Docker environment for hands-on testing

## Contents

| File | Description | Language |
|------|-------------|----------|
| [TUTORIAL_OTEL_zh-TW.md](./TUTORIAL_OTEL_zh-TW.md) | Complete OTEL configuration tutorial | Chinese (zh-TW) |
| [OPENCODE_OTEL_COMPARISON.md](./OPENCODE_OTEL_COMPARISON.md) | Claude Code vs OpenCode comparison | Chinese (zh-TW) |
| [verify/](./verify/) | Verifiable Docker test environment | - |

## Quick Start: Verification Environment

Want to try it yourself? This repo provides a one-click test environment:

```bash
# Clone
git clone https://github.com/ChiTienHsieh/claude-code-otel-exploration
cd claude-code-otel-exploration/verify

# Start observability stack
docker compose up -d

# Configure Claude Code environment variables
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Use Claude Code
claude "hello world"

# View results
open http://localhost:3000  # Grafana (admin/admin)
open http://localhost:16686 # Jaeger
open http://localhost:9090  # Prometheus
```

See [verify/README.md](./verify/README.md) for detailed instructions.

## Architecture

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

## Tutorial Outline

### TUTORIAL_OTEL_zh-TW.md

1. **Environment Variables Reference** - 40+ OTEL-related environment variables
2. **Privacy Controls** - How to protect sensitive data
3. **Resource Attributes** - Multi-team environment tagging
4. **Advanced Tuning** - Enterprise-level configuration
5. **OTEL Collector Setup** - Docker/Docker Compose examples
6. **Complete Configuration Examples** - Local/Enterprise scenarios
7. **Output Data Reference** - Metrics/Events/Attributes

### OPENCODE_OTEL_COMPARISON.md

- OpenCode Introduction (80.5k stars open-source alternative)
- OTEL Support Comparison (Claude Code 80% vs OpenCode 35%)
- Feature Comparison Table
- Pros/Cons Analysis
- Use Case Recommendations

## Requirements

- **Claude Code** v2.1.1 or later
- **Docker** & **Docker Compose** (for test environment)
- Valid **Anthropic API Key**

## Key Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Enable telemetry | `0` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL | - |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Protocol (`grpc`/`http/protobuf`) | `grpc` |
| `OTEL_LOG_USER_PROMPTS` | Log user prompts | `false` |
| `OTEL_LOG_TOOL_CONTENT` | Log tool output | `false` |

## Official Resources

- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code Telemetry Docs](https://docs.anthropic.com/en/docs/claude-code/telemetry)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

## Contributing

Found an error or have suggestions? Issues and PRs are welcome!

Especially welcome:
- Bug fixes
- Updates for new Claude Code versions
- Translations
- Additional observability backend integration examples

## License

MIT License - See [LICENSE](./LICENSE)
