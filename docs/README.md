# Advanced Documentation

> **Last Updated**: 2026-02-03

This directory contains advanced documentation covering topics not fully explored in the main tutorial. These guides fill the gaps identified in the initial repository coverage analysis.

## Documentation Index

### Core Concepts

| Document | Description | Priority |
|----------|-------------|----------|
| [01 - Traces Deep Dive](./01-traces-deep-dive.md) | Understanding traces support, configuration, and testing | Critical |
| [09 - Jaeger & Distributed Tracing](./09-jaeger-tracing.md) | Using Jaeger for trace visualization and analysis | Important |

### Deployment & Operations

| Document | Description | Priority |
|----------|-------------|----------|
| [02 - Production Deployment](./02-production-deployment.md) | Kubernetes, HA, scaling patterns | Critical |
| [10 - Migration & Upgrade](./10-migration-upgrade.md) | Migration paths and upgrade procedures | Important |
| [11 - Security Hardening](./11-security-hardening.md) | TLS, authentication, data privacy, compliance | Critical |

### Monitoring & Analysis

| Document | Description | Priority |
|----------|-------------|----------|
| [03 - Cost Optimization](./03-cost-optimization.md) | Using metrics to analyze and reduce costs | Critical |
| [06 - Grafana Dashboards](./06-grafana-dashboards.md) | Creating and customizing dashboards | Important |
| [08 - Data Export & Analysis](./08-data-export-analysis.md) | PromQL queries and data analysis workflows | Important |

### Integration

| Document | Description | Priority |
|----------|-------------|----------|
| [05 - Backend Integrations](./05-backend-integrations.md) | Datadog, AWS, GCP, New Relic, and more | Important |
| [07 - CI/CD Integration](./07-cicd-integration.md) | GitHub Actions, GitLab CI, Jenkins patterns | Important |

### Troubleshooting

| Document | Description | Priority |
|----------|-------------|----------|
| [04 - Troubleshooting Playbook](./04-troubleshooting.md) | Common issues and diagnostic procedures | Critical |

## Quick Navigation

### By Use Case

**"I want to get started with OTEL monitoring"**
1. [Main Tutorial](../TUTORIAL_OTEL_zh-TW.md) - Basic setup
2. [Troubleshooting Playbook](./04-troubleshooting.md) - When things go wrong

**"I want to deploy to production"**
1. [Production Deployment](./02-production-deployment.md) - K8s, HA, scaling
2. [Security Hardening](./11-security-hardening.md) - Secure your deployment
3. [Migration Guide](./10-migration-upgrade.md) - Plan your rollout

**"I want to understand and reduce costs"**
1. [Cost Optimization](./03-cost-optimization.md) - Analyze and optimize
2. [Grafana Dashboards](./06-grafana-dashboards.md) - Visualize costs
3. [Data Export & Analysis](./08-data-export-analysis.md) - Deep dive analysis

**"I want to integrate with my existing tools"**
1. [Backend Integrations](./05-backend-integrations.md) - Connect to your stack
2. [CI/CD Integration](./07-cicd-integration.md) - Automate with pipelines

**"I want to understand tracing"**
1. [Traces Deep Dive](./01-traces-deep-dive.md) - Trace fundamentals
2. [Jaeger Tracing](./09-jaeger-tracing.md) - Visualization and analysis

## Coverage Summary

### Previously Documented (Main Tutorial)
- Environment variables configuration
- Privacy controls
- Resource attributes
- OTEL Collector basic setup
- Metrics and logs exporters
- Verification environment

### Now Documented (This Directory)

| Area | Status | Documentation |
|------|--------|---------------|
| Traces Support | Documented | [01-traces-deep-dive.md](./01-traces-deep-dive.md) |
| Production Deployment | Documented | [02-production-deployment.md](./02-production-deployment.md) |
| Cost Optimization | Documented | [03-cost-optimization.md](./03-cost-optimization.md) |
| Troubleshooting | Documented | [04-troubleshooting.md](./04-troubleshooting.md) |
| Backend Integrations | Documented | [05-backend-integrations.md](./05-backend-integrations.md) |
| Grafana Dashboards | Documented | [06-grafana-dashboards.md](./06-grafana-dashboards.md) |
| CI/CD Integration | Documented | [07-cicd-integration.md](./07-cicd-integration.md) |
| Data Export & Analysis | Documented | [08-data-export-analysis.md](./08-data-export-analysis.md) |
| Jaeger Tracing | Documented | [09-jaeger-tracing.md](./09-jaeger-tracing.md) |
| Migration & Upgrade | Documented | [10-migration-upgrade.md](./10-migration-upgrade.md) |
| Security Hardening | Documented | [11-security-hardening.md](./11-security-hardening.md) |

## Document Relationships

```
                    ┌─────────────────────┐
                    │   Main Tutorial     │
                    │ (TUTORIAL_OTEL.md)  │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Core Docs   │    │  Operations   │    │  Integration  │
├───────────────┤    ├───────────────┤    ├───────────────┤
│ 01-Traces     │    │ 02-Production │    │ 05-Backends   │
│ 09-Jaeger     │    │ 04-Troublesh. │    │ 07-CI/CD      │
│               │    │ 10-Migration  │    │               │
│               │    │ 11-Security   │    │               │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                             ▼
                    ┌───────────────┐
                    │   Analysis    │
                    ├───────────────┤
                    │ 03-Cost       │
                    │ 06-Grafana    │
                    │ 08-Data Export│
                    └───────────────┘
```

## Contributing

Found an error or want to add more content? Contributions are welcome:

1. Fork the repository
2. Create a feature branch
3. Add or update documentation
4. Submit a pull request

### Documentation Standards

- Use clear, concise language
- Include code examples for all configurations
- Add diagrams for complex concepts
- Test all code snippets before publishing
- Keep the table of contents updated

## Feedback

If you find issues or have suggestions:

- Open a GitHub issue
- Tag with `documentation` label
- Include specific file and section references

## License

MIT License - See [LICENSE](../LICENSE)
