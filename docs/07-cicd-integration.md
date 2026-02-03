# CI/CD Integration Patterns

> **Last Updated**: 2026-02-03

This guide covers integrating Claude Code OTEL monitoring with various CI/CD platforms for automated observability, cost tracking, and quality gates.

## Table of Contents

- [Overview](#overview)
- [GitHub Actions](#github-actions)
- [GitLab CI](#gitlab-ci)
- [Jenkins](#jenkins)
- [CircleCI](#circleci)
- [Azure DevOps](#azure-devops)
- [Cost-Based Quality Gates](#cost-based-quality-gates)
- [Automated Reporting](#automated-reporting)

## Overview

### Benefits of CI/CD Integration

1. **Automated Telemetry Collection**: Ensure all CI Claude Code usage is tracked
2. **Cost Visibility**: Track costs per pipeline, branch, and PR
3. **Quality Gates**: Block expensive operations from merging
4. **Audit Trail**: Complete history of Claude Code usage in CI

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CI/CD Pipeline                            │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐         │
│  │  Build   │───▶│ Claude Code  │───▶│   Test        │         │
│  │  Stage   │    │  Analysis    │    │   Stage       │         │
│  └──────────┘    └──────┬───────┘    └───────────────┘         │
│                         │                                       │
│                         │ OTLP                                  │
│                         ▼                                       │
│               ┌──────────────────┐                              │
│               │  OTEL Collector  │                              │
│               └────────┬─────────┘                              │
│                        │                                        │
└────────────────────────┼────────────────────────────────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │    Prometheus /    │
              │   Observability    │
              │     Backend        │
              └────────────────────┘
```

## GitHub Actions

### Basic Integration

```yaml
# .github/workflows/claude-code-analysis.yml
name: Claude Code Analysis

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  CLAUDE_CODE_ENABLE_TELEMETRY: "1"
  OTEL_EXPORTER_OTLP_ENDPOINT: ${{ secrets.OTEL_ENDPOINT }}
  OTEL_EXPORTER_OTLP_HEADERS: ${{ secrets.OTEL_HEADERS }}
  OTEL_EXPORTER_OTLP_PROTOCOL: grpc
  OTEL_RESOURCE_ATTRIBUTES: >-
    service.name=claude-code-ci,
    team=${{ github.repository_owner }},
    environment=ci,
    ci.pipeline.id=${{ github.run_id }},
    ci.pipeline.name=${{ github.workflow }},
    vcs.repository.name=${{ github.repository }},
    vcs.ref.name=${{ github.ref_name }},
    vcs.commit.sha=${{ github.sha }}

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Claude Code
        run: |
          npm install -g @anthropic-ai/claude-code
          claude --version

      - name: Run Code Analysis
        id: analysis
        run: |
          # Run Claude Code with telemetry
          claude "analyze the code quality of this PR" \
            --output-format json > analysis.json

          # Extract metrics for GitHub summary
          echo "## Code Analysis Results" >> $GITHUB_STEP_SUMMARY
          cat analysis.json | jq -r '.summary' >> $GITHUB_STEP_SUMMARY

      - name: Upload Analysis Report
        uses: actions/upload-artifact@v4
        with:
          name: claude-analysis
          path: analysis.json
```

### With Cost Tracking

```yaml
# .github/workflows/claude-with-cost-tracking.yml
name: Claude Code with Cost Tracking

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  analyze-with-budget:
    runs-on: ubuntu-latest
    env:
      MAX_COST_PER_PR: "5.00"

    steps:
      - uses: actions/checkout@v4

      - name: Setup
        run: npm install -g @anthropic-ai/claude-code

      - name: Configure Telemetry
        run: |
          export CLAUDE_CODE_ENABLE_TELEMETRY=1
          export OTEL_METRICS_EXPORTER=console,otlp
          export OTEL_RESOURCE_ATTRIBUTES="pr=${{ github.event.pull_request.number }},repo=${{ github.repository }}"

      - name: Run Analysis
        id: claude
        run: |
          # Capture output including metrics
          claude "review this PR" 2>&1 | tee output.log

          # Parse cost from OTEL output
          COST=$(grep "claude_code_cost_total" output.log | tail -1 | awk '{print $2}')
          echo "cost=$COST" >> $GITHUB_OUTPUT

      - name: Check Budget
        run: |
          COST="${{ steps.claude.outputs.cost }}"
          MAX="${{ env.MAX_COST_PER_PR }}"

          if (( $(echo "$COST > $MAX" | bc -l) )); then
            echo "::error::Cost $COST exceeds budget $MAX"
            exit 1
          fi
          echo "Cost $COST is within budget $MAX"

      - name: Report Cost
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Claude Code Analysis Cost\n\nThis PR analysis cost: **$${${{ steps.claude.outputs.cost }}}**`
            })
```

### Reusable Workflow

```yaml
# .github/workflows/claude-code-reusable.yml
name: Claude Code Reusable Workflow

on:
  workflow_call:
    inputs:
      prompt:
        required: true
        type: string
      max-cost:
        required: false
        type: string
        default: "10.00"
    secrets:
      ANTHROPIC_API_KEY:
        required: true
      OTEL_ENDPOINT:
        required: true

jobs:
  claude-code:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.run.outputs.result }}
      cost: ${{ steps.run.outputs.cost }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Claude Code
        id: run
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          CLAUDE_CODE_ENABLE_TELEMETRY: "1"
          OTEL_EXPORTER_OTLP_ENDPOINT: ${{ secrets.OTEL_ENDPOINT }}
        run: |
          result=$(claude "${{ inputs.prompt }}" 2>&1)
          echo "result<<EOF" >> $GITHUB_OUTPUT
          echo "$result" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
```

## GitLab CI

### Basic Integration

```yaml
# .gitlab-ci.yml
variables:
  CLAUDE_CODE_ENABLE_TELEMETRY: "1"
  OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_ENDPOINT}
  OTEL_EXPORTER_OTLP_HEADERS: ${OTEL_HEADERS}
  OTEL_RESOURCE_ATTRIBUTES: >-
    service.name=claude-code-ci,
    team=${CI_PROJECT_NAMESPACE},
    environment=ci,
    ci.pipeline.id=${CI_PIPELINE_ID},
    ci.job.name=${CI_JOB_NAME},
    vcs.repository.name=${CI_PROJECT_PATH},
    vcs.ref.name=${CI_COMMIT_REF_NAME},
    vcs.commit.sha=${CI_COMMIT_SHA}

stages:
  - analyze
  - test
  - deploy

code-analysis:
  stage: analyze
  image: node:20
  before_script:
    - npm install -g @anthropic-ai/claude-code
  script:
    - claude "analyze code quality and suggest improvements"
  artifacts:
    reports:
      dotenv: claude-metrics.env
  rules:
    - if: $CI_MERGE_REQUEST_ID

security-review:
  stage: analyze
  image: node:20
  before_script:
    - npm install -g @anthropic-ai/claude-code
  script:
    - |
      claude "review for security vulnerabilities" \
        --output-format json > security-report.json
  artifacts:
    paths:
      - security-report.json
    reports:
      sast: security-report.json
  rules:
    - if: $CI_MERGE_REQUEST_ID
```

### With Cost Gates

```yaml
# .gitlab-ci.yml
.claude-code-template: &claude-code
  image: node:20
  before_script:
    - npm install -g @anthropic-ai/claude-code
    - export CLAUDE_CODE_ENABLE_TELEMETRY=1

claude-analyze:
  <<: *claude-code
  stage: analyze
  script:
    - |
      # Run with cost tracking
      claude "analyze this code" 2>&1 | tee output.log

      # Extract and check cost
      COST=$(grep -oP 'cost_total\{[^}]*\}\s+\K[\d.]+' output.log | tail -1 || echo "0")
      echo "CLAUDE_COST=$COST" >> claude-metrics.env

      # Check against budget
      if [ $(echo "$COST > $MAX_COST" | bc -l) -eq 1 ]; then
        echo "Cost $COST exceeds maximum $MAX_COST"
        exit 1
      fi
  variables:
    MAX_COST: "5.00"
  artifacts:
    reports:
      dotenv: claude-metrics.env

report-costs:
  stage: deploy
  needs: [claude-analyze]
  script:
    - |
      # Post cost to MR
      curl --request POST \
        --header "PRIVATE-TOKEN: ${GITLAB_API_TOKEN}" \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes" \
        --data "body=Claude Code cost for this pipeline: \$${CLAUDE_COST}"
  rules:
    - if: $CI_MERGE_REQUEST_ID
```

## Jenkins

### Jenkinsfile

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        CLAUDE_CODE_ENABLE_TELEMETRY = '1'
        OTEL_EXPORTER_OTLP_ENDPOINT = credentials('otel-endpoint')
        OTEL_EXPORTER_OTLP_HEADERS = credentials('otel-headers')
        OTEL_RESOURCE_ATTRIBUTES = """
            service.name=claude-code-ci,
            team=${env.JOB_NAME.split('/')[0]},
            environment=ci,
            ci.pipeline.id=${env.BUILD_ID},
            ci.pipeline.name=${env.JOB_NAME},
            vcs.repository.name=${env.GIT_URL},
            vcs.ref.name=${env.GIT_BRANCH},
            vcs.commit.sha=${env.GIT_COMMIT}
        """.replaceAll('\\s+', '')
    }

    stages {
        stage('Setup') {
            steps {
                sh 'npm install -g @anthropic-ai/claude-code'
            }
        }

        stage('Code Analysis') {
            steps {
                script {
                    def result = sh(
                        script: 'claude "analyze code quality" 2>&1',
                        returnStdout: true
                    )
                    echo result

                    // Parse cost from output
                    def costMatch = result =~ /cost_total\{[^}]*\}\s+([\d.]+)/
                    if (costMatch) {
                        env.CLAUDE_COST = costMatch[0][1]
                    }
                }
            }
        }

        stage('Cost Check') {
            when {
                expression { env.CLAUDE_COST?.toFloat() > 10.0 }
            }
            steps {
                error "Claude Code cost ${env.CLAUDE_COST} exceeds budget"
            }
        }
    }

    post {
        always {
            script {
                // Record cost in Jenkins metrics
                if (env.CLAUDE_COST) {
                    currentBuild.description = "Claude Cost: \$${env.CLAUDE_COST}"
                }
            }
        }
    }
}
```

### Jenkins Shared Library

```groovy
// vars/claudeCode.groovy
def call(Map config = [:]) {
    def prompt = config.prompt ?: 'analyze this code'
    def maxCost = config.maxCost ?: 10.0

    withEnv([
        'CLAUDE_CODE_ENABLE_TELEMETRY=1',
        "OTEL_RESOURCE_ATTRIBUTES=ci.job=${env.JOB_NAME}"
    ]) {
        def output = sh(
            script: "claude '${prompt}' 2>&1",
            returnStdout: true
        )

        def cost = parseCost(output)
        if (cost > maxCost) {
            error "Cost ${cost} exceeds maximum ${maxCost}"
        }

        return [output: output, cost: cost]
    }
}

def parseCost(String output) {
    def matcher = output =~ /cost_total\{[^}]*\}\s+([\d.]+)/
    return matcher ? matcher[0][1].toFloat() : 0.0
}
```

## CircleCI

### Config.yml

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  node: circleci/node@5.0

executors:
  claude-executor:
    docker:
      - image: cimg/node:20.0
    environment:
      CLAUDE_CODE_ENABLE_TELEMETRY: "1"

commands:
  setup-claude:
    steps:
      - run:
          name: Install Claude Code
          command: npm install -g @anthropic-ai/claude-code

  run-claude:
    parameters:
      prompt:
        type: string
      max-cost:
        type: string
        default: "10.00"
    steps:
      - run:
          name: Run Claude Code
          command: |
            export OTEL_RESOURCE_ATTRIBUTES="ci.pipeline=${CIRCLE_WORKFLOW_ID},ci.job=${CIRCLE_JOB}"
            claude "<< parameters.prompt >>" 2>&1 | tee claude-output.log

            # Check cost
            COST=$(grep -oP 'cost_total\{[^}]*\}\s+\K[\d.]+' claude-output.log | tail -1 || echo "0")
            if (( $(echo "$COST > << parameters.max-cost >>" | bc -l) )); then
              echo "Cost $COST exceeds budget << parameters.max-cost >>"
              exit 1
            fi

jobs:
  analyze:
    executor: claude-executor
    steps:
      - checkout
      - setup-claude
      - run-claude:
          prompt: "analyze this repository for code quality issues"
          max-cost: "5.00"
      - store_artifacts:
          path: claude-output.log

  security-review:
    executor: claude-executor
    steps:
      - checkout
      - setup-claude
      - run-claude:
          prompt: "review for security vulnerabilities"
          max-cost: "10.00"

workflows:
  pr-analysis:
    jobs:
      - analyze:
          context: claude-code-context
      - security-review:
          context: claude-code-context
          requires:
            - analyze
```

## Azure DevOps

### Azure Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
  - group: claude-code-secrets
  - name: CLAUDE_CODE_ENABLE_TELEMETRY
    value: '1'

stages:
  - stage: Analysis
    jobs:
      - job: ClaudeCodeAnalysis
        pool:
          vmImage: ubuntu-latest

        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'

          - script: npm install -g @anthropic-ai/claude-code
            displayName: Install Claude Code

          - script: |
              export OTEL_RESOURCE_ATTRIBUTES="ci.pipeline=$(Build.BuildId),ci.stage=$(System.StageName)"
              claude "analyze code quality" 2>&1 | tee $(Build.ArtifactStagingDirectory)/analysis.log
            displayName: Run Analysis
            env:
              ANTHROPIC_API_KEY: $(ANTHROPIC_API_KEY)
              OTEL_EXPORTER_OTLP_ENDPOINT: $(OTEL_ENDPOINT)

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: $(Build.ArtifactStagingDirectory)
              artifactName: claude-analysis

          - script: |
              COST=$(grep -oP 'cost_total\{[^}]*\}\s+\K[\d.]+' $(Build.ArtifactStagingDirectory)/analysis.log | tail -1)
              echo "##vso[task.setvariable variable=claudeCost]$COST"
              echo "##vso[build.addbuildtag]claude-cost-$COST"
            displayName: Extract Cost

      - job: CostGate
        dependsOn: ClaudeCodeAnalysis
        condition: gt(variables['claudeCost'], '10.00')
        steps:
          - script: |
              echo "Cost $(claudeCost) exceeds budget"
              exit 1
            displayName: Budget Check Failed
```

## Cost-Based Quality Gates

### Prometheus Alert for CI Costs

```yaml
# prometheus-alerts.yaml
groups:
  - name: ci-cost-alerts
    rules:
      - alert: CIPipelineCostExceeded
        expr: |
          sum(increase(claude_code_cost_total{environment="ci"}[1h])) by (ci_pipeline_id)
          > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CI pipeline {{ $labels.ci_pipeline_id }} exceeded cost budget"

      - alert: DailyCI CostExceeded
        expr: |
          sum(increase(claude_code_cost_total{environment="ci"}[24h])) > 500
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Daily CI Claude Code costs exceeded $500"
```

### GitHub Action Cost Gate

```yaml
# .github/actions/claude-cost-gate/action.yml
name: Claude Cost Gate
description: Check Claude Code costs against budget

inputs:
  prometheus-url:
    description: Prometheus URL
    required: true
  max-cost:
    description: Maximum allowed cost
    required: true
    default: "10.00"
  pipeline-id:
    description: Pipeline identifier
    required: true

runs:
  using: composite
  steps:
    - shell: bash
      run: |
        COST=$(curl -s "${{ inputs.prometheus-url }}/api/v1/query" \
          --data-urlencode "query=sum(claude_code_cost_total{ci_pipeline_id=\"${{ inputs.pipeline-id }}\"})" \
          | jq -r '.data.result[0].value[1] // "0"')

        echo "Pipeline cost: $COST"

        if (( $(echo "$COST > ${{ inputs.max-cost }}" | bc -l) )); then
          echo "::error::Cost $COST exceeds budget ${{ inputs.max-cost }}"
          exit 1
        fi
```

## Automated Reporting

### Daily Cost Report Workflow

```yaml
# .github/workflows/daily-cost-report.yml
name: Daily Claude Code Cost Report

on:
  schedule:
    - cron: '0 9 * * *'  # 9 AM daily
  workflow_dispatch:

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - name: Query Costs
        id: costs
        run: |
          # Query Prometheus for daily costs
          RESPONSE=$(curl -s "${{ secrets.PROMETHEUS_URL }}/api/v1/query" \
            --data-urlencode "query=sum(increase(claude_code_cost_total[24h])) by (team)")

          echo "data<<EOF" >> $GITHUB_OUTPUT
          echo "$RESPONSE" | jq -r '.data.result[] | "- \(.metric.team): $\(.value[1])"' >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

          TOTAL=$(echo "$RESPONSE" | jq -r '[.data.result[].value[1] | tonumber] | add')
          echo "total=$TOTAL" >> $GITHUB_OUTPUT

      - name: Create Issue
        uses: actions/github-script@v7
        with:
          script: |
            const date = new Date().toISOString().split('T')[0];
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Daily Claude Code Cost Report - ${date}`,
              body: `## Daily Cost Summary\n\n**Total: $${${{ steps.costs.outputs.total }}**\n\n### By Team\n${{ steps.costs.outputs.data }}`,
              labels: ['cost-report']
            });
```

### Slack Notification

```yaml
# Include in any pipeline
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v1
  with:
    channel-id: ${{ secrets.SLACK_CHANNEL }}
    slack-message: |
      Claude Code CI Run Complete
      - Pipeline: ${{ github.workflow }}
      - Status: ${{ job.status }}
      - Cost: ${{ env.CLAUDE_COST }}
      - <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## Related Documentation

- [Cost Optimization Guide](./03-cost-optimization.md)
- [Grafana Dashboard Development](./06-grafana-dashboards.md)
- [Production Deployment Guide](./02-production-deployment.md)
