# Standards First

**How OpenSearch Aligned Its Observability Stack with OpenTelemetry + Prometheus**
Observability Day · Nov 9 · 17:15–17:25 · Salt Palace, Level 2, 250 A-C
Dotan Horovits · Anirudha Jadhav (AWS)

Deck: [`index.html`](./index.html) · [live](https://anirudha.github.io/talks/standards-first/)
Session: https://events.linuxfoundation.org/kubecon-cloudnativecon-north-america/co-located-events/cncf-hosted-co-located-schedule/?id=1269147

## 8 slides, 10 minutes

Visual deck: each slide is one headline and one diagram or one screenshot. Nothing on a slide needs
to be read out, so the words are yours.

| # | Slide | What you say over it | Time |
|---|---|---|---|
| 1 | Title + speakers | Hello, who we are. | 0:00 |
| 2 | "We stopped building adapters" + layer stack | OTel and Prometheus stopped being decisions. Walk the seven rows; this is the spine. | 0:30 |
| 3 | Five front doors → one OTLP endpoint | The sprawl was visible, the field names were the real problem. Land `service.name`. | 1:50 |
| 4 | "Keep your Prometheus" + datasource picker | Not re-educating the market. Datasource, not migration. | 3:20 |
| 5 | It all comes together (3 screenshots, no labels) | Logs, metrics and traces payoff. Topology map, one canvas, one alert list across both datasources. Let the screens talk. | 4:40 |
| 6 | Ten vocabularies → one normalizer | Same bet one layer up. Upstream in the OTel Collector; the OpenSearch GenAI SDK lights it up. Footer carries both links. | 5:50 |
| 7 | Every hop. Every token. (agent traces) | Same spans, agent-shaped: traces list with tokens and latency, then the trace tree and trace map for one request. | 7:30 |
| 8 | "Come build with us" + 5 QR codes + booth | Playground, docs, GenAI SDK, the stack, Slack. Then the booth: end on the invite. | 8:50 |

Live demo goes wherever you want it; slides 5 and 7 double as the fallback if the stack misbehaves.

Running long: compress slide 2 to the first three rows. Protect 3, 4, 5 and 7.

## QR codes

Generated for this deck, each encoding exactly:

| Asset | URL |
|---|---|
| `qr-playground.png` | https://observability.playground.opensearch.org |
| `qr-stack-docs.png` | https://observability.opensearch.org/docs/ |
| `qr-genai-integrations.png` | https://observability.opensearch.org/docs/send-data/ai-agents/integrations/ |
| `qr-stack-platform.png` | https://opensearch.org/platform/observability-stack/ |
| `qr-slack-tag.png` | https://opensearch.org/slack |
| `qr-eval-integrations.png` | eval-library integrations page (spare, not on a slide) |
| `qr-deck.png` | this deck (spare, for the booth) |

Slide 8 ends on an orange booth band: "The OpenSearch booth, KubeCon expo hall." No booth number in it. Add one if you have it.

## Live demo checklist

- Prometheus datasource registered and green
- An agent trace with `gen_ai.*` spans loading
- One dashboard with a Prometheus panel and an OpenSearch panel under a shared variable
- Alerts list showing both sources

## Sources

- Agent integrations: https://observability.opensearch.org/docs/send-data/ai-agents/integrations/
- Eval-library integrations: https://observability.opensearch.org/docs/agent-evals/evaluation-integrations/
