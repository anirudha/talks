# AI Agent Monitoring with the OpenSearch Observability Stack

**CloudOps Webinar · Tue Sep 15, 2026 · 08:00–09:00 PDT · NAMER / English · Anirudha Jadhav (OpenSearch, AWS)**

Slides-only, no live demo. Every screen in the deck is a baked-in screenshot, so there is zero live risk — but it also means the numbers on the slides are the numbers you must speak to. They are all transcribed from the actual pixels; see **Numbers you can say out loud** below.

---

## North star

> Agents don't behave like microservices. Rate, errors and duration will never show you a hallucination, a quality regression, or a doubled token bill. Span-level GenAI telemetry will — and it runs on open projects you already know: Prometheus for metrics, OpenSearch for logs and traces, OpenTelemetry for instrumentation.

Additive, open-source, standards-first. Managed on AWS as **AMP** + **Amazon OpenSearch Service**.

---

## Run of show (52 slides / ~48 min + Q&A)

| Slides | Section | Minutes | Beat |
|---|---|---|---|
| 1–7 | Open | 6 | Cover, project credibility, **the AWS managed stack**, three open projects, "your agent is not a microservice", agenda |
| 8–13 | **Part 1** · Agents break differently | 7 | Hop chain, black box, three failure modes, the 22-spans number |
| 14–19 | **Part 2** · One vocabulary, two stores | 8 | GenAI semconv, span categories, AMP + AOS architecture, why two stores, the label-promotion bridge |
| 20–24 | **Part 3** · Instrument your agent | 7 | `register()`, `@observe`, coverage, six runtimes |
| 25–32 | **Part 4** · Walk the reasoning chain | 8 | Trace tree, metrics bar, timeline, trace map, span I/O, PPL, token builder |
| 33–36 | **Part 5** · Metrics beside traces | 5 | Metrics explorer, PromQL RED rules, one dashboard |
| 37–41 | **Part 6** · Health view to exact span | 6 | Application map, services, four hops, SLOs |
| 42–48 | **Part 7** · Grade the agent | 8 | Agent Evals, golden path, test case, benchmarks, Claude Code + MCP, preview |
| 49–52 | Close | 3 | Production config change, recap, CTA, closing |
| — | **Q&A** | 10–13 | — |

**If you are running long,** these five slides cut cleanly without breaking the narrative: **17** (why-two-stores table), **24** (six runtimes), **38** (services catalogue), **43** (golden-path code — slide 42 already makes the point), **49** (production config). That takes you to 47 slides / ~41 min.

---

## Naming — get these right on air

The docs contain real naming drift. Use the left column.

| Say this | Not this | Why |
|---|---|---|
| **Agent Traces** | "Agent Tracing" | `Agent Traces` is the plugin and page H1; "Agent Tracing" is only a sidebar label |
| **AI Observability** | — | Page H1. The nav section label is "Agent Observability"; both are live in the docs |
| **Agent Evals** | — | SDK-side: `score()`, `evaluate()`, `Benchmark`. Emits scores as spans through the same OTLP pipeline |
| **Agent Health** | "Agent Evals" for the :4001 app | **A separate product** — own repo, own npm package, own UI on `localhost:4001`. Do not conflate the two |
| **Application Map** | "service map" | The doc slug is `/docs/apm/service-map/` but the UI label is Application Map |
| **OpenSearch Dashboards** (OSS) / **OpenSearch UI** (AWS) | using them interchangeably | In these docs "OpenSearch UI" specifically means the Amazon OpenSearch Service surface |
| test case | — | Agent Health displays it as **"Use Case"** in the UI, and calls an experiment a **"Benchmark"** in the CLI. Match whichever surface is on screen |

**The Agent Evals / Agent Health distinction, precisely** — verified against the docs' raw markdown source:

- They are **two products, not a rename.** `/docs/ai-observability/` lists them as two sibling bullets under one "Evaluate" heading; separate sidebar groups, separate GitHub repos. Never say "renamed" — the redirect table shows Agent Evals absorbed `/ai-observability/evaluation`, never anything under `agent-health`.
- **"Benchmark" means two different things across them.** In Agent Evals it is the SDK class that uploads external results; in the Agent Health CLI it is what the UI calls an experiment. Don't use the word bare while both are on screen.
- **Agent Evals has no pass/fail primitive**, and `score(value=...)` is an unbounded float. **Pass/Fail and the 0–100% Accuracy figure are Agent Health only** — which is why slide 45's percentages are labelled Agent Health and slide 42's `score()` calls are not.
- Agent Health's judge runs on **"AWS Bedrock"** — that is the docs' exact wording, not "Amazon Bedrock".
- Exactly four third-party eval libraries are documented: **DeepEval, RAGAS, MLflow, pytest.** Promptfoo and OpenAI Evals appear nowhere — don't offer them.
- Agent Evals scoring is **Python-side**; the docs are self-contradictory on TypeScript status, so say "instrumentation in Node, scoring in Python" rather than claiming TS parity.

---

## Two examples, keep them separate

The deck deliberately uses two different agents. Say which one you are on.

- **Travel Planner / weather-agent / events-agent / mcp-server** — the example services bundled with the stack. These are what every *screenshot* shows (slides 9, 25–32, 37–38).
- **Acme support agent** — the companion repo. This is what every *code slide* shows (slides 20–24, 42–43, 49). Three deterministic tools: `lookup_order`, `check_inventory`, `search_policy`.

---

## Numbers you can say out loud

All transcribed from the screenshots in the deck.

**The trace (slides 9, 25, 27)** — `POST /plan`: **1.71 s**, **37 spans**, **2,970 tokens**, status success.
`chat planning` 284.50 ms → `invoke_agent weather-agent` 1.14 s → `execute_tool get_current_weather` 135.91 ms → `tools/call fetch_weather_api` 135.52 ms → `POST /mcp` 124.49 ms.
The tool span and the MCP call under it are **0.4 ms apart** — that is the "was the tool slow, or the thing it called?" moment.

**The metrics bar (slide 26)** — 169 traces · 3.5K spans · 144 errors · 381K tokens · P50 1.58 s · P99 7.73 s.

**The eval run (slide 13)** — 5 questions → **110 spans** across 5 traces, 22 per trace. Only **10 of the 110** carry token usage: the `chat` spans.

**Application map (slide 37)** — `weather-agent` 7 requests, `events-agent` 8, `mcp-server` 28.

**Services catalogue (slide 38)** — `checkout` 95 ms P99 at **16.4% failure rate**; `events-agent` 248 ms.

**Metrics explorer (slide 33)** — **2,340** Prometheus metrics, 40 previewed at a time.

**Benchmarks (slide 45)** — Travel Planner on `claude-sonnet-4.5`, five runs:

| | #1 | #2 | #3 | #4 | #5 |
|---|---|---|---|---|---|
| Pass rate | 71% | 57% | 57% | 43% | 29% |
| Avg accuracy | 82% | 72% | 71% | 64% | 50% |
| Tokens | 21.6M | 19.8M | 18.2M | 8.8M | 5.7M |
| Cost | $65.71 | $60.33 | $55.52 | $27.28 | $17.78 |

The point to land: **quality and cost move together here.** The best run is also the most expensive. That is exactly why they belong in one table.

**SLOs (slide 40)** — `checkout` and `mcp-server` on p95 < 500 ms at 95.0%; `frontend` and `product-catalog` on availability at 99.0%. Canonical-kind filter includes **GenAI availability**.

---

## Claims to state carefully

Three things in the published abstract need care. Two are handled in the deck; know why.

1. **"Federates queries across both stores at runtime."** The docs never use the words federate/federated/federation. Slide 16 says **"one interface, two stores"** instead, and names the real mechanism: a Prometheus data source for metrics, OpenSearch datasets for traces and logs, joined by **trace ID and service name**. The defensible quote is on the slide: *"It combines topology data stored in OpenSearch with time-series RED metrics (Rate, Errors, Duration) stored in Prometheus."*
   **If asked "is it a federated query engine?"** — no. Nothing joins a Prometheus series to an OpenSearch document in a single query. The UI routes per signal and correlates on trace ID.

2. **"Context-aware chatbot and hypothesis-driven investigation agent."** Not in the published docs. Slide 47 carries a **Preview · not yet released** badge and the conversation on it is explicitly labelled illustrative. Slide 46 covers what ships today: the **Claude Code Observability Plugin** (eight Agent Skills: Traces, Logs, Metrics, Stack Health, PPL Reference, Correlation, APM RED, SLO/SLI) and the **built-in MCP server**. Keep the preview framing verbal too — "this is where we're going", not "you can do this today".

3. **"Any framework emitting spans."** Nearly right, with one condition worth saying: a framework must emit OpenTelemetry spans **with GenAI semantic-convention attributes**. The Agent Traces list only surfaces spans carrying `gen_ai.operation.name`. Slide 15 makes this explicit.

Two smaller ones:
- **Auto-instrumentation is opt-in and Python-first.** You install the extra (`[bedrock]`, `[anthropic]`, `[openai]`, `[langchain]`, `[llamaindex]`). The TypeScript SDK is a **function wrapper, not a decorator**, and its coverage is narrower today. Slide 22 says both.
- **AMP for ingest is not in the companion repo.** The repo's Prometheus is pull-only — no `remote_write`, no SigV4 ingest. AMP appears on the **query** side (slide 49). If asked about writing metrics to AMP, that is the OTLP/remote-write path in the managed deployment, not something the sample repo configures.

---

## Likely questions

- **"Cost of storing all these spans?"** No pricing or sampling guidance specific to agent traces is published. Point at generic OTel sampling and offer to follow up rather than quoting a number.
- **"Which version is this in?"** The docs publish no version numbers for the SDKs, Agent Health, or the Agent Traces plugin. Only the MCP server states a floor: **OpenSearch 2.19+**.
- **"Does the SLO app deploy rules into AMP?"** Unverified. The SLO docs describe "the Prometheus rule engine"; the local stack uses the Cortex ruler plus Alertmanager. Don't assert it.
- **"LlamaIndex demo?"** The `[llamaindex]` extra exists, but there is no LlamaIndex code example in the docs. Framework Integrations covers Strands, LangGraph, CrewAI, OpenAI Agents SDK and Bedrock.
- **"Why is `gen_ai.provider.name` sometimes `bedrock` and sometimes `aws.bedrock`?"** The docs are inconsistent. Refer to the attribute, not a hard-coded value.

---

## Pre-flight (15 min before)

- Open `index.html` and press **`?`** once to confirm reveal.js keybindings are live, then **`Esc`** for the slide overview to check all 52 thumbnails render.
- The deck loads reveal.js, Inter and JetBrains Mono **from jsDelivr**. If the webinar network blocks CDNs the deck falls back to system fonts and loses layout. Load it once on the presenting machine beforehand so it is in browser cache, or vendor the three files locally.
- Present at **1440×960 or wider**, browser zoom 100%, bookmarks bar hidden. The slide frame is 1280×860; narrower windows scale down and the smallest captions suffer.
- All four QR codes were decoded and verified: tutorial, playground, sample code, docs. The playground link answers with an anonymous-auth redirect first — it resolves in a browser, so scan it once yourself to be sure it lands.
- Every screenshot is local under `assets/`. Nothing on a slide needs the network at presentation time except the two CDN font/CSS loads.

## Links

- Tutorial: https://github.com/anirudha/os-agent-observability-evals/blob/main/blog.md
- Sample code: https://github.com/anirudha/os-agent-observability-evals
- Docs: https://observability.opensearch.org/docs/ai-observability/
- Agent Evals: https://observability.opensearch.org/docs/agent-evals/
- Playground: https://observability.playground.opensearch.org
- Agent Health: `npx @opensearch-project/agent-health@latest` → `localhost:4001`

> `otel.opensearch.org` redirects to `observability.opensearch.org` — use the latter on air.
