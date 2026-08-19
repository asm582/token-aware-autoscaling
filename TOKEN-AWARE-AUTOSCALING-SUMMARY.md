# Token-aware autoscaling on llm-d P/D — design summary

## Introduction

**P/D disaggregation** splits inference across two different kinds of pod. **Prefill** pods read the incoming
prompt and build its KV cache — compute-heavy, over in a moment. **Decode** pods then generate the answer one
token at a time — memory-heavy, and they hold the request for its whole lifetime. The KV cache is copied from
prefill to decode over a side channel (NIXL). Splitting the roles means each can be sized against its own
bottleneck instead of both being dragged along by one number.

**The EPP** (Endpoint Picker) is llm-d's router. For every request it chooses which prefill pod and which
decode pod will serve it, and it publishes metrics about what it dispatched.

**KEDA** is the autoscaler. Each role gets a `ScaledObject` holding one Prometheus query and one threshold.
KEDA runs the query, divides the result by the threshold, and rounds up — that is the replica count. So the
query's job is to answer *"how many pods' worth of work is outstanding right now?"*

**"Token-aware"** means those queries count **tokens**, not requests. A 8192-token prompt is 16× the work of
a 512-token one; a request counter rates them the same, and that is the gap this design closes.

## 1. Prerequisite: measure your prefill token rate

Token-aware autoscaling works by dividing **tokens of queued work** by **tokens per second of capacity**.
That second number is `peakPrefillThroughput`, and `guides/recipes/router/calibration/calibrate.sh` is what
measures it — sustained prefill tokens/s **through the full P/D path**, including the KV transfer and sidecar
hop.

**This is why it's a prerequisite, not tuning.** It is the unit conversion the whole design rests on — get it
wrong and the threshold is meaningless. For when to re-run it and how, see the
[calibrate.sh guide](https://github.com/llm-d/llm-d/blob/main/guides/recipes/router/calibration/README.md).

Measure it on your own hardware, through your own P/D path, rather than borrowing a published figure. The
value it produces — **`peakPrefillThroughput`** — is what the **prefill** trigger divides by, so it sets when
prefill scales and how accurately. Nothing on the decode side reads it.

## 2. `peakPrefillThroughput` and the 1.5 s SLO

Two components read `peakPrefillThroughput`, and **both act on prefill only — nothing in the decode path uses
it at all**:

1. **The EPP router**, as a `prefix-cache-affinity-filter` parameter — it prices recomputing a prompt against
   reusing a cached prefix when choosing which prefill pod gets the request.
2. **The KEDA prefill trigger**, as the denominator that converts queued tokens into seconds of backlog.

**`peakPrefillThroughput` is not the KEDA threshold.** It sits inside the Prometheus query, where it converts
a token count into a number of seconds. KEDA then takes that result — a duration — and compares it against its
own threshold, `1.5`:

```
replicas = ceil(  queued_tokens ÷ peakPrefillThroughput  ÷  1.5  )
                 └──── converts tokens → seconds ────┘   └ the threshold ┘
```

**Where 1.5 comes from: your TTFT SLO.** It is not measured from anything — it is the share of your
time-to-first-token budget you're willing to spend waiting in the prefill queue. Queue wait is only one part of
TTFT; a request also pays its own prefill pass, the KV transfer to decode, and the first decode step. So 1.5 s
only makes sense if your TTFT target sits comfortably above it — roughly a few seconds, which suits
long-context or batch-style traffic. For interactive chat aiming at ~2 s TTFT, 1.5 s of pure queueing is most of
the budget and you should set it lower.

**This is a tuning parameter.** Lower it to scale earlier and hold more prefill pods; raise it to tolerate more
queueing and run leaner. Being a little off costs slower first tokens, not failures — prefill latency degrades
gradually.

Note: with `always-disagg-pd-decider` configured, `peakPrefillThroughput` does **not** decide whether to
disaggregate a request — every request goes through the prefill/decode split.

## 3. The decode threshold is a different kind of number

Decode has no equivalent of `peakPrefillThroughput` — no measured rate, no unit conversion. Its query returns
KV cache occupancy, which is already a fraction of capacity, so the threshold `0.8` is compared against it
directly. The unit is **"pods' worth of full KV cache."**

**`0.8` is a tuning parameter — how much headroom you keep.** When a decode pod's KV cache fills, vLLM stops
accepting requests outright, so you scale before reaching it: `0.8` means "act at 80% full." Lower it if you
want more margin, raise it to run hotter.

## 4. The two queries

**Prefill — 100% llm-d metrics, no vLLM.** `AverageValue`, `threshold: 1.5`, min 1 / max 10.

Reads: *tokens the router has queued on live prefill pods, divided by prefill capacity in tokens/s.*

```promql
(
  sum(
      label_replace(
        llm_d_epp_inflight_tokens{namespace="pd-test",
                                  producer_name="inflight-load-producer",
                                  endpoint_name=~".*prefill.*"},
        "target_pod", "$1", "endpoint_name", "(.+)")
    and on (target_pod)
      label_replace(
        llm_d_epp_per_endpoint_queue_size{name="qwen-qwe-ea5367d7-wen3-32b-router",
                                          model_server_endpoint=~".*prefill.*"},
        "target_pod", "$1", "model_server_endpoint", "(.+)")
  ) / 15928        # peakPrefillThroughput — your calibrate.sh figure (§1), not a constant
) or vector(0)
```

**Decode — one vLLM metric.** `threshold: 0.8`, min 1 / max 10.

Reads: *how full the decode KV caches are, added up across pods.*

```promql
sum(vllm:kv_cache_usage_perc{namespace="pd-test", pod=~".*decode.*"}) or vector(0)
```

**Why decode uses a vLLM metric.** The autoscaler needs a KV reading **per pod**, so it can filter to decode
pods and sum them. vLLM reports exactly that. The EPP's only KV metric is a single pool-wide average covering
both roles, which cannot be split by role.

## 5. EPP plugins in P/D mode (live `pd-config.yaml`, 11 plugins)

| Plugin | Purpose |
|---|---|
| `disagg-headers-handler` | Attaches the chosen prefill target as a header, before the request is sent |
| `always-disagg-pd-decider` | Decides to disaggregate — here, unconditionally |
| `disagg-profile-handler` | Runs both scheduling profiles, merges results |
| `prefill-filter` / `decode-filter` | Split the one shared pool by the `llm-d.ai/role` label |
| `approx-prefix-cache-producer` | Tracks which pods already hold a prompt's prefix (`maxPrefixTokensToMatch: 131072`) |
| `inflight-load-producer` | Publishes per-pod `inflight_tokens`/`inflight_requests` — the prefill trigger's source |
| `prefix-cache-affinity-filter` | Prefers cache-warm prefill pods; **consumes `peakPrefillThroughput`** |
| `token-load-scorer` | Ranks prefill pods by **queued token load** |
| `active-request-scorer` | Ranks decode pods by concurrent requests |
| `max-score-picker` | Picks the winner — runs **twice per request** (10804 = 2 × 5402) |

**How a request flows:** the `prefill` profile (filter → affinity → token-load-scorer → picker) and the
`decode` profile (filter → active-request-scorer → picker) both resolve in **one** scheduling cycle, before
anything is dispatched. The request is then routed to the **decode** pod, whose `routing-proxy` sidecar calls
the prefill pod named in the header, pulls the KV cache back over NIXL, and generates.

## References

The token-velocity approach this design follows — sizing each role by dividing a token rate by that role's
token throughput, so prefill and decode share a common denominator in tokens/s — comes from:

> Ruiqi Lai, Hongrui Liu, Chengzhi Lu, Zonghao Liu, Siyu Cao, Siyang Shao, Yixin Zhang, Luo Mai, and
> Dmitrii Ustiugov. **"TokenScale: Timely and Accurate Autoscaling for Disaggregated LLM Serving with Token
> Velocity."** arXiv:2512.03416 [cs.DC], December 2025. <https://arxiv.org/abs/2512.03416>

llm-d documentation:

- [P/D disaggregation guide](https://github.com/llm-d/llm-d/tree/main/guides/pd-disaggregation) — the
  deployment this summary describes
- [`calibrate.sh` guide](https://github.com/llm-d/llm-d/blob/main/guides/recipes/router/calibration/README.md) —
  measuring `peakPrefillThroughput` (§1)
- [NIXL connector notes](https://github.com/llm-d/llm-d/blob/main/docs/operations/disaggregation/vllm.md) — how
  the KV cache moves from prefill to decode
