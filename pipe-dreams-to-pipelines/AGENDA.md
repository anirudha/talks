# Pipe Dreams to Pipelines — Facilitator Notes

**Workshop · OpenSearchCon North America 2026 · Thu Sep 24 · 10:40–12:20 (100 min) · hands-on**
**Anirudha Jadhav** (Sr. Engineering Manager, OpenSearch · AWS) and **Wolfgang Theilmann** (SAP)

Published abstract:

> Participants will construct a comprehensive observability infrastructure for AI agents, progressing from initial instrumentation through production deployment. The session covers instrumenting agents with OpenTelemetry, directing traces via Data Prepper into OpenSearch, and aggregating OTel metrics through Prometheus. Attendees learn to generate service maps displaying request flows, reconstruct agent reasoning using PPL, correlate against RED metrics, benchmark against golden paths, and integrate production issues into evaluation frameworks.

Deck: `index.html` (56 slides, reveal.js). Participant guide: `LAB.md`.

---

## North star

> A workshop is not a talk with pauses. **The deliverable is that everyone leaves with something running on their own laptop** — so the slide count is deliberately low per minute, the checkpoints are non-negotiable, and every lab has a skip-forward.

The narrative hook is the title: three pipes. **The pipe dream** ("we'll add observability later"), **the data pipeline** (Collector → Data Prepper → OpenSearch, plus the Prometheus scrape), and **the PPL pipe** (`source = … | where … | stats …`). Slide 2 sets this up; call back to it on the Lab 02 and Lab 04 dividers.

The single thread that holds the 100 minutes together is **one bad trace**. It is created in Lab 01 (question 3: order `A-9999` does not exist, the tool says `order_not_found` four times, the model invents a delivery date), reconstructed in Lab 04, graded in Lab 05, and promoted to a regression test in Lab 06. **If you drop everything else, keep the specimen trace.** Every "so what" in the deck is that trace.

---

## Run of show

Clock times are on the divider slides so participants can self-pace. Hold them.

| Clock | Slides | Segment | Min | Driver | Ends on |
|---|---|---|---|---|---|
| 10:40 | 1–10 | Open + framing — three pipes, outcomes, run of show, prereqs, ground rules, why agents break differently, GenAI semconv | 8 | Ani | — |
| 10:48 | 11–14 | **Lab 00** · Stack up | 10 | Ani | Checkpoint 0 (5 tabs) |
| 10:58 | 15–21 | **Lab 01** · Instrument — `register()`, `@observe`, `enrich()`, run 3 questions | 14 | Ani | Checkpoint 1 (span I/O) |
| 11:12 | 22–27 | **Lab 02** · Pipelines — Collector fan-out, Data Prepper, indices, label bridge, metric families | 13 | **Wolfgang** | Checkpoint 2 (both indices) |
| 11:25 | 28–33 | **Lab 03** · Service map + RED, four hops, and "at a hundred agents" | 12 | **Wolfgang** | Checkpoint 3 (agent on map) |
| 11:37 | 34–40 | **Lab 04** · PPL — trajectory, tokens, tool failures, loop detector, write your own | 15 | Ani | Checkpoint 4 (own query) |
| 11:52 | 41–47 | **Lab 05** · Golden paths, `score()`, Agent Health, benchmarks | 16 | Ani | Checkpoint 5 (`0.25`) |
| 12:08 | 48–50 | **Lab 06** · Promote a trace, gate it in CI | 8 | Ani | Checkpoint 6 |
| 12:16 | 51–56 | Wrap — recap, naming card, to production, homework, CTA, close | 4 | both | — |

Whoever is not presenting **walks the aisles**. That is the job, not a courtesy — in a 100-minute hands-on session the person at the back finds the broken Docker VM eight minutes before the presenter would.

### If you are running long

Cut in this order. Each cut is clean and breaks nothing downstream.

1. **Slide 33** (at-a-hundred-agents) — the content survives as a verbal aside during Lab 02. Saves 3 min.
2. **Slide 52** (naming card) — it is a reference slide; it lives in `LAB.md` anyway. Saves 1.5 min.
3. **Slide 54** (homework) — fold the one line that matters ("point `register()` at your own agent tonight") into the close. Saves 2 min.
4. **Slide 46** (benchmarks) — slide 45 already lands "grading is real". Saves 2 min.
5. **Lab 06 down to slide 49 only** — drop the CI slide and say it. Saves 3 min.

That recovers ~11 minutes without touching a checkpoint. **Never cut a checkpoint slide** — they are the only mechanism you have for knowing whether the room is with you.

### If you are running short

The bonus round at the end of Lab 06 (actually fix the prompt and re-run to green) is the intended overflow. It is the most satisfying five minutes in the workshop and it is deliberately optional.

---

## Speaker split, concretely

**Ani** — the agent-side arc: framing, instrumentation, PPL, evals, closing the loop. Owns the specimen trace and the narrative.

**Wolfgang** — the platform-side arc: Labs 02 and 03. This is the right split because the pipeline and the scale story are where an operator's credibility matters more than an SDK author's. Specifically his:

- Slide 23–27: the Collector and Data Prepper configs, including the `resource_to_telemetry_conversion` line and the cardinality trap on slide 26.
- Slide 33: the three things that change at a hundred agents. **The deck deliberately keeps this structural rather than naming specific SAP numbers or systems** — fill it with whatever you can say publicly, or keep it generic. Nothing downstream depends on it.
- All "how does this behave at scale / what does it cost / how do you govern it" questions in the wrap.

Agree the handoff line in advance. Suggested: Ani closes Checkpoint 1 with *"you have spans. Wolfgang is going to show you where they actually go, which is the half that bites you in production."*

---

## The specimen trace — get this right

Lab 01 runs three questions. The third is the one that matters:

```
python -m agent.cli ask "where is order A-9999?"
```

Expected shape: ~14 spans, ~3.9 s, ~5,600 tokens, `lookup_order` called **four times**, answer invents a delivery date. Contrast with question 1 (6 spans, 1.24 s, 1,180 tokens, one tool call).

Tell the room to **write the trace ID down**. Then it is used:

| Lab | What the specimen does |
|---|---|
| 01 | Checkpoint — the `execute_tool` span says `order_not_found`, the root span's output says "shipped Tuesday" |
| 04, query 1 | Trajectory replay shows four identical 12 ms tool calls and five `chat` spans |
| 04, query 4 | Loop detector puts it at the top: 5 chat steps, ~2,845 output tokens |
| 05 | The `order-missing` case fails: `tool_path_match 0.25`, `no_confabulation 0.0`, `step_budget 0.0` |
| 06 | `evals.promote --trace <id>` turns it into case #5 |

**`tool_path_match 0.25`** is the number to land the workshop on. One expected call, four actual. Say it plainly: *"Twenty minutes ago this was a 200 OK."*

If the agent behaves differently on the day (a model that handles `order_not_found` correctly), do not fight it — use whatever the loop detector surfaces as the specimen instead, and say so. The mechanism is the point, not the specific trace.

---

## Numbers on the slides

Screenshot numbers are transcribed from the actual pixels of the bundled example stack (Travel Planner / weather-agent / events-agent / mcp-server). Terminal transcripts are the Acme support agent from the companion repo. **Keep the two examples verbally separate** — say which one is on screen.

- **Application map (slide 29)** — `weather-agent` 7 requests, `events-agent` 8, `mcp-server` 28.
- **Services catalogue (slide 30)** — `checkout` 95 ms P99 at **16.4% failure rate**; `events-agent` 248 ms.
- **Eval run scale (slide 9)** — 5 questions → **110 spans**, 22 per trace. Only **10 of the 110** carry token usage: the `chat` spans. The slide rounds this to "2 of 22" per trace; both are true, don't mix them mid-sentence.
- **Indices (slide 25)** — 347 spans → 9 service-map edges. The ratio is the teaching point, not the absolute numbers.
- **Benchmarks (slide 46)** — Travel Planner on `claude-sonnet-4.5`, five runs:

| | #1 | #2 | #3 | #4 | #5 |
|---|---|---|---|---|---|
| Pass rate | 71% | 57% | 57% | 43% | 29% |
| Avg accuracy | 82% | 72% | 71% | 64% | 50% |
| Tokens | 21.6M | 19.8M | 18.2M | 8.8M | 5.7M |
| Cost | $65.71 | $60.33 | $55.52 | $27.28 | $17.78 |

  Land: **quality and cost move together.** The best run is also the most expensive. That is exactly why they belong in one table rather than two teams' dashboards.

---

## Naming — get these right on stage

The docs contain real naming drift. Slide 52 is the reference; these are the ones that matter live.

| Say this | Not this | Why |
|---|---|---|
| **Agent Traces** | "Agent Tracing" | `Agent Traces` is the plugin and page H1; "Agent Tracing" is only a sidebar label |
| **Agent Evals** | Agent Health for the `:4001` app | **Two products, not a rename.** Separate repos, separate sidebar groups |
| **Agent Health** | Agent Evals for pass/fail | Pass/Fail and the 0–100% accuracy figure are **Agent Health only** |
| **Application Map** | "service map" (for the UI) | Doc slug is `/docs/apm/service-map/`, UI label is Application Map. `service_map` *is* correct for the Data Prepper processor |
| **"one interface, two stores"** | "federated queries" | The docs never use federate/federated/federation |
| **AWS Bedrock** | "Amazon Bedrock" | The Agent Health docs' exact wording for its judge |

Two more:

- **"Benchmark" means two different things.** In Agent Evals it is the SDK class that uploads external results; in the Agent Health CLI it is what the UI calls an experiment. Don't use the word bare while both are on screen.
- **Agent Evals has no pass/fail primitive** and `score(value=…)` is an unbounded float. Slide 44 is careful about this; stay careful verbally.

---

## Claims to state carefully

1. **"Federates queries across both stores."** It does not. Slide 13 says "one interface, two stores" and names the real mechanism: a Prometheus data source for metrics, OpenSearch datasets for traces and logs, joined by **trace ID and service name**. If asked whether it is a federated query engine — **no**. Nothing joins a Prometheus series to an OpenSearch document in a single query. The UI routes per signal and correlates on trace ID.
2. **"Any framework emitting spans."** Nearly right, with one condition: a framework must emit OTel spans **with GenAI semantic-convention attributes**. Slide 10's intro line says it in so many words: the Agent Traces view surfaces spans carrying `gen_ai.operation.name`.
3. **Auto-instrumentation is opt-in and Python-first.** You install the extra (`[bedrock]`, `[anthropic]`, `[openai]`, `[langchain]`, `[llamaindex]`). The TypeScript SDK is a **function wrapper, not a decorator**, with narrower coverage today. Slide 19 says both.
4. **AMP for ingest is not what the companion repo does.** The repo's Prometheus is pull-only — the Collector's `prometheus` exporter is scraped. AMP appears on the **query** side (slide 53). If asked about writing metrics to AMP, that is the OTLP/remote-write path in the managed deployment, not something the sample repo configures.
5. **Slide 50's CI workflow and slide 49's `evals.promote` are the intended shape**, matched to the companion repo's layout. If the repo's flag names have drifted by the day, correct them verbally rather than pretending — participants are reading `LAB.md` at the same time and will notice.

---

## Likely questions

- **"What does storing all these spans cost?"** No pricing or sampling guidance specific to agent traces is published. Point at generic OTel tail sampling, at the 347-spans-to-9-edges ratio on slide 25 as the retention argument, and offer to follow up rather than quoting a number.
- **"Which version has this?"** The docs publish no version numbers for the SDKs, Agent Health, or the Agent Traces plugin. Only the MCP server states a floor: **OpenSearch 2.19+**.
- **"Does the SLO app deploy rules into AMP?"** Unverified. The SLO docs describe "the Prometheus rule engine"; the local stack uses the Cortex ruler plus Alertmanager. Don't assert it.
- **"LlamaIndex example?"** The `[llamaindex]` extra exists, but there is no LlamaIndex code example in the docs. Documented framework integrations: Strands, LangGraph, CrewAI, OpenAI Agents SDK, Bedrock.
- **"Why is `gen_ai.provider.name` sometimes `bedrock` and sometimes `aws.bedrock`?"** The docs are inconsistent. Refer to the attribute, not a hard-coded value.
- **"Can I use my own eval library?"** Four are documented: **DeepEval, RAGAS, MLflow, pytest.** Promptfoo and OpenAI Evals appear nowhere — don't offer them.
- **"Can the chatbot investigate for me?"** What ships today is the **Claude Code Observability Plugin** (eight Agent Skills: Traces, Logs, Metrics, Stack Health, PPL Reference, Correlation, APM RED, SLO/SLI) over the built-in MCP server. Slide 54 mentions it. A hypothesis-driven investigation agent is not published — if you go there verbally, say "where we're going", not "you can do this today".

---

## Room mechanics

**Sticky notes, three colours, handed out at the door.** Green = at the checkpoint. Red = stuck, come here. Amber = question for the wrap. This is on slide 6. It is worth 30 seconds of setup because it replaces the "raise your hand if you're done" ritual that never works in a room of 60 people staring at terminals.

**Pair up.** Say it explicitly at slide 6: one laptop between two people beats two half-working laptops, and the person not typing reads the checkpoint out loud.

**Every lab has `make lab-NN`** that jumps to that lab's starting state. Say this at slide 6 and again before Lab 02. Falling behind on the pipeline lab must not cost anyone the evals lab.

**The escape hatch is the playground.** <https://observability.playground.opensearch.org> — same stack, live agent traces already flowing, nothing to install. Labs 03–05 work read-only and every PPL query in Lab 04 runs verbatim. Anyone whose Docker is hopeless goes there at 10:55 rather than losing the whole session.

---

## Pre-flight

**The night before**

- Run all six labs end to end on the presenting laptop, from a clean clone. Not a partial run.
- Note the actual specimen trace numbers you get and, if they differ from the deck's transcripts, decide whether to update the slides or narrate the difference.
- Confirm `make lab-01` … `make lab-06` all land in a working state.
- Pull every Docker image on the presenting laptop and on a spare.
- Load `index.html` once so reveal.js, Inter and JetBrains Mono are in browser cache — **the deck loads all three from jsDelivr**. If the venue blocks CDNs it falls back to system fonts and the layout suffers. Vendor the files locally if the venue is known to be hostile.

**Fifteen minutes before**

- Open `index.html`, press `?` to confirm reveal.js keybindings are live, then `Esc` for the overview to check all 56 thumbnails render.
- Present at **1440×960 or wider**, browser zoom 100%, bookmarks bar hidden. The slide frame is 1280×860; narrower windows scale down and the `.term` transcripts suffer first.
- `docker compose up -d` on the presenting laptop **now**, and run `agent.cli seed --questions 12` so the map and catalogue have data by Lab 03.
- Both new QR codes were generated and decode-verified: `qr-lab.png` → the lab guide, `qr-deck.png` → the published deck. `qr-repo.png`, `qr-docs.png` and `qr-playground-live.png` carry over from the CloudOps deck and were verified there.
- Write the lab-guide URL on the physical whiteboard if there is one. It is the one URL people need and it is not memorable.

**Assets**

Every screenshot is local under `assets/`. Nothing on a slide needs the network at presentation time except the two CDN font/CSS loads. Spares carried over and available if you want to add a slide: `crop-slo-catalog.png`, `crop-slo-kinds.png` (SLOs on agents), `crop-timeline.png`, `crop-trace-map.png` (trace views), `crop-tokens-kpi.png`, `crop-tokens-builder.png` (token metrics), `crop-metrics-header.png`, `crop-metrics-cards.png` (metrics explorer), `crop-dash-faultrate.png`, `crop-dash-agentlogs.png` (one dashboard, both stores), `crop-foundation.png`, `slide-numbers.png` (OpenSearch credibility openers).

---

## Links

- Slides — <https://anirudha.github.io/talks/pipe-dreams-to-pipelines/>
- Lab guide — [`LAB.md`](./LAB.md)
- Code — <https://github.com/anirudha/os-agent-observability-evals>
- Tutorial — <https://github.com/anirudha/os-agent-observability-evals/blob/main/blog.md>
- Docs — <https://observability.opensearch.org/docs/ai-observability/>
- Agent Evals — <https://observability.opensearch.org/docs/agent-evals/>
- Playground — <https://observability.playground.opensearch.org>
- Agent Health — `npx @opensearch-project/agent-health@latest` → `localhost:4001`

> `otel.opensearch.org` redirects to `observability.opensearch.org` — use the latter on stage.
