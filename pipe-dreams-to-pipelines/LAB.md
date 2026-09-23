# Pipe Dreams to Pipelines — Lab Guide

**Workshop · OpenSearchCon North America 2026 · Thu Sep 24 · 10:40–12:20**
Anirudha Jadhav (OpenSearch · AWS) and Wolfgang Theilmann (SAP)

Slides: [anirudha.github.io/talks/pipe-dreams-to-pipelines](https://anirudha.github.io/talks/pipe-dreams-to-pipelines/)
Code: [github.com/anirudha/os-agent-observability-evals](https://github.com/anirudha/os-agent-observability-evals)

This guide is the copy-paste companion to the deck. **Don't type from the slides** — the slides show the shape, this file has the exact text.

---

## Before you start

| | |
|---|---|
| **Docker** | Docker Desktop or Podman, running, **8 GB** allocated to the VM. Six containers, one of them a JVM. |
| **Python** | 3.10+ with `pip`. `uv` works too — both commands are given below. |
| **Ports** | `4317` `4318` `5601` `8080` `9090` `9200` `21890` free. `lsof -i :5601` settles arguments. Set `PORT_PREFIX` in `.env` to shift all of them. |
| **Disk** | ~4 GB for images. |
| **A model** | One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or AWS credentials with Bedrock access in `us-west-2`. |
| **No model key?** | Add `--replay` to every `agent.cli` command. The agent runs against recorded model responses and **every lab still works**. |

Pull the images before the room fills — the venue wifi is the bottleneck, not your laptop:

```bash
git clone https://github.com/anirudha/os-agent-observability-evals
cd os-agent-observability-evals
docker compose pull
```

**No Docker at all?** Open <https://observability.playground.opensearch.org>. It is the same stack with live agent traces already flowing. You can follow Labs 03–05 read-only, and every PPL query in Lab 04 works verbatim.

**Falling behind?** Each lab has a `make lab-NN` that jumps you to its starting state. Losing Lab 02 must not cost you Lab 05.

---

## Lab 00 · Stack up

**10:48 → 10:58 · 10 min**

```bash
cd os-agent-observability-evals
cp .env.example .env      # then add ONE model key — or leave it empty and use --replay
docker compose up -d
```

Six containers come up: the agent, an OTel Collector, Data Prepper, OpenSearch, OpenSearch Dashboards, Prometheus.

### Checkpoint 0

| Open this | You should see |
|---|---|
| `localhost:5601` | Dashboards home, with an **Observability Stack** section in the left nav |
| `localhost:9090/targets` | The `otel-collector` scrape target, state **UP** |
| `localhost:9200/_cat/indices?v` | Green or yellow cluster. No `otel-v1-apm-*` yet — correct, nothing has been traced |
| `localhost:8080/healthz` | `{"ok":true,"model":"…"}` or `{"ok":true,"mode":"replay"}` |
| `docker compose logs data-prepper` | `Started Data Prepper with 3 pipelines`, no stack trace |

**Green sticky when** Dashboards loads, **Observability Stack → Agent Traces** opens (empty is fine), and the Prometheus target is `UP`.

### If it doesn't come up

- **Dashboards stuck on "optimizing bundles"** — give it 90 seconds. It is single-threaded on first boot.
- **OpenSearch exits with code 137** — the Docker VM has less than 8 GB. Raise it, then `docker compose up -d` again.
- **Port already in use** — set `PORT_PREFIX` in `.env` and `docker compose up -d` again. It shifts all six.
- **Data Prepper restarting** — it starts before OpenSearch is healthy on slow machines. `docker compose restart data-prepper`.

---

## Lab 01 · Instrument the agent

**10:58 → 11:12 · 14 min**

The starting point is `agent/core.py`: a support agent with three deterministic tools (`lookup_order`, `check_inventory`, `search_policy`) and no observability at all.

Deterministic tools are deliberate. When the eval in Lab 05 fails, you want to know the *agent* was wrong — not that a third-party API was flaky.

### Step 1 — Install and register

```bash
pip install "opensearch-genai-observability-sdk-py[anthropic]"
# or: uv pip install "opensearch-genai-observability-sdk-py[anthropic]"
# swap the extra for your stack: [bedrock] [openai] [langchain] [llamaindex] [all]
```

In `agent/core.py`, at import time:

```python
from opensearch_genai_observability_sdk_py import register, observe, enrich, Op

register(
    endpoint="http://localhost:4318/v1/traces",
    service_name="acme-support-agent",
    # auto_instrument=True by default — it discovers installed instrumentors
)
```

`register()` configures the tracer provider, the OTLP exporter and auto-instrumentation in one call. In production this line does not change — you set `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` and let the environment decide.

### Step 2 — Decorate the entry point and the tools

```python
# the entry point — one invoke_agent root span per turn
@observe(op=Op.INVOKE_AGENT, name="acme-support-agent")
def handle_support_question(question: str, conversation_id: str = "anonymous") -> str:
    enrich(session_id=conversation_id)   # → gen_ai.conversation.id
    return run_turn(question)            # your existing loop, untouched

# a tool — arguments and return value captured for you
@observe(op=Op.EXECUTE_TOOL, name="lookup_order")
def lookup_order(order_id: str) -> dict:
    return _ORDERS.get(order_id) or {"error": "order_not_found"}
```

Do the same for `check_inventory` and `search_policy`.

| You write | The span carries |
|---|---|
| `name=` on an `INVOKE_AGENT` | `gen_ai.agent.name` |
| `name=` on an `EXECUTE_TOOL` | `gen_ai.tool.name` |
| nothing — the arguments | `gen_ai.tool.call.arguments` |
| nothing — the return value | `gen_ai.tool.call.result` |
| `enrich(session_id=…)` | `gen_ai.conversation.id` |

You did not wrap a single model call, yet every `chat` span with its model name, token counts and stop reason will be on the trace. That is the instrumentor for whichever client library you installed, picked up by `register()`.

### Step 3 — Make it emit

```bash
python -m agent.cli ask "where is order A-1042?"
python -m agent.cli ask "can I return a SKU-77 after 40 days?"
python -m agent.cli ask "where is order A-9999?"    # this order does not exist
python -m agent.cli seed --questions 12             # background traffic for Lab 03
```

**Keep the trace ID from the third command.** It is the specimen for the rest of the workshop: Lab 04 reconstructs what it did, Lab 05 grades it, Lab 06 turns it into a regression test. The tool returned `order_not_found` four times and the model invented a delivery date anyway.

### Checkpoint 1

Open **Observability Stack → Agent Traces**, sort newest first, open your third trace, click the `execute_tool` span.

**Green sticky when**

- Three traces are listed, and the third has visibly more spans than the first.
- The `execute_tool` span shows `order_not_found` in its result — and the root span's output text contradicts it.

### Brought your own agent?

Use it instead. Add `register()` and one `@observe(op=Op.INVOKE_AGENT)` on your entry point, point it at `localhost:4318`, and rejoin at Lab 02. Everything downstream is attribute-driven, so it does not care whose agent it is.

**Node.js:** `@opensearch-project/genai-observability-sdk-ts` gives you `register`, `observe`, `enrich`, `Op`. It is a **function wrapper, not a decorator** — `observe({op}, fn)`, with `await register()` — and its auto-instrumentation coverage is narrower than Python's today. Instrumentation in Node is fine; the scoring side in Lab 05 is Python.

---

## Lab 02 · The pipelines

**11:12 → 11:25 · 13 min** · driving: Wolfgang

The half that demos skip and production punishes. Read these two files — they are the ones you will edit in your own stack.

### Step 1 — The Collector: one receiver, two pipelines

`otel/collector.yaml`

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

exporters:
  otlp/dataprepper:                       # traces → Data Prepper → OpenSearch
    endpoint: data-prepper:21890
    tls: { insecure: true }
  prometheus:                             # metrics → scraped by Prometheus
    endpoint: 0.0.0.0:8889
    resource_to_telemetry_conversion: { enabled: true }

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/dataprepper]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

`resource_to_telemetry_conversion` is the line that matters most and gets the least attention — it turns resource attributes into Prometheus labels, which is what makes `by (service_name)` work at all.

### Step 2 — Data Prepper: one entry, two sinks

`data-prepper/pipelines.yaml`

```yaml
entry-pipeline:
  source:
    otel_trace_source:
      ssl: false                          # :21890
  sink:
    - pipeline: { name: "raw-pipeline" }
    - pipeline: { name: "service-map-pipeline" }    # same spans, twice

raw-pipeline:                             # the spans you query in Lab 04
  source:
    pipeline: { name: "entry-pipeline" }
  processor:
    - otel_traces:
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        index_type: trace-analytics-raw

service-map-pipeline:                     # the topology in Lab 03
  source:
    pipeline: { name: "entry-pipeline" }
  processor:
    - service_map:
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        index_type: trace-analytics-service-map
```

This is the answer to "who drew the service map?" — nobody did. `service_map` derives caller/callee edges from parent-child span relationships and writes them as their own documents. A sub-agent or MCP server you add tomorrow appears the first time it serves a request.

### Step 3 — What landed

```bash
curl -s "localhost:9200/_cat/indices/otel-v1-*?v&h=index,docs.count,store.size"
```

```
index                              docs.count  store.size
otel-v1-apm-span-000001                   347      1.1mb
otel-v1-apm-service-map                     9     48.4kb
```

| Index pattern | One document is | You use it for |
|---|---|---|
| `otel-v1-apm-span-*` | One span, with all its `gen_ai.*` attributes, its `traceId` and its `parentSpanId` | Trace trees, PPL over reasoning, token attribution, span-level evals |
| `otel-v1-apm-service-map` | One caller→callee edge, with hashed IDs and the target resource | The Application Map, and the dependency view that separates errors from faults |

Note the ratio: **347 spans produced 9 edges.** Topology is tiny and slow-changing; spans are large and fast-growing. That asymmetry is why they are separate indices with separate retention — and the first knob you turn when the bill arrives.

### Step 4 — The attribute-to-label bridge

| On the span (OTel) | In Prometheus | Why |
|---|---|---|
| `gen_ai.agent.name` | `gen_ai_agent_name` | Dots are illegal in Prometheus label names |
| `gen_ai.request.model` | `gen_ai_request_model` | Same rule — an OTel convention, not an OpenSearch choice |
| `service.name` (resource) | `service_name` | Only present because `resource_to_telemetry_conversion` is on |
| `gen_ai.conversation.id` | *deliberately not promoted* | Unbounded cardinality. Belongs on a span, never on a time series |

**The trap:** promoting one high-cardinality attribute to a label is how a Prometheus instance falls over. `agent name` and `model` are bounded — tens of values. `conversation id`, `user id`, `order id` and tool arguments are not. Put those on spans, where unbounded cardinality is the storage engine's whole job.

### Step 5 — The metrics side

```bash
curl -s localhost:9090/api/v1/label/__name__/values | jq -r '.data[]' | grep gen_ai
```

```
gen_ai_client_operation_duration_bucket
gen_ai_client_operation_duration_count
gen_ai_client_operation_duration_sum
gen_ai_client_token_usage_bucket
gen_ai_client_token_usage_count
gen_ai_client_token_usage_sum
```

You did not define these. They are OpenTelemetry GenAI metric conventions, emitted by the same instrumentors that produced your spans. No bespoke metric names, which means no dashboard rewrite when you swap frameworks.

### Checkpoint 2

**Green sticky when**

- `_cat/indices/otel-v1-*` shows both index patterns with non-zero counts.
- The `gen_ai_*` families are present in Prometheus.
- You can say out loud which of the three Data Prepper pipelines produced which index.

---

## Lab 03 · Service map and RED

**11:25 → 11:37 · 12 min** · driving: Wolfgang

### Step 1 — Application Map

**Observability Stack → Application Map.** Your agent is a node beside your services. Nobody configured this; the ingestion pipeline derived it from trace data in Lab 02.

Click a node — the side panel gives you its RED numbers and a **View traces** link.

### Step 2 — Services

**Observability Stack → Services.** P99 latency, throughput and failure ratio for every service, agents in the same table with the same columns. This is your triage entry point.

### Step 3 — RED per agent, in native PromQL

Paste into `prometheus/rules.yaml`, or straight into the expression browser at `localhost:9090`:

```yaml
# rate — invocations per second, per agent
- record: acme:agent_requests:rate5m
  expr: sum(rate(gen_ai_client_operation_duration_count[5m])) by (gen_ai_agent_name)

# errors — error ratio, per agent
- record: acme:agent_errors:ratio5m
  expr: |
    sum(rate(gen_ai_client_operation_duration_count{error_type!=""}[5m])) by (gen_ai_agent_name)
      / sum(rate(gen_ai_client_operation_duration_count[5m])) by (gen_ai_agent_name)

# duration — p95 latency, per agent
- record: acme:agent_latency:p95_5m
  expr: |
    histogram_quantile(0.95,
      sum(rate(gen_ai_client_operation_duration_bucket[5m])) by (le, gen_ai_agent_name))

# and the one RED never had — cost, per agent, per model
- record: acme:agent_output_tokens:rate5m
  expr: |
    sum(rate(gen_ai_client_token_usage_sum{gen_ai_token_type="output"}[5m]))
      by (gen_ai_agent_name, gen_ai_request_model)
```

Two more rules make these page: p95 above 4 s for 5 minutes, error ratio above 5% for 10 minutes. Both `severity: page`.

### Step 4 — Walk the four hops

1. **Symptom** — a burn-rate alert fires. The alert name carries the SLO, the tier and the window pair.
2. **Narrow** — Services nominates the service by fault rate; the dependency view separates *errors* (caller's fault) from *faults* (callee's).
3. **Drill** — **View traces** drops you into Agent Traces, pre-filtered to that service and window.
4. **Confirm** — the span detail has a Logs tab: application log lines matched by trace ID. You wrote no query to get here.

### Checkpoint 3

**Green sticky when**

- `acme-support-agent` appears on the Application Map and in the Services catalogue.
- You clicked a node, hit **View traces**, and landed on a filtered trace list.
- At least one `acme:` recording rule returns data in the Prometheus expression browser.

---

## Lab 04 · Reconstruct the reasoning with PPL

**11:37 → 11:52 · 15 min**

Run these in **Observability Stack → Logs**, or on the Agent Traces page. Same editor, same index.

Every query has the same four verbs:

```
source = otel-v1-apm-span-*        // where to read
| where  …                          // narrow
| stats  … by …                     // aggregate
| sort   - …                        // order
```

### Query 1 — Replay one turn's trajectory

Substitute your specimen trace ID from Lab 01:

```sql
source = otel-v1-apm-span-*
| where traceId = 'c30d4f1…'
| fields startTime, name,
         attributes.gen_ai.operation.name,
         attributes.gen_ai.tool.name,
         attributes.gen_ai.usage.output_tokens,
         durationInNanos
| sort startTime
```

You should see the same tool four times at ~12 ms each, with five `chat` spans around them. The tool was never slow. The agent just would not accept the answer. That is a prompt bug, and no latency dashboard would have pointed at it.

### Query 2 — Where the money goes

```sql
source = otel-v1-apm-span-*
| where isnotnull(attributes.gen_ai.request.model)
| stats sum(attributes.gen_ai.usage.input_tokens)  as in_tokens,
        sum(attributes.gen_ai.usage.output_tokens) as out_tokens,
        count()                                    as calls
  by attributes.gen_ai.agent.name, attributes.gen_ai.request.model
| sort - out_tokens
```

`isnotnull` on the model field is doing real work — it selects **only the model-call spans**. Tool and agent spans have no token usage, and summing over them silently divides your averages by ten.

Same question on the metrics side, one line:

```promql
sum(rate(gen_ai_client_token_usage_sum{gen_ai_token_type="output"}[5m])) by (gen_ai_request_model)
```

Use PPL to find out *which trace*; use PromQL to watch the trend.

### Query 3 — Which tool is letting you down

```sql
source = otel-v1-apm-span-*
| where attributes.gen_ai.operation.name = 'execute_tool'
| eval failed = if(status.code = 2, 1, 0)
| stats count()      as calls,
        sum(failed)  as failures,
        avg(durationInNanos) / 1000000 as avg_ms
  by attributes.gen_ai.tool.name
| eval failure_pct = round(failures * 100.0 / calls, 1)
| sort - failure_pct
```

A tool failing 30% of the time is not necessarily broken — it may be getting bad arguments from the model. `gen_ai.tool.call.arguments` is on the same span, so the next query writes itself.

### Query 4 — The loop detector

```sql
source = otel-v1-apm-span-*
| where attributes.gen_ai.operation.name = 'chat'
| stats count() as chat_steps,
        sum(attributes.gen_ai.usage.output_tokens) as out_tokens
  by traceId
| where chat_steps > 3
| sort - chat_steps
```

Your specimen trace should be at the top: five model calls and ~2,800 output tokens to fail at answering one question — roughly four times the cost of the two that succeeded. **Save this query.** It is the cheapest agent alert you will ever write.

### Your turn · 5 minutes

Pick any one, easiest first:

- Which **session** spent the most tokens? (group by `attributes.gen_ai.conversation.id`)
- What is the **P99** of `execute_tool` spans, per tool?
- How many traces called **zero** tools? (a pure-model answer — sometimes right, sometimes a refusal)
- Of the failed tool calls, **what arguments** did the model pass?
- Which model gives the most **output tokens per call** — and is that better or worse?

Three things that will trip you:

- **Field names keep their dots in PPL.** `attributes.gen_ai.usage.output_tokens`, not underscores. The underscore form is the Prometheus side.
- **Duration is `durationInNanos`.** Divide by 1,000,000 in an `eval` so the column header stays readable.
- **Status code 2 means error** in OTel, not 500. `status.code = 2` is your failure predicate.

When a query works, save it. Saved queries become dashboard panels, and dashboard variables are themselves populated by PPL — so one dashboard keeps working for every agent you add later.

### Checkpoint 4

**Green sticky when**

- You ran the loop detector and your worst trace came back at the top.
- You wrote one query of your own that returned rows. Wrong answers count; a syntax error does not.

---

## Lab 05 · Grade the trajectory

**11:52 → 12:08 · 16 min**

Your specimen trace returned 200. It also called the same tool four times, ignored `order_not_found`, invented a delivery date, and cost four times as much as a correct answer. Every SLO you own says that request was fine.

### Step 1 — Define the golden path

`evals/dataset.py`. A golden path is not an expected string — it is the expected **sequence of decisions**, plus a budget.

```python
CASES = [
    {
        "id":               "order-missing",
        "question":         "where is order A-9999?",
        "golden_tools":     ["lookup_order"],           # once. not four times.
        "must_contain":     ["could not find", "check the order number"],
        "must_not_contain": ["shipped", "arrive", "delivery"],   # no invented facts
        "max_chat_steps":   2,
        "max_output_tokens": 600,
    },
    # ...
]
```

Four criteria, four different kinds of wrong: `golden_tools` catches bad tool selection, `must_not_contain` catches confabulation, `max_chat_steps` catches looping, `max_output_tokens` catches the cost regression nobody notices until the invoice.

### Step 2 — Score, in the same trace

```python
from opensearch_genai_observability_sdk_py import observe, score, Op

@observe(op=Op.INVOKE_AGENT, name="acme-support-agent")
def run_case(case: dict) -> str:
    answer = handle_support_question(case["question"])
    trace  = collect_trace()                 # spans from this run

    score(name="tool_path_match",  value=path_match(trace, case["golden_tools"]))
    score(name="no_confabulation", value=absent(answer, case["must_not_contain"]))
    score(name="step_budget",      value=chat_steps(trace) <= case["max_chat_steps"])
    return answer
```

The scores ride the **same OTLP pipeline** you built in Lab 02 and land on the **same spans** you queried in Lab 04. No second datastore, no eval warehouse, no CSV export — "why did this case fail?" and "what did the agent do?" are one click apart, not one team apart.

**Name it correctly, it will save you an argument.** *Agent Evals* is the SDK side: `score()`, `evaluate()`, `Benchmark`. `score(value=…)` is an unbounded float — there is **no pass/fail primitive** here. *Agent Health* is a separate product with its own repo and its own UI on `localhost:4001`; pass/fail and the 0–100% accuracy figure are its. Two products, not a rename.

### Step 3 — Run it

```bash
python -m evals.run --dataset evals/dataset.py
```

```
✔ order-lookup        tool_path_match 1.0   no_confabulation 1.0   step_budget 1.0
✔ return-policy       tool_path_match 1.0   no_confabulation 1.0   step_budget 1.0
✘ order-missing       tool_path_match 0.25  no_confabulation 0.0   step_budget 0.0
✔ inventory-check     tool_path_match 1.0   no_confabulation 1.0   step_budget 1.0

  4 cases · 3 passed · 1 failed · scores emitted as spans · trace 1a77c9b…
```

### Step 4 — Agent Health, for pass/fail and accuracy

```bash
npx @opensearch-project/agent-health@latest    # → localhost:4001
```

Per-case criteria with a judge verdict on each. The UI calls a case a **Use Case** and a run a **Benchmark**, so match whichever surface is on your screen.

The judge runs on AWS Bedrock. No Bedrock access? The deterministic criteria — tool path, step budget, substring checks — still run and still fail your specimen case, which is all Lab 06 needs.

### Checkpoint 5

**Green sticky when**

- `order-missing` fails and the other three pass.
- You can open the eval run in Agent Traces and see the `score` attributes on the spans.
- `tool_path_match` is `0.25` and you can say why — one expected call, four actual.

That `0.25` is the whole workshop in one number. Twenty minutes ago this was a 200 OK.

---

## Lab 06 · Close the loop

**12:08 → 12:16 · 8 min**

A bad trace in production should become a test case without anyone rewriting it by hand.

```bash
python -m evals.promote --trace c30d4f1… --id order-missing-regression
```

```
✔ question       ← root span gen_ai.input.messages
✔ observed_tools ← [lookup_order ×4]
… golden_tools   ← you decide: [lookup_order]
✔ wrote evals/dataset.py  (+1 case, now 5)
```

The one field a machine cannot fill is `golden_tools` — correct behaviour is a product decision. Everything else is already on the span, which is the payoff for having instrumented properly in Lab 01.

### Gate it in CI

`.github/workflows/evals.yml`

```yaml
on:
  pull_request:
    paths: ["agent/**", "prompts/**", "evals/**"]

jobs:
  evals:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e ".[evals]"
      - run: python -m evals.run --dataset evals/dataset.py --fail-under 0.9
        env:
          OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: ${{ secrets.OTLP_ENDPOINT }}
```

Note the `paths` filter: **prompt changes trigger the eval suite.** Most teams gate code and let prompts through on a text diff review — which is exactly how an 82%-to-64% accuracy regression ships on a Friday.

### Checkpoint 6

**Green sticky when**

- Your dataset has five cases and the promoted one fails.
- *Bonus round, if the clock allows:* you fixed it — a prompt line telling the agent to trust `order_not_found` — and re-ran to green.

---

## Taking it to production

Your agent code does not change. The endpoint and the auth do.

| Local today | Managed on AWS |
|---|---|
| `opensearch` container | **Amazon OpenSearch Service** (Serverless supported) — same `otel-v1-apm-*` indices, same PPL |
| `prometheus` container | **Amazon Managed Service for Prometheus** — same recording rules, registered as a Prometheus data source with SigV4 |
| `otel-collector` + `data-prepper` containers | **Amazon OpenSearch Ingestion** — the managed OTLP path |
| `localhost:5601` | **OpenSearch UI** — Agent Traces, Application Map, Services, SLOs, Unified Alerts |

```bash
npx @opensearch-project/observability-stack     # or CDK · ~15 min · VPC optional · --serverless
```

Underneath, still nothing proprietary: Prometheus for metrics, OpenSearch for logs and traces, OpenTelemetry for instrumentation. Apache 2.0, governed by the OpenSearch Software Foundation under the Linux Foundation.

### The three things that change at a hundred agents

1. **Volume.** A request that used to be one span is now twenty-two. Sample at the **trace** level, never the span level — a half-sampled reasoning chain is worse than none, because it reads as a complete chain that made no sense.
2. **Cardinality.** Labels are a budget. Bound the set deliberately: agent, model, operation, service. Anything identifying a user, a session or an order stays on the span.
3. **Content.** Tool arguments and model output are the fields most likely to carry customer data. Redact in the Collector, not in every agent.

All three are Collector-side decisions — one config, not a hundred agent redeploys.

---

## Reference

### Attributes you will keep using

| Attribute | Carries |
|---|---|
| `gen_ai.operation.name` | `chat` · `invoke_agent` · `execute_tool` · `embeddings` |
| `gen_ai.agent.name` | Which agent or sub-agent ran this step |
| `gen_ai.request.model` | The model actually called |
| `gen_ai.usage.input_tokens` / `output_tokens` | Token counts, per model call |
| `gen_ai.tool.name` + `.call.arguments` / `.call.result` | What the agent called and what came back |
| `gen_ai.conversation.id` | Session, across turns |

A framework has to emit OpenTelemetry spans **with these attributes**. The Agent Traces view surfaces spans carrying `gen_ai.operation.name`.

### Names that are easy to get wrong

| Say this | It is | Not to be confused with |
|---|---|---|
| **Agent Traces** | The plugin and page from Labs 01–04 | "Agent Tracing" — only a sidebar label |
| **Agent Evals** | The SDK: `score()`, `evaluate()`, `Benchmark`. Scoring is Python-side | Agent Health |
| **Agent Health** | Separate product, own repo, own UI on `:4001`. Pass/fail and accuracy live here | Agent Evals, which has no pass/fail primitive |
| **Application Map** | The UI label for the topology view | doc slug is `/docs/apm/service-map/` |
| **"one interface, two stores"** | Per-signal routing, correlated on trace ID and service name | "federated queries" — nothing joins a Prometheus series to an OpenSearch doc in one query |

Documented third-party eval libraries: **DeepEval, RAGAS, MLflow, pytest.** The MCP server requires **OpenSearch 2.19+**. `otel.opensearch.org` redirects to `observability.opensearch.org`.

### Links

- Slides — <https://anirudha.github.io/talks/pipe-dreams-to-pipelines/>
- Code — <https://github.com/anirudha/os-agent-observability-evals>
- Tutorial — <https://github.com/anirudha/os-agent-observability-evals/blob/main/blog.md>
- Docs — <https://observability.opensearch.org/docs/ai-observability/>
- Agent Evals — <https://observability.opensearch.org/docs/agent-evals/>
- Playground — <https://observability.playground.opensearch.org>
- Agent Health — `npx @opensearch-project/agent-health@latest` → `localhost:4001`
