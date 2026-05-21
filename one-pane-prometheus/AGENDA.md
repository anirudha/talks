# One Pane to Rule Them All
**Uniting the Prometheus Community with OpenSearch Dashboards, Logs & Traces**

Observability Summit NA 2026 · Anirudha Jadhav & Kevin Fallis (AWS) · 30 min

---

## North star
"Keep your Prometheus metrics stack exactly as it is, and get a single pane (and API for humans and AI) for logs, metrics, traces, and alerts across every cluster, in OpenSearch."

Additive. Open source. Standards-first. Not rip-and-replace.

---

## Agenda

1. The pane problem: why "single pane" keeps failing
2. Prometheus metrics, natively: design principles & architecture
3. **Demo I** — Discover Metrics & the metrics builder
4. Dashboards + variables across logs, metrics, traces
5. **Demo II** — Alerts, SLO/SLI catalog
6. How to join: playground, docs, TAG

---

## Slide-by-slide flow (28 slides)

| # | Slide | Beat |
|---|---|---|
| 1 | Cover | Title, speakers, event |
| 2 | The OpenSearch platform | Core · Dashboards · Data Prepper |
| 3 | How people use OpenSearch | Search · Observability (highlight) · Security · AI/ML |
| 4 | By the numbers | OpenSearch scale + adoption |
| 5 | Foundation | OpenSearch Software Foundation under Linux Foundation |
| 6 | Members | Foundation members logos |
| 7 | Bridge to topic | "Keep your Prometheus metrics. Get one pane for everything else." |
| 8 | Agenda | This list |
| 9 | Part 1 divider | The Pane Problem |
| 10 | Operator's reality at 3 AM | Five tabs, three query languages, one incident |
| 11 | Pull-quote | "What if the pane already exists where your logs and traces live?" |
| 12 | The missing leg | Logs + traces already in OpenSearch; metrics was the gap |
| 13 | Part 2 divider | Prometheus Metrics, Natively |
| 14 | Three rules we held ourselves to | Don't fork PromQL · don't replace Alertmanager · don't lock data in |
| 15 | How it fits together | Architecture: Dashboards ↔ Prometheus datasource ↔ {Prom, federated, multi-cluster} |
| 16 | "One pane" thesis in four pixels | Datasource picker — Prometheus alongside OpenSearch logs/traces |
| 17 | Part 3 divider | Demo I · Discover Metrics |
| 18 | Finding metrics shouldn't feel like archaeology | A–Z grid, histogram badges, filter to a metric |
| 19 | Five tabs, one query language | Explore · Row · Columns · Metrics builder · Query — all PromQL on the wire |
| 20 | Part 4 divider | Dashboards + Variables |
| 21 | Astronomy Shop dashboard | Business + telemetry on one canvas, one time picker |
| 22 | Variables across all three signals | PromQL + PPL driven, refresh-on-load |
| 23 | Part 5 divider | Demo II · Alerts, SLOs |
| 24 | Every alert. Every datasource. One list. | Unified OpenSearch text/log alerts + Prometheus metrics alerts |
| 25 | Alertmanager, finally legible | Read-only routing tree, receivers, matchers |
| 26 | SLOs as first-class citizens | Filter by service / team / SLI type, worst-error-budget-first |
| 27 | Get involved (CTA) | Playground · Docs · Stack · TAG + QR |
| 28 | Q&A | Speakers + contact |

---

## Demo guardrails

- **Pre-flight (30 min before):** load every URL in fresh tabs, force-refresh, verify no `Failed to load datasources` banner. Have screenshots queued for every live step.
- **Time budget:** two demos × 5–6 min ≈ 11 min. If either runs long, drop the multi-cluster flip in Demo II and keep the SLO drill-down.
- **Browser:** Chrome, zoom 110%, hide bookmarks bar, close all extensions.

---

## Key links

- Live playground: https://observability.playground.opensearch.org
- Docs: https://observability.opensearch.org/docs/investigate/discover-metrics/
- Stack: https://opensearch.org/platform/observability-stack
- TAG: https://opensearch.org/slack · #observability
- Talk: https://observabilitysummitna26.sched.com/event/2HJVi/

---

## Core message

> Don't migrate off your Prometheus metrics.
> Migrate off the *five tabs around it.*
