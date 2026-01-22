# Claude Code OTEL Verification Container
#
# 這個 container 讓你可以在隔離環境中測試 Claude Code + OTEL
# 不會影響你本機的設定
#
# Build: docker build -t claude-code-otel-verify ./yolo-cc
# Run:   docker run -it --rm --network verify_otel-net \
#          -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
#          claude-code-otel-verify

FROM node:22-slim

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code

# Create workspace
WORKDIR /workspace

# Pre-configure OTEL settings
# 這些會連到 docker network 裡的 otel-collector
ENV CLAUDE_CODE_ENABLE_TELEMETRY=1
ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
ENV OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Optional: Debug output
ENV OTEL_TRACES_EXPORTER=otlp,console
ENV OTEL_METRICS_EXPORTER=otlp,console
ENV OTEL_LOGS_EXPORTER=otlp,console

# Create a sample project to play with
RUN mkdir -p /workspace/sample-project && \
    echo '{"name": "otel-test", "version": "1.0.0"}' > /workspace/sample-project/package.json && \
    echo '# OTEL Test Project\n\nThis is a sample project for testing Claude Code OTEL integration.' > /workspace/sample-project/README.md

# Default command
CMD ["claude"]
