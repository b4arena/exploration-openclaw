# 15 — Langfuse Cloud as Audit Trail for Agent Communication

## Question

Can we use Langfuse Cloud (hosted SaaS) to capture, visualize, and audit agent-to-agent communication in OpenClaw — including `sessions_send`, `sessions_spawn`, token usage, costs, and full conversation traces with provenance tracking?

## TL;DR

**Yes, partially.** OpenClaw's `diagnostics-otel` plugin exports traces, metrics, and logs via OTLP HTTP/protobuf. Langfuse Cloud accepts OTLP directly — no OTel Collector needed for basic setup. You get token/cost dashboards, nested span visualization, and session grouping out of the box. However, **OpenClaw does not currently emit dedicated spans for `sessions_send` or `sessions_spawn`**, so A2A communication is only visible indirectly through `model.usage` and `message.processed` spans that carry `openclaw.sessionKey` attributes. A custom OTel Collector with attribute processors can enrich these spans for better filtering, but true A2A provenance tracking (`inter_session` vs `user_input`) requires upstream changes to the `diagnostics-otel` plugin.

---

## 1. Langfuse Cloud Setup

### Sign-Up and Project Creation

1. Go to [https://cloud.langfuse.com](https://cloud.langfuse.com) (EU region) or [https://us.cloud.langfuse.com](https://us.cloud.langfuse.com) (US region).
2. Create an account (GitHub, Google, or email).
3. Create a project — this is the container for all traces, observations, and scores.
4. Generate API keys: go to **Settings > API Keys** and create a key pair (`pk-lf-...` / `sk-lf-...`).

Source: [Langfuse OTel Integration](https://langfuse.com/integrations/native/opentelemetry)

### API Keys and Authentication

Langfuse uses HTTP Basic Auth for the OTLP endpoint. The auth string is a Base64-encoded `publicKey:secretKey`:

```bash
# Generate the auth string
AUTH_STRING=$(echo -n "pk-lf-1234567890:sk-lf-1234567890" | base64 -w 0)
# On macOS: echo -n "pk-lf-...:sk-lf-..." | base64
```

This goes into the `Authorization: Basic <AUTH_STRING>` header.

### Pricing Tiers

| Plan | Price | Included Units | Ingestion Rate | Data Retention | Notes |
|------|-------|----------------|----------------|----------------|-------|
| **Hobby** | Free | 50,000 units/month | 1,000 req/min | Configurable (min 3 days) | No credit card needed |
| **Core** | — | — | 4,000 req/min | Configurable (min 3 days) | — |
| **Pro** | ~$59/month | 100,000 units/month | 20,000 req/min | Configurable (min 3 days) | Unlimited users |
| **Team** | ~$499+/month | Unlimited ingestion | 20,000 req/min | Configurable (min 3 days) | All features, SOC 2 |
| **Enterprise** | Custom | Custom | Custom | Custom | HIPAA, SSO, custom DPA |

**Billable unit** = 1 trace + 1 observation + 1 score = 3 units. Formula: `Units = Count(Traces) + Count(Observations) + Count(Scores)`.

Source: [Langfuse Pricing](https://langfuse.com/pricing), [Langfuse Billable Units](https://langfuse.com/docs/administration/billable-units), [Langfuse API Limits](https://langfuse.com/faq/all/api-limits)

### Data Retention

- Default: data stored **indefinitely** (on paid plans).
- Configurable per project with a minimum of **3 days**.
- Deletion runs nightly; deleted data is **not recoverable**.
- Retention applies independently to traces, observations, scores, and media assets.

Source: [Langfuse Data Retention](https://langfuse.com/docs/administration/data-retention)

---

## 2. OpenTelemetry Integration

### Direct OTLP Export (No Collector Needed)

Langfuse Cloud accepts OTLP HTTP/protobuf directly. The `diagnostics-otel` plugin already uses `@opentelemetry/exporter-trace-otlp-http` which is exactly what Langfuse expects.

**OTLP Endpoints:**

| Region | Endpoint |
|--------|----------|
| EU (Ireland) | `https://cloud.langfuse.com/api/public/otel` |
| US (Oregon) | `https://us.cloud.langfuse.com/api/public/otel` |

The plugin auto-appends `/v1/traces`, `/v1/metrics`, `/v1/logs` — and Langfuse's endpoint handles the path resolution (`extensions/diagnostics-otel/src/service.ts:22-29`).

**Protocol:** Only `http/protobuf` is supported. No gRPC. This matches OpenClaw's default protocol (`extensions/diagnostics-otel/src/service.ts:57`).

### Minimal OpenClaw Configuration

```json5
// ~/.openclaw/openclaw.json
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: { "diagnostics-otel": { enabled: true } },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "https://cloud.langfuse.com/api/public/otel",
      protocol: "http/protobuf",
      headers: {
        Authorization: "Basic ${LANGFUSE_AUTH_BASE64}",
      },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,   // Langfuse ignores metrics — they go to /v1/metrics which Langfuse does not process
      logs: false,      // Langfuse ignores logs via OTLP — use traces only
      sampleRate: 1.0,  // 1.0 = capture everything for audit trail
      flushIntervalMs: 30000,
    },
  },
}
```

Set the auth string in `~/.openclaw/.env`:

```bash
# Generate once, paste into .env
LANGFUSE_AUTH_BASE64=$(echo -n "pk-lf-YOUR_PUBLIC_KEY:sk-lf-YOUR_SECRET_KEY" | base64 -w 0)
```

Source: `extensions/diagnostics-otel/src/service.ts:63-64` (endpoint resolution), `src/config/types.base.ts:138-151` (config type), `docs/logging.md:222-264` (OTel config docs)

### With OTel Collector as Middleware (Recommended for Production)

For production use, an OTel Collector provides benefits:

1. **Fan-out**: Send to both Langfuse and another backend (Jaeger, Grafana Tempo) simultaneously.
2. **Attribute enrichment**: Add `langfuse.trace.session.id` and `langfuse.trace.tags` attributes based on OpenClaw's `openclaw.sessionKey`.
3. **Filtering**: Only send LLM-related spans to Langfuse; send infra spans elsewhere.
4. **Buffering**: Protect against network blips.

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: "0.0.0.0:4318"

processors:
  # Transform OpenClaw attributes to Langfuse-recognized attributes
  transform/langfuse:
    trace_statements:
      - context: span
        statements:
          # Map OpenClaw session key to Langfuse session ID
          - set(attributes["langfuse.trace.session.id"], attributes["openclaw.sessionKey"])
            where attributes["openclaw.sessionKey"] != ""
          # Tag A2A-related spans for filtering
          - set(attributes["langfuse.trace.tags"], ["a2a", "openclaw"])
            where attributes["openclaw.sessionKey"] != ""
          # Map model to gen_ai namespace for Langfuse generation detection
          - set(attributes["gen_ai.request.model"], attributes["openclaw.model"])
            where attributes["openclaw.model"] != ""

  batch:
    timeout: 10s
    send_batch_size: 512

exporters:
  otlphttp/langfuse:
    endpoint: "https://cloud.langfuse.com/api/public/otel"
    headers:
      Authorization: "Basic YOUR_AUTH_STRING"
  # Optional: parallel export to Grafana Tempo
  otlphttp/tempo:
    endpoint: "http://tempo:4318"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [transform/langfuse, batch]
      exporters: [otlphttp/langfuse, otlphttp/tempo]
```

Then point OpenClaw at the collector:

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      // no headers needed — collector handles auth
    },
  },
}
```

Source: [Langfuse OTel Integration](https://langfuse.com/integrations/native/opentelemetry), [Langfuse Existing OTel Setup FAQ](https://langfuse.com/faq/all/existing-otel-setup)

### GenAI Semantic Conventions Support

Langfuse maps `gen_ai.*` attributes to its data model:

| OTel Attribute | Langfuse Field | Notes |
|----------------|----------------|-------|
| `gen_ai.request.model` | `model` | Used to identify generations |
| `gen_ai.usage.input_tokens` | `usage.input` | Token counts |
| `gen_ai.usage.output_tokens` | `usage.output` | Token counts |
| `gen_ai.system` | System identifier | e.g., "openai", "anthropic" |

**The problem:** OpenClaw's `diagnostics-otel` plugin uses `openclaw.*` namespaced attributes (e.g., `openclaw.model`, `openclaw.tokens.input`), NOT the `gen_ai.*` convention. This means Langfuse will **not** automatically recognize spans as "generations" — they appear as generic spans. The OTel Collector transform processor shown above bridges this gap.

Source: `extensions/diagnostics-otel/src/service.ts:340-401` (attribute naming), [Langfuse GenAI conventions](https://langfuse.com/integrations/native/opentelemetry)

---

## 3. What You Can See in Langfuse

### Trace Visualization

Langfuse displays spans in a nested tree view. For OpenClaw, this means:

- **`openclaw.model.usage`** spans show each LLM call with token counts and cost.
- **`openclaw.webhook.processed`** spans show inbound message processing.
- **`openclaw.message.processed`** spans show message handling with session keys.
- **`openclaw.session.stuck`** spans highlight problematic sessions.

Each span shows its attributes, duration, and status (OK/ERROR).

### Token and Cost Dashboards

Langfuse provides built-in dashboards for:

- Token usage over time (input, output, cache_read, cache_write)
- Cost per model and provider
- Latency distributions (agent run duration)
- Context window utilization

Custom dashboards can be created to slice data by `openclaw.channel`, `openclaw.provider`, `openclaw.model`.

Source: [Langfuse Custom Dashboards](https://langfuse.com/docs/metrics/features/custom-dashboards)

### Session Grouping

Langfuse sessions aggregate traces by `sessionId`. If the OTel Collector maps `openclaw.sessionKey` to `langfuse.trace.session.id`, you get:

- All LLM calls for a given agent session grouped together.
- A "session replay" showing the sequence of model calls within a conversation.
- Subagent sessions (key pattern `agent:<agentId>:subagent:<uuid>`) visible as separate sessions.

Source: [Langfuse Sessions](https://langfuse.com/docs/observability/features/sessions)

### Filtering by Provenance

**Not directly possible today.** The `diagnostics-otel` plugin does not emit `provenance.kind` (`inter_session` vs `user_input`) as a span attribute. You can only infer A2A activity by:

1. **Session key pattern matching**: Subagent sessions have keys matching `agent:*:subagent:*`.
2. **Tag-based filtering**: If using the OTel Collector to add tags based on session key patterns.
3. **Metadata search**: Searching for `openclaw.sessionKey` values that indicate cross-agent communication.

### Search and Export

- **Search**: Full-text search on trace content, filter by tags, metadata, user, session, time range, environment.
- **Export**: Export traces, sessions, and generations from the UI. Programmatic export via the SDK and REST API with pagination.
- **Data query SDK**: Python and JS SDKs support fetching traces, observations, and scores with filters.

Source: [Langfuse Query Data via SDK](https://langfuse.com/docs/api-and-data-platform/features/query-via-sdk)

---

## 4. Alternative: Direct Langfuse SDK Integration

### Is There a Langfuse Plugin for OpenClaw?

**No.** There is no `@openclaw/langfuse` plugin. No references to Langfuse exist anywhere in the OpenClaw codebase (confirmed via search).

### SDK vs OTel Approach

| Dimension | OTel (via `diagnostics-otel`) | Langfuse SDK (hypothetical plugin) |
|-----------|-------------------------------|------------------------------------|
| **Setup effort** | Low — config-only | High — new plugin needed |
| **Attribute mapping** | Manual (OTel Collector transforms) | Native Langfuse data model |
| **Session grouping** | Requires attribute mapping | Native `sessionId` propagation |
| **Generation detection** | Requires `gen_ai.*` mapping | Native via SDK calls |
| **Cost tracking** | Manual attribute mapping | Built-in model cost lookup |
| **Prompt management** | Not supported | Full prompt versioning |
| **Scores/evaluation** | Not supported | Native scoring API |
| **Vendor lock-in** | None (standard OTel) | Langfuse-specific |
| **Maintenance** | Upstream-maintained | Custom plugin to maintain |

**Recommendation:** Start with the OTel approach. It requires no code changes. If you need richer Langfuse features (prompt management, scoring, evaluations), build a custom plugin later using the `@langfuse/langfuse` JS SDK and OpenClaw's Plugin SDK (`openclaw/plugin-sdk`).

### Building a Custom Plugin (Sketch)

A Langfuse plugin would follow the same pattern as `diagnostics-otel` — subscribe to diagnostic events via `onDiagnosticEvent()` and forward them to Langfuse:

```typescript
import { onDiagnosticEvent } from "openclaw/plugin-sdk";
import { Langfuse } from "langfuse";

const langfuse = new Langfuse({
  publicKey: process.env.LANGFUSE_PUBLIC_KEY,
  secretKey: process.env.LANGFUSE_SECRET_KEY,
  baseUrl: "https://cloud.langfuse.com",
});

onDiagnosticEvent((evt) => {
  if (evt.type === "model.usage") {
    const trace = langfuse.trace({
      name: "model.usage",
      sessionId: evt.sessionKey,
      metadata: { channel: evt.channel, provider: evt.provider },
    });
    trace.generation({
      name: evt.model,
      model: evt.model,
      usage: { input: evt.usage.input, output: evt.usage.output },
      // ... token details
    });
  }
});
```

This is illustrative only — a production plugin would need proper lifecycle management, batching, and error handling.

---

## 5. Practical Audit Trail Patterns

### Tagging A2A Traces for Filtering

Use the OTel Collector to tag spans based on session key patterns:

```yaml
processors:
  transform/a2a-tags:
    trace_statements:
      - context: span
        statements:
          # Tag subagent spawns
          - set(attributes["langfuse.trace.tags"], ["subagent", "a2a"])
            where IsMatch(attributes["openclaw.sessionKey"], "agent:.*:subagent:.*")
          # Tag A2A sends (requires knowing the session key pattern)
          - set(attributes["langfuse.trace.tags"], ["a2a-send"])
            where attributes["openclaw.source"] == "a2a"
```

In Langfuse, filter by tag `a2a` or `subagent` to see only inter-agent communication.

### Alerting for Anomalies

Langfuse Cloud alerting is limited today:

- **Usage alerts**: Set a threshold for monthly events; get notified once per billing cycle when crossed.
- **Webhooks**: Available for prompt changes, not for trace-level anomalies.
- **Custom alerting**: Use the Langfuse Metrics API to build external alerting (e.g., with Grafana or PagerDuty).

```bash
# Example: Poll Langfuse API for stuck sessions
curl -s "https://cloud.langfuse.com/api/public/traces?tags=a2a&minTimestamp=$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" \
  -H "Authorization: Basic $AUTH_STRING" | jq '.data | length'
```

Source: [Langfuse Webhooks](https://langfuse.com/docs/prompt-management/features/webhooks-slack-integrations), [GitHub Discussion #3997](https://github.com/orgs/langfuse/discussions/3997)

### Exporting Data for Compliance

- **UI Export**: CSV/JSON export of traces, sessions, and generations directly from the Langfuse web UI.
- **SDK Export**: Programmatic export via Python/JS SDK with pagination:

```python
from langfuse import Langfuse
langfuse = Langfuse()

# Export all A2A traces from the last 7 days
traces = langfuse.fetch_traces(
    tags=["a2a"],
    from_timestamp=datetime.now() - timedelta(days=7),
)
for page in traces:
    for trace in page.data:
        # Write to compliance archive
        archive.write(trace.dict())
```

Source: [Langfuse Query via SDK](https://langfuse.com/docs/api-and-data-platform/features/query-via-sdk)

### Data Sovereignty

| Aspect | Detail |
|--------|--------|
| **EU Region** | AWS `eu-west-1` (Ireland), data stays within EEA |
| **US Region** | AWS `us-west-2` (Oregon) |
| **Encryption** | TLS in transit, AES-256 at rest |
| **Certifications** | SOC 2 Type II, ISO 27001 |
| **GDPR** | Compliant, DPA available |
| **HIPAA** | Available on dedicated region (Enterprise plan) |

For EU data sovereignty, use the EU endpoint (`cloud.langfuse.com`). All processing stays within the EEA.

Source: [Langfuse Security](https://langfuse.com/security), [Langfuse Data Regions](https://langfuse.com/security/data-regions), [Langfuse GDPR](https://langfuse.com/security/gdpr)

---

## 6. Limitations

### What Langfuse Cannot Show About OpenClaw A2A

1. **No dedicated A2A spans**: The `diagnostics-otel` plugin does not emit spans for `sessions_send` or `sessions_spawn` tool invocations. These tools are invoked within the LLM tool-use loop, but the OTel plugin only captures `model.usage`, `message.processed`, `webhook.*`, and `session.*` events (`extensions/diagnostics-otel/src/service.ts:573-611`).

2. **No provenance tracking**: Message provenance (`inter_session` vs `user_input`) is stored in session transcripts (`docs/concepts/session-tool.md`) but is **not** exported to OTel spans. You cannot filter in Langfuse by "show me only messages that came from another agent."

3. **No ping-pong turn visibility**: The A2A ping-pong loop (`src/agents/tools/sessions-send-tool.a2a.ts:18-149`) runs multiple model calls, but each appears as an independent `model.usage` span. The causal chain (requester -> target -> requester -> ...) is not captured as parent-child spans.

4. **No agent identity on spans**: Spans carry `openclaw.sessionKey` and `openclaw.sessionId` but not `agentId` directly. You must parse the session key (format: `agent:<agentId>:...`) to extract the agent identity.

### Rate Limits and Data Size Limits

| Constraint | Hobby | Pro/Team |
|------------|-------|----------|
| Ingestion rate | 1,000 req/min | 20,000 req/min |
| Max payload | 5 MB per request | 5 MB per request |
| Monthly units (Hobby) | 50,000 | Varies by plan |

With `sampleRate: 1.0` and active A2A communication, a busy OpenClaw gateway could exhaust the Hobby tier quickly. Each model call generates at minimum 1 trace + 1 observation = 2 units.

Source: [Langfuse API Limits](https://langfuse.com/faq/all/api-limits)

### Gaps Between OTel Export and Langfuse Ingestion

1. **Metrics not ingested**: Langfuse's OTLP endpoint processes **traces only**. OpenClaw's OTel metrics (`openclaw.tokens`, `openclaw.cost.usd`, histograms) are sent to `/v1/metrics` which Langfuse ignores. Token/cost data must come from span attributes.

2. **Logs not ingested**: Langfuse does not process OTLP logs. Setting `logs: true` in OpenClaw sends log records nowhere useful when pointing directly at Langfuse.

3. **Custom attribute namespace**: OpenClaw uses `openclaw.*` attributes, not `gen_ai.*`. Without an OTel Collector transform, Langfuse treats all spans as generic spans rather than LLM generations. This means the "Generations" tab in Langfuse will be empty.

4. **No parent-child span linking**: The `diagnostics-otel` plugin creates each span independently (no parent context). All spans appear as root spans in Langfuse, not as nested trees within a trace. This is a fundamental limitation of the current plugin architecture (`extensions/diagnostics-otel/src/service.ts:325-337` — `spanWithDuration` creates standalone spans).

---

## 7. Recommended Implementation Path

### Phase 1: Quick Win (1 hour)

Point OpenClaw directly at Langfuse Cloud. You get basic span visibility and can verify the pipeline works.

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "https://cloud.langfuse.com/api/public/otel",
      headers: { Authorization: "Basic ${LANGFUSE_AUTH_BASE64}" },
      traces: true,
      metrics: false,  // Langfuse ignores these
      logs: false,
      sampleRate: 1.0,
    },
  },
}
```

### Phase 2: OTel Collector (1 day)

Deploy an OTel Collector to transform `openclaw.*` attributes into `gen_ai.*` and `langfuse.*` namespaces. This unlocks generation detection and session grouping.

### Phase 3: Upstream Contribution (longer term)

Contribute to the `diagnostics-otel` plugin:

1. Add `agentId` as a span attribute on all spans.
2. Emit dedicated spans for `sessions_send` and `sessions_spawn` tool calls.
3. Add `provenance.kind` attribute from message provenance.
4. Implement parent-child span relationships (trace context propagation).
5. Add `gen_ai.*` semantic convention attributes alongside `openclaw.*` attributes.

These changes would make Langfuse (and any OTel backend) show proper nested traces with A2A communication clearly visible.

---

## References

### OpenClaw Source Files

| File | Relevance |
|------|-----------|
| `extensions/diagnostics-otel/src/service.ts` | Full OTel plugin implementation — spans, metrics, attribute naming |
| `extensions/diagnostics-otel/index.ts` | Plugin registration |
| `extensions/diagnostics-otel/package.json` | OTel SDK dependencies (v2.5.1) |
| `src/config/types.base.ts:138-151` | `DiagnosticsOtelConfig` type definition |
| `src/config/zod-schema.ts:138-147` | Zod validation for otel config |
| `src/agents/tools/sessions-send-tool.a2a.ts:18-149` | A2A ping-pong implementation |
| `src/agents/tools/sessions-send-helpers.ts:73-166` | A2A context builders |
| `src/agents/subagent-spawn.ts` | Subagent spawn implementation |

### OpenClaw Documentation

| Doc | Relevance |
|-----|-----------|
| `docs/logging.md:222-324` | OTel config, exported metrics, exported spans |
| `docs/concepts/session-tool.md` | sessions_send, sessions_spawn, provenance |
| `docs/concepts/multi-agent.md` | Multi-agent routing, agent-to-agent config |

### Langfuse Documentation

| Resource | URL |
|----------|-----|
| OTel Integration | [langfuse.com/integrations/native/opentelemetry](https://langfuse.com/integrations/native/opentelemetry) |
| Existing OTel Setup FAQ | [langfuse.com/faq/all/existing-otel-setup](https://langfuse.com/faq/all/existing-otel-setup) |
| Pricing | [langfuse.com/pricing](https://langfuse.com/pricing) |
| Data Retention | [langfuse.com/docs/administration/data-retention](https://langfuse.com/docs/administration/data-retention) |
| Billable Units | [langfuse.com/docs/administration/billable-units](https://langfuse.com/docs/administration/billable-units) |
| API Limits | [langfuse.com/faq/all/api-limits](https://langfuse.com/faq/all/api-limits) |
| Sessions | [langfuse.com/docs/observability/features/sessions](https://langfuse.com/docs/observability/features/sessions) |
| Data Model | [langfuse.com/docs/observability/data-model](https://langfuse.com/docs/observability/data-model) |
| Security & Compliance | [langfuse.com/security](https://langfuse.com/security) |
| Data Regions | [langfuse.com/security/data-regions](https://langfuse.com/security/data-regions) |
| GDPR | [langfuse.com/security/gdpr](https://langfuse.com/security/gdpr) |
| Custom Dashboards | [langfuse.com/docs/metrics/features/custom-dashboards](https://langfuse.com/docs/metrics/features/custom-dashboards) |
| Query via SDK | [langfuse.com/docs/api-and-data-platform/features/query-via-sdk](https://langfuse.com/docs/api-and-data-platform/features/query-via-sdk) |

### Prior Exploration

| Doc | Relevance |
|-----|-----------|
| `exploration/deployment/best-practices.md` | OTel config examples, exported metrics/spans reference |
| `exploration/agents/agent-to-agent.md` | A2A mechanisms, session keys, provenance |
