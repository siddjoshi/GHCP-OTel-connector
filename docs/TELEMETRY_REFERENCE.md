# Copilot OpenTelemetry Data & Dashboard Reference

> Comprehensive catalog of all telemetry signals emitted by GitHub Copilot, where each data point surfaces in the Grafana dashboards, and what derived intelligence is available.
>
> **Grafana URL:** [http://localhost:3010](http://localhost:3010) (admin / admin)

---

## Table of Contents

- [1. Architecture & Data Flow](#1-architecture--data-flow)
- [2. Signal Sources Overview](#2-signal-sources-overview)
- [3. Prometheus Metrics (70 metrics)](#3-prometheus-metrics-70-metrics)
  - [3A. Copilot Chat Metrics](#3a-copilot-chat-metrics)
  - [3B. GenAI Semantic Convention Metrics](#3b-genai-semantic-convention-metrics)
  - [3C. GitHub Copilot Metrics (new naming)](#3c-github-copilot-metrics-new-naming)
  - [3D. Tempo Span-Metrics (auto-generated)](#3d-tempo-span-metrics-auto-generated)
  - [3E. OTel Collector Internal Metrics](#3e-otel-collector-internal-metrics)
- [4. Loki Log Events (4 event types, 26 fields)](#4-loki-log-events-4-event-types-26-fields)
- [5. Tempo Trace Data (spans + events)](#5-tempo-trace-data-spans--events)
- [6. Dashboard Catalog (10 dashboards)](#6-dashboard-catalog-10-dashboards)
- [7. Derived Intelligence & Computed Metrics](#7-derived-intelligence--computed-metrics)
- [8. Data-to-Dashboard Cross-Reference](#8-data-to-dashboard-cross-reference)
- [9. Enabling Content Capture](#9-enabling-content-capture)

---

## 1. Architecture & Data Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│  VS Code + GitHub Copilot Extension                                  │
│  (github.copilot.chat.otel.captureContent = true)                    │
│                                                                      │
│  Emits:  Traces (spans) ──→ OTLP                                     │
│          Metrics ─────────→ OTLP                                     │
│          Logs (events) ───→ OTLP                                     │
└────────────────────┬─────────────────────────────────────────────────┘
                     │ gRPC :4317 / HTTP :4318
                     ▼
         ┌──────────────────────┐
         │  OTel Collector      │
         │  memory_limiter →    │
         │  batch → export      │
         └──┬───────┬───────┬───┘
            │       │       │
   ┌────────┘       │       └────────┐
   ▼                ▼                ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│Prometheus│  │  Tempo    │  │  Loki    │
│ (metrics)│  │ (traces)  │  │  (logs)  │
│ :9090    │  │ :3200     │  │  :3100   │
└────┬─────┘  └────┬──────┘  └────┬────┘
     │             │              │
     │   ┌─────────┘              │
     │   │  span-metrics &        │
     │   │  service-graphs        │
     │   │  (written back to      │
     │◄──┘   Prometheus)          │
     │                            │
     └────────────┬───────────────┘
                  ▼
          ┌──────────────┐
          │   Grafana    │
          │   :3010      │
          │  (10 dashboards)
          └──────────────┘
```

**Key point:** Tempo auto-generates `traces_spanmetrics_*` and service-graph metrics from trace data and writes them back to Prometheus via remote write. This means trace-derived metrics are queryable via PromQL.

---

## 2. Signal Sources Overview

| Signal | Backend | Volume (7 days) | Primary Use |
|--------|---------|-----------------|-------------|
| **Metrics** | Prometheus | 70 relevant time series families | Counters, histograms, rates, percentiles |
| **Logs** | Loki | 1,291 entries across 16 streams | Structured event metadata, token counts, tool details |
| **Traces** | Tempo | Spans across 23+ traces | Full conversation waterfall, prompt/response content, tool call arguments/results |

---

## 3. Prometheus Metrics (70 metrics)

### 3A. Copilot Chat Metrics

These are emitted by the `copilot-chat` service (job=`copilot-chat`).

| Metric | Type | Labels | Description | Dashboards |
|--------|------|--------|-------------|------------|
| `copilot_chat_session_count_total` | counter | `job` | Number of Copilot chat sessions started | [LogQL Insights], [Advanced Metrics] |
| `copilot_chat_time_to_first_token_{bucket,sum,count}` | histogram | `job`, `gen_ai_request_model` | Time-to-first-token (TTFT) per model | [LogQL Insights], [Advanced Metrics] |
| `copilot_chat_agent_turn_count_{bucket,sum,count}` | histogram | `job`, `gen_ai_operation_name` | Number of turns per agent invocation | [Advanced Metrics] |
| `copilot_chat_agent_invocation_duration_{bucket,sum,count}` | histogram | `job` | Total agent invocation duration | *(available, no data yet)* |
| `copilot_chat_tool_call_count_total` | counter | `job`, `gen_ai_tool_name`, `success` | Tool call count per tool with success/failure | [Copilot Insights], [LogQL Insights] |
| `copilot_chat_tool_call_duration_{bucket,sum,count}` | histogram | `job`, `gen_ai_tool_name` | Tool call duration per tool | [LogQL Insights] |

**Models observed** (via `gen_ai_request_model`): `claude-haiku-4.5`, `claude-opus-4.6`, `claude-opus-4.6-1m`, `copilot-nes-oct`, `copilot-suggestions-himalia-001`, `gpt-4.1`, `gpt-4o-mini`, `gpt-4o-mini-2024-07-18`, `gpt-5.4`

### 3B. GenAI Semantic Convention Metrics

Standard [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) metrics.

| Metric | Type | Labels | Description | Dashboards |
|--------|------|--------|-------------|------------|
| `gen_ai_client_token_usage_{bucket,sum,count}` | histogram | `gen_ai_operation_name`, `gen_ai_provider_name`, `gen_ai_response_model`, `gen_ai_token_type`, `job` | Token usage per model and type (input/output) | [Copilot Insights], [LogQL Insights], [Advanced Metrics] |
| `gen_ai_client_operation_duration_{bucket,sum,count}` | histogram | `gen_ai_operation_name`, `gen_ai_response_model`, `job` | Operation duration (unitless) | [LogQL Insights] |
| `gen_ai_client_operation_duration_seconds_{bucket,sum,count}` | histogram | `gen_ai_operation_name`, `gen_ai_response_model`, `job` | Operation duration in seconds | [LogQL Insights], [Advanced Metrics] |

**Labels:**
- `gen_ai_operation_name`: `chat`, `invoke_agent`
- `gen_ai_provider_name`: `github`
- `gen_ai_token_type`: `input`, `output`
- `gen_ai_response_model`: 9 models (claude-haiku-4-5-20251001, claude-haiku-4.5, claude-opus-4-6, claude-opus-4.6, claude-opus-4.6-1m, claude-sonnet-4.5, gpt-4.1-2025-04-14, gpt-4o-mini-2024-07-18, gpt-5.4-2026-03-05)

### 3C. GitHub Copilot Metrics (new naming)

Newer metric naming convention from `github.copilot` scope (job=`github-copilot`).

| Metric | Type | Labels | Description | Dashboards |
|--------|------|--------|-------------|------------|
| `github_copilot_agent_turn_count_{bucket,sum,count}` | histogram | `gen_ai_operation_name`, `job` | Turn count distribution per session | [Advanced Metrics] |
| `github_copilot_tool_call_count_total` | counter | `gen_ai_tool_name`, `success`, `job` | Tool calls with 33 distinct tools | [Advanced Metrics] |
| `github_copilot_tool_call_duration_seconds_{bucket,sum,count}` | histogram | `gen_ai_tool_name`, `job` | Tool call duration in seconds | [Advanced Metrics] |

**Tools observed** (33): `ask_user`, `create`, `edit`, `exit_plan_mode`, `extensions_manage`, `fetch_copilot_cli_documentation`, `github-mcp-server-actions_get`, `github-mcp-server-actions_list`, `github-mcp-server-get_file_contents`, `github-mcp-server-get_job_logs`, `github-mcp-server-issue_read`, `github-mcp-server-list_issues`, `github-mcp-server-list_pull_requests`, `github-mcp-server-pull_request_read`, `github-mcp-server-search_pull_requests`, `glob`, `grep`, `list_agents`, `powershell`, `project_history`, `read_agent`, `read_powershell`, `report_intent`, `skill`, `sql`, `stop_powershell`, `store_memory`, `task`, `task_complete`, `view`, `web_fetch`, `web_search`, `write_agent`

### 3D. Tempo Span-Metrics (auto-generated)

Generated by Tempo's metrics-generator from trace data. Written to Prometheus with exemplars.

| Metric | Type | Labels | Description | Dashboards |
|--------|------|--------|-------------|------------|
| `traces_spanmetrics_calls_total` | counter | `service`, `span_name`, `span_kind`, `status_code` | Span call count | [Copilot Usage], [Copilot Insights], [LogQL Insights], [Metrics Explorer] |
| `traces_spanmetrics_latency_{bucket,sum,count}` | histogram | `service`, `span_name`, `span_kind`, `status_code` | Span duration | [Copilot Usage], [Copilot Insights], [LogQL Insights], [Metrics Explorer] |
| `traces_spanmetrics_size_total` | counter | `service`, `span_name`, `span_kind`, `status_code` | Span payload size in bytes | [Advanced Metrics] |

**Labels:**
- `service`: `copilot-chat`, `github-copilot`, `buildx`
- `span_kind`: `SPAN_KIND_CLIENT`, `SPAN_KIND_INTERNAL`, `SPAN_KIND_SERVER`
- `status_code`: `STATUS_CODE_ERROR`, `STATUS_CODE_OK`, `STATUS_CODE_UNSET`
- `span_name`: 90+ operations including `chat <model>`, `execute_tool <name>`, `invoke_agent <name>`, `hook <name>`, etc.

### 3E. OTel Collector Internal Metrics

Health and performance of the collector itself.

| Metric | Description | Dashboard |
|--------|-------------|-----------|
| `otelcol_receiver_accepted_spans_total` | Spans received | [App Overview], [OTel Collector], [Traces Explorer] |
| `otelcol_receiver_accepted_metric_points_total` | Metric points received | [App Overview], [OTel Collector], [Metrics Explorer] |
| `otelcol_receiver_accepted_log_records_total` | Log records received | [App Overview], [OTel Collector], [Logs Explorer] |
| `otelcol_receiver_refused_spans_total` | Rejected spans | [App Overview], [OTel Collector], [Traces Explorer] |
| `otelcol_receiver_refused_metric_points_total` | Rejected metric points | [OTel Collector] |
| `otelcol_receiver_refused_log_records_total` | Rejected log records | [App Overview] |
| `otelcol_exporter_sent_spans_total` | Spans exported | [App Overview], [OTel Collector], [Traces Explorer] |
| `otelcol_exporter_sent_metric_points_total` | Metric points exported | [App Overview], [Metrics Explorer] |
| `otelcol_exporter_sent_log_records_total` | Log records exported | [App Overview], [Logs Explorer] |
| `otelcol_exporter_send_failed_metric_points_total` | Failed metric exports | [Metrics Explorer] |
| `otelcol_exporter_send_failed_log_records_total` | Failed log exports | [Logs Explorer] |
| `otelcol_process_memory_rss_bytes` | Collector RSS memory | [OTel Collector] |
| `otelcol_processor_batch_batch_send_size_{bucket,sum,count}` | Batch processor sizes | [Traces Explorer] |

---

## 4. Loki Log Events (4 event types, 26 fields)

All logs are stored in Loki as structured metadata. The `service_name` is the only indexed label; all other fields are structured metadata queryable via LogQL.

### Event Type: `copilot_chat.session.start`

Emitted once per chat session.

| Field | Type | Description |
|-------|------|-------------|
| `gen_ai_agent_name` | string | Agent name (e.g., "GitHub Copilot Chat") |
| `gen_ai_request_model` | string | Default model for the session |
| `session_id` | string | Unique session identifier |
| `trace_id` | string | Trace correlation ID |
| `span_id` | string | Span correlation ID |
| `scope_name` | string | Always "copilot-chat" |
| `scope_version` | string | Copilot version (e.g., "0.42.2026032703") |
| `service_version` | string | Service version |

**Dashboards:** [Prompt & Response Explorer] (session timeline, session start logs)

### Event Type: `gen_ai.client.inference.operation.details`

Emitted per AI model inference call. This is the richest event type.

| Field | Type | Cardinality | Description |
|-------|------|-------------|-------------|
| `gen_ai_operation_name` | string | 1 | Always "chat" |
| `gen_ai_request_model` | string | 9 | Requested model name |
| `gen_ai_response_model` | string | 5 | Actual responding model |
| `gen_ai_request_max_tokens` | int | 6 | Max tokens requested (40, 4096, 64000, etc.) |
| `gen_ai_request_temperature` | string | 2 | Temperature setting (0, 0.1) |
| `gen_ai_response_finish_reasons` | string | 1 | How inference ended (e.g., `["stop"]`) |
| `gen_ai_response_id` | string | 245 | Unique response identifier |
| `gen_ai_usage_input_tokens` | int | 305 | Input tokens consumed |
| `gen_ai_usage_output_tokens` | int | 187 | Output tokens generated |
| `session_id` | string | 14 | Session correlation |
| `trace_id` / `span_id` | string | — | Trace correlation |

**Dashboards:** [Copilot Usage], [Copilot Insights], [LogQL Insights], [Prompt & Response Explorer] (inference logs, token analytics, model distribution)

### Event Type: `copilot_chat.agent.turn`

Emitted per agent turn (one LLM round-trip).

| Field | Type | Cardinality | Description |
|-------|------|-------------|-------------|
| `gen_ai_usage_input_tokens` | int | — | Total input tokens for this turn |
| `gen_ai_usage_output_tokens` | int | — | Total output tokens for this turn |
| `tool_call_count` | int | 1 | Number of tool calls in this turn |
| `turn_index` | int | 30 | Turn number within the session (0-based) |
| `session_id` | string | — | Session correlation |
| `trace_id` / `span_id` | string | — | Trace correlation |

**Dashboards:** [Copilot Usage], [Copilot Insights], [LogQL Insights], [Prompt & Response Explorer], [Advanced Metrics]

### Event Type: `copilot_chat.tool.call`

Emitted per tool invocation.

| Field | Type | Cardinality | Description |
|-------|------|-------------|-------------|
| `gen_ai_tool_name` | string | 20 | Tool name (e.g., "grep", "edit", "powershell") |
| `duration_ms` | int | 175 | Tool execution time in milliseconds |
| `success` | boolean | 2 | Whether the tool call succeeded |
| `error_type` | string | 1 | Error classification (e.g., "Error") when `success=false` |
| `session_id` | string | — | Session correlation |
| `trace_id` / `span_id` | string | — | Trace correlation |

**Dashboards:** [Copilot Usage], [Copilot Insights], [LogQL Insights], [Advanced Metrics]

---

## 5. Tempo Trace Data (spans + events)

Traces provide the richest data, including **full prompt/response content** when `captureContent` is enabled.

### Span: `chat <model>` (e.g., `chat claude-opus-4.6-1m`)

Individual LLM inference call — one per model invocation.

| Attribute | Description |
|-----------|-------------|
| `gen_ai.input.messages` | **Full prompt text** — JSON array with role, content, tool calls |
| `gen_ai.output.messages` | **Full response text** — JSON array with assistant replies, tool call requests |
| `gen_ai.operation.name` | "chat" |
| `gen_ai.request.model` | Requested model |
| `gen_ai.response.model` | Actual model that responded |
| `gen_ai.request.max_tokens` | Max tokens setting |
| `gen_ai.request.top_p` | Top-p sampling parameter |
| `gen_ai.response.finish_reasons` | Finish reason (e.g., `["stop"]`) |
| `gen_ai.response.id` | Response UUID |
| `gen_ai.usage.input_tokens` | Input token count |
| `gen_ai.usage.output_tokens` | Output token count |
| `gen_ai.usage.cache_read.input_tokens` | Cached input tokens (prompt caching) |
| `gen_ai.provider.name` | "github" |
| `gen_ai.agent.name` | Agent name |
| `gen_ai.conversation.id` | Conversation ID |
| `copilot_chat.chat_session_id` | Chat session UUID |
| `copilot_chat.session_id` | Session ID |
| `copilot_chat.server_request_id` | Server request ID |
| `copilot_chat.request.max_prompt_tokens` | Prompt token limit |
| `copilot_chat.time_to_first_token` | TTFT for this specific call |

**Dashboards:** [Prompt & Response Explorer] (TraceQL content search)

### Span: `invoke_agent <name>` (e.g., `invoke_agent GitHub Copilot Chat`)

Top-level agent invocation — wraps the entire multi-turn session.

| Attribute | Description |
|-----------|-------------|
| `gen_ai.input.messages` | **User's original prompt** |
| `gen_ai.output.messages` | **Final response to user** |
| `gen_ai.tool.definitions` | JSON array of all available tool schemas |
| `gen_ai.usage.input_tokens` | Total input tokens across all turns |
| `gen_ai.usage.output_tokens` | Total output tokens across all turns |
| `copilot_chat.turn_count` | Total turns in this invocation |
| `gen_ai.agent.name` | Agent name |
| `gen_ai.conversation.id` | Conversation ID |
| `gen_ai.request.model` / `gen_ai.response.model` | Model info |

**Dashboards:** [Prompt & Response Explorer] (TraceQL content search)

### Span: `execute_tool <name>` (e.g., `execute_tool grep`)

Individual tool execution.

| Attribute | Description |
|-----------|-------------|
| `gen_ai.tool.name` | Tool name |
| `gen_ai.tool.call.arguments` | **Full tool call arguments** (JSON) |
| `gen_ai.tool.call.result` | **Full tool call result** (JSON) |
| `gen_ai.tool.call.id` | Tool call UUID |
| `gen_ai.tool.description` | Tool description text |
| `gen_ai.tool.type` | Tool type |
| `gen_ai.operation.name` | Operation name |
| `copilot_chat.chat_session_id` | Session correlation |

**Dashboards:** Visible in trace waterfall via any dashboard with trace links

### Span Event: `user_message`

Attached to `invoke_agent` spans — captures the user's prompt.

| Attribute | Description |
|-----------|-------------|
| `content` | **User's prompt text** |
| `copilot_chat.chat_session_id` | Session correlation |

**Dashboards:** Visible in trace waterfall

---

## 6. Dashboard Catalog (10 dashboards)

All dashboards are at [http://localhost:3010](http://localhost:3010) and auto-provisioned from `grafana/dashboards/`.

### Copilot-Specific Dashboards (5)

#### [GitHub Copilot - Usage Overview](http://localhost:3010/d/copilot-usage)
**UID:** `copilot-usage` · **Panels:** 20 · **Filter:** `$service`

| Section | Panels | Data Sources |
|---------|--------|-------------|
| KPI Stats | Total Events, Events/min, Error Events, Avg Response Latency | Prometheus (span-metrics) |
| Event Analysis | Event Rate by Operation, Error Rate Over Time | Prometheus (span-metrics) |
| Distribution | Events by Operation, Events by Status, Events by Service | Prometheus (span-metrics) |
| Latency Analysis | Response Latency Percentiles (P50/P95/P99), Latency by Operation | Prometheus (span-metrics) |
| Log Volume | Log Volume by Event Type, Log Volume by Service | Loki |
| Exploration | Recent Traces (Tempo), Copilot Logs (Loki) | Tempo, Loki |

---

#### [GitHub Copilot - Smart Insights](http://localhost:3010/d/copilot-insights)
**UID:** `copilot-insights` · **Panels:** 35 · **Filter:** `$service`

| Section | Panels | Data Sources |
|---------|--------|-------------|
| AI Model & Token Analytics | Total Input/Output Tokens, Inference Calls, Agent Turns, Token Usage Over Time, Calls by Model, Events by Type, Token Usage by Model | Loki |
| Tool Call Analytics | Total/Failed Tool Calls, Avg Tool Duration, Avg Calls/Turn, Calls by Tool Name, Call Rate Over Time | Loki |
| Performance Deep-Dive | Latency Percentiles by Operation, Latency Heatmap, Throughput vs Latency, Span Rate by Kind | Prometheus (span-metrics) |
| Error Intelligence | Error Rate %, Failed Tool Calls, Error Spans, Error Rate by Operation, Failed Tool Call Logs | Prometheus + Loki |
| Session Analysis | Inference Logs, Agent Turn Logs, Copilot Traces, Full Log Stream | Loki, Tempo |

---

#### [GitHub Copilot - LogQL Insights](http://localhost:3010/d/copilot-logql-insights)
**UID:** `copilot-logql-insights` · **Panels:** 57 · **Filter:** `$service`

The most comprehensive dashboard — uses **both** Loki and Prometheus for every section.

| Section | Panels | Data Sources |
|---------|--------|-------------|
| Volume & Activity | Log Events, Inference Events, Agent Turns, Tool Calls, Sessions, Span Events, Event Rate by Type, Span Rate by Operation | Loki + Prometheus |
| AI Model & Token Analytics | Input/Output Tokens (Loki + Prometheus), Token Usage Over Time, Calls by Model, Token by Model, Token Rate by Model Over Time | Loki + Prometheus |
| Performance & Latency | P50/P95/P99 Latency Stats, Avg TTFT, Latency Trends, TTFT by Model, Avg Latency by Operation, Throughput vs Latency, Inference Duration by Model, Latency Heatmap | Prometheus |
| Tool Call Analytics | Total/Failed Calls (Prom+Loki), Avg Duration, Calls/Turn, Calls by Tool (Prom), Call Rate (Loki), Duration by Tool (Prom+Loki) | Prometheus + Loki |
| Top-N Leaderboards | Top Tools by Count, Top Operations by Count, Top Models by Tokens, Top Operations by Latency | Prometheus |
| Error Intelligence | Error Rate %, Error Spans, Failed Tools (Prom+Loki), Error Rate Over Time, Error Rate by Operation, Failed Tool Logs | Prometheus + Loki |
| Log & Trace Exploration | Inference Logs, Agent Turn Logs, Tool Call Logs, Recent Traces | Loki + Tempo |

---

#### [GitHub Copilot - Prompt & Response Explorer](http://localhost:3010/d/copilot-prompt-response)
**UID:** `copilot-prompt-response` · **Panels:** 25 · **Filter:** `$service`

| Section | Panels | Data Sources |
|---------|--------|-------------|
| Conversation Overview | Total Sessions, Agent Turns, Inferences, Unique Models | Loki + Prometheus |
| Session Timeline & Depth | Session Start Rate, Turns per Interval, Turn Depth by Session, Tokens per Turn | Loki |
| Inference Request Parameters | Temperature Distribution, Finish Reasons, Max Tokens Requested, Request Model Distribution, Tokens per Inference by Model | Loki |
| **Prompt & Response Content** | **Traces with Prompt/Response Content**, Traces with Input Messages, Traces with Output Messages | **Tempo (TraceQL)** |
| Inference & Session Explorer | Inference Detail Logs, Session Start Logs, Recent Traces | Loki + Tempo |

> **To view full prompt/response text:** Click any Trace ID → open the `chat <model>` span → expand the `gen_ai.input.messages` and `gen_ai.output.messages` attributes.

---

#### [GitHub Copilot - Advanced Metrics](http://localhost:3010/d/copilot-advanced-metrics)
**UID:** `copilot-advanced-metrics` · **Panels:** 39 · **Filter:** `$service`

| Section | Panels | Data Sources |
|---------|--------|-------------|
| Token Efficiency | Input/Output Ratio, Avg Tokens/Turn (Input+Output), Total Tokens, Efficiency Ratio Over Time, Cumulative Token Usage | Prometheus + Loki |
| Session Analytics | Session Count, Avg/P95 Turns per Session, Max Turn Depth, Turn Count Distribution Histogram, Session Depth Over Time, Tokens per Session, Tool Calls per Session | Prometheus + Loki |
| Model Comparison | Avg Operation Duration by Model, Avg TTFT by Model, Total Token Usage by Model, Inference Count by Model, Model Output Throughput (tokens/sec) | Prometheus |
| Agentic Workflow | Agent Invocation Rate, Tool Calls per Turn Over Time, Tool Usage Frequency, Tool Duration by Tool | Prometheus + Loki |
| Extended Error Intelligence | Tool Success Rate %, Failed Tool Calls, Avg Failed Duration, Error Type Breakdown, Per-Tool Success Rate %, Failed vs Successful Duration | Prometheus + Loki |
| Span Size & Payload | Span Size Rate by Operation, Payload Size vs Latency, Top Operations by Total Span Size | Prometheus (span-metrics) |

---

### Infrastructure Dashboards (5)

#### [Application Overview](http://localhost:3010/d/app-overview)
**UID:** `app-overview` · **Panels:** 14

General-purpose OTel overview — spans/metrics/logs received, exporter rates, recent traces and logs.

#### [OpenTelemetry Collector](http://localhost:3010/d/otel-collector)
**UID:** `otel-collector` · **Panels:** 6

Collector health — received/exported spans & metrics per second, memory usage, dropped spans.

#### [Traces Explorer](http://localhost:3010/d/traces-explorer)
**UID:** `traces-explorer` · **Panels:** 9

Trace throughput, span send duration P95, batch sizes, trace search, and service graph (node graph).

#### [Metrics Explorer](http://localhost:3010/d/metrics-explorer)
**UID:** `metrics-explorer` · **Panels:** 13

OTel metric pipeline health + application HTTP metrics (request rate, duration percentiles, error rate, status codes) + span-metrics.

#### [Logs Explorer](http://localhost:3010/d/logs-explorer)
**UID:** `logs-explorer` · **Panels:** 8

Log volume by severity and service, live log stream, error-only logs.

---

## 7. Derived Intelligence & Computed Metrics

These metrics don't exist as raw signals — they are **computed from combinations** of raw data in the dashboards.

### Token Efficiency Metrics

| Derived Metric | Formula | Dashboard |
|---------------|---------|-----------|
| **Input/Output Token Ratio** | `sum(input_tokens) / sum(output_tokens)` | [Advanced Metrics] |
| **Avg Tokens per Turn** | `avg_over_time(agent.turn.input_tokens)` | [Advanced Metrics], [Prompt Explorer] |
| **Token Efficiency Over Time** | Input/output rate ratio as time series | [Advanced Metrics] |
| **Cumulative Token Usage** | `increase()` over time range | [Advanced Metrics] |
| **Model Output Throughput** | `rate(output_tokens)` per model — tokens/sec | [Advanced Metrics] |

### Error Intelligence

| Derived Metric | Formula | Dashboard |
|---------------|---------|-----------|
| **Error Rate %** | `error_spans / total_spans * 100` | [Copilot Insights], [LogQL Insights] |
| **Tool Success Rate %** | `successful_calls / total_calls * 100` per tool | [Advanced Metrics] |
| **Failed vs Successful Duration** | Avg duration comparison by success status | [Advanced Metrics] |

### Performance Correlations

| Derived Metric | Formula | Dashboard |
|---------------|---------|-----------|
| **Throughput vs Latency** | Dual-axis: event rate + avg latency overlaid | [Copilot Insights], [LogQL Insights] |
| **Payload Size vs Latency** | Span size rate vs avg latency | [Advanced Metrics] |
| **Latency Heatmap** | Histogram bucket distribution over time | [Copilot Insights], [LogQL Insights] |

### Session-Level Aggregations

| Derived Metric | Formula | Dashboard |
|---------------|---------|-----------|
| **Turns per Session** | `github_copilot_agent_turn_count` histogram | [Advanced Metrics] |
| **P95 Turns per Session** | `histogram_quantile(0.95, turn_count)` | [Advanced Metrics] |
| **Tokens per Session** | `sum by (session_id) (input + output tokens)` | [Advanced Metrics] |
| **Tool Calls per Session** | `count by (session_id) (tool.call events)` | [Advanced Metrics] |
| **Session Depth Over Time** | Max `turn_index` per session | [Advanced Metrics], [Prompt Explorer] |

### Top-N Leaderboards

| Derived Metric | Formula | Dashboard |
|---------------|---------|-----------|
| **Top Tools by Count** | `topk(10, tool_call_count)` | [LogQL Insights] |
| **Top Operations by Count** | `topk(10, spanmetrics_calls)` | [LogQL Insights] |
| **Top Models by Tokens** | `topk(10, token_usage)` | [LogQL Insights] |
| **Top Operations by Latency** | `topk(10, avg_latency)` | [LogQL Insights] |
| **Top Operations by Span Size** | `topk(15, span_size)` | [Advanced Metrics] |

---

## 8. Data-to-Dashboard Cross-Reference

Quick lookup: "I have this data, where can I see it?"

| Data Point | Copilot Usage | Copilot Insights | LogQL Insights | Prompt Explorer | Advanced Metrics | Infra Dashboards |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|
| Event rate by operation | ✅ | | ✅ | | | |
| Error rate over time | ✅ | ✅ | ✅ | | | |
| Latency percentiles (P50/P95/P99) | ✅ | ✅ | ✅ | | | |
| Latency heatmap | | ✅ | ✅ | | | |
| Token usage (input/output) | | ✅ | ✅ | ✅ | ✅ | |
| Token usage by model | | ✅ | ✅ | ✅ | ✅ | |
| Token efficiency ratio | | | | | ✅ | |
| Inference calls by model | | ✅ | ✅ | ✅ | ✅ | |
| Tool call distribution | | ✅ | ✅ | | ✅ | |
| Tool call duration | | ✅ | ✅ | | ✅ | |
| Tool success rate % | | | | | ✅ | |
| Failed tool logs | | ✅ | ✅ | | | |
| TTFT by model | | | ✅ | | ✅ | |
| Session count | | | ✅ | ✅ | ✅ | |
| Turns per session | | | | ✅ | ✅ | |
| Turn count distribution | | | | | ✅ | |
| Session depth (turn index) | | | | ✅ | ✅ | |
| Tokens per session | | | | | ✅ | |
| Request temperature | | | | ✅ | | |
| Finish reasons | | | | ✅ | | |
| Max tokens requested | | | | ✅ | | |
| **Prompt/Response text** | | | | **✅** | | |
| **Tool call arguments/results** | | | | **✅** *(trace)* | | |
| Span size by operation | | | | | ✅ | |
| Payload vs latency | | | | | ✅ | |
| Error type breakdown | | | | | ✅ | |
| Model comparison (side-by-side) | | | | | ✅ | |
| Throughput vs latency | | ✅ | ✅ | | | |
| Log volume by event type | ✅ | | ✅ | ✅ | | Logs Explorer |
| Collector health (recv/export) | | | | | | App Overview, OTel Collector |
| Service graph | | | | | | Traces Explorer |
| HTTP request metrics | | | | | | Metrics Explorer |

---

## 9. Enabling Content Capture

To capture full prompt/response text in traces, enable this VS Code setting:

```json
{
  "github.copilot.chat.otel.captureContent": true
}
```

This populates:
- `gen_ai.input.messages` — full prompt JSON on `chat` and `invoke_agent` spans
- `gen_ai.output.messages` — full response JSON on `chat` and `invoke_agent` spans
- `gen_ai.tool.call.arguments` — tool call input parameters on `execute_tool` spans
- `gen_ai.tool.call.result` — tool call output on `execute_tool` spans
- `user_message` event with `content` attribute on `invoke_agent` spans

**To view content:**
1. Open [Prompt & Response Explorer](http://localhost:3010/d/copilot-prompt-response)
2. Scroll to "Prompt & Response Content" section
3. Click any Trace ID in the table
4. In the trace waterfall, click a `chat <model>` span
5. Expand `gen_ai.input.messages` (prompt) and `gen_ai.output.messages` (response)

**TraceQL queries for content search:**
```
{ span.gen_ai.input.messages != "" }                    # All traces with prompts
{ span.gen_ai.output.messages != "" }                   # All traces with responses
{ span.gen_ai.input.messages != "" && name =~ "chat.*" }  # Chat spans with content
```

---

*Generated from live data on 2026-03-28. Grafana at http://localhost:3010*
