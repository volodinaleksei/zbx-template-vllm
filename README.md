# Zabbix template: vLLM (LLM API)

Monitoring for [vLLM](https://github.com/vllm-project/vllm) OpenAI-compatible servers via Zabbix 7.x.

Built and tested on **Zabbix 7.4**; should work on 7.0+ (HTTP agent + dependent items + JS preprocessing).

## How it works

Three master items (`HTTP agent`, `TEXT` value type, history disabled) fetch raw responses;
everything else is **dependent** items with JavaScript preprocessing:

| Master item | Endpoint | Interval |
|---|---|---|
| `vllm.metrics.raw` | `http://{HOST.CONN}:{$VLLM.PORT}{$VLLM.METRICS.PATH}` (Prometheus) | 1m |
| `vllm.health.raw` | `/health` | 30s |
| `vllm.test.raw` | `{$VLLM.TEST.PATH}` (default `/v1/models`) | 5m |

Prometheus metrics are parsed with a regex inside a JS step — Zabbix HTTP agent has no native
Prometheus parsing without agent 2, and the monitored host may not run an agent at all.

## Items

- `vllm.requests.running` / `vllm.requests.waiting` — requests in flight / queued (summed across engines)
- `vllm.kv_cache.usage` — KV-cache usage, % (from `vllm:kv_cache_usage_perc` ×100)
- `vllm.ttft.avg` — average Time To First Token (`_sum` / `_count`)
- `vllm.preemptions` — `vllm:num_preemptions_total`
- `vllm.tokens.prompt` / `vllm.tokens.generation` — total tokens (use rate() on graphs)
- `vllm.process.rss` — process resident memory
- `vllm.health` / `vllm.test` — 1/0 availability flags
- `vllm.models.count` — number of models from `/v1/models`
- `net.tcp.service[http,,{$VLLM.PORT}]` — simple port check

## LLD (per model)

`vllm.model.discovery` (dependent on `/v1/models`) creates per model:
requests running/queued, successful requests, prompt/generation tokens, context size (`max_model_len`).

## Triggers

| Event | Severity | Condition |
|---|---|---|
| vLLM: server does not respond (/health) | High | `/health` unavailable for 5m |
| vLLM: port {$VLLM.PORT} unreachable | High | TCP closed while /health answered recently (process died, not network) |
| vLLM: API not responding (/health alive) | Medium | /health up, test path unavailable |
| vLLM: no models loaded | Warning | `/v1/models` returned an empty list |
| vLLM: request queue backlog | Warning | average queue > `{$VLLM.QUEUE.MAX}` over 5m |
| vLLM: KV-cache > {$VLLM.KV.MAX}% | Warning | risk of preemptions / admission rejections |
| vLLM: model {#MODELID} unloaded | Warning | model disappeared from `/v1/models` |

## Macros

| Macro | Default | Purpose |
|---|---|---|
| `{$VLLM.PORT}` | `8000` | vLLM server port |
| `{$VLLM.METRICS.PATH}` | `/metrics` | Prometheus metrics path |
| `{$VLLM.TEST.PATH}` | `/v1/models` | functional check path |
| `{$VLLM.QUEUE.MAX}` | `5` | queue warning threshold |
| `{$VLLM.KV.MAX}` | `90` | KV-cache warning threshold, % |

## Setup

1. Import `template_vllm.yaml` (Data collection → Templates → Import).
2. Create a host with an **Agent** interface set to the vLLM server IP — it is used only as
   `{HOST.CONN}`; no agent is required, all items are HTTP agent / simple checks.
3. Link the template. For HTTPS or auth, edit the master items manually.

Verified with live polling against a vLLM server running `qwen3-vl-8b`.

## Grafana

The metrics are standard Prometheus names (`vllm:*`) — Grafana can scrape `/metrics` directly.
This template is complementary alerting, not a Grafana replacement.
