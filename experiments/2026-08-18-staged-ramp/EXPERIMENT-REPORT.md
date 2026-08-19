# Staged autoscaling experiment — 2026-08-18

Upstream llm-d `pd-disaggregation` guide, Qwen3-32B, prefill/decode TP=2, namespace `pd-test`,
token-aware ScaledObjects from `TOKEN-AWARE-AUTOSCALING-SUMMARY.md` §4 with
`peakPrefillThroughput = 2665` (measured on this stack), min 1 / max 4 per role.

Workload: [`workloads/pd-autoscaling-ramp.yaml`](../../workloads/pd-autoscaling-ramp.yaml) (copy preserved here as `workload-used.yaml`) —
7 stages, ISL 2048 / OSL 2048, 3420 s of load. Run via `run-benchmark.sh` (run-only; the
stack was not redeployed). Autoscaling **armed**, no paused baseline arm.

## Result: both triggers fired, the fleet scaled 4 → 12 GPUs and back

2482 requests succeeded, 8 failed. `llm_d_epp_disagg_decision_total` rose by **+2490** for
+2490 requests — every request was served through the P/D split.

| stage | rate | TTFT mean | TTFT p90 | ITL mean | out tok/s | replicas p\|d |
|---|---|---|---|---|---|---|
| 0 | 0.25 | 0.831 | 0.875 | 16.4 ms | 423 | 1\|1 |
| 1 | 0.50 | 1.096 | 1.618 | 17.7 ms | 854 | 1\|1 |
| 2 | 0.75 | 1.512 | 2.695 | 19.0 ms | 1376 | 1\|1 → 2\|1 |
| 3 | 1.00 | 1.761 | 3.134 | 20.5 ms | 1803 | 2\|1 → **1\|1** → 3\|1 |
| 4 | 1.50 | 2.084 | 3.630 | 26.2 ms | 2692 | 3\|1 → 4\|1 |
| 5 | 0.75 | 2.022 | 4.744 | 18.0 ms | 1267 | 4\|1 → 4\|2 |
| 6 | 0.25 | 0.962 | 1.183 | 15.1 ms | 481 | 4\|2 → 3\|2 → 3\|1 → 2\|1 |

Peak trigger values: **prefill 10.76** (vs threshold 1.5 — it wanted `ceil(10.76/1.5)` = 8
replicas, clamped to 4) and **decode 0.994** (vs 0.8, crossed on 11 samples). Aggregate:
TTFT mean 1.73 s / p90 3.12 s / p99 7.53 s, ITL mean 21.4 ms, 1305 output tok/s, 0.672 req/s
against a stage-weighted target of ~0.78 — the fleet did not fully keep up even at the cap.

## Finding 1: the prefill trigger flaps, because it samples an instantaneous gauge

```
t=776   1 → 2   metric above target
t=1224  2 → 1   "All metrics below target"   <-- during rising load (stage 3, rate 1.0)
t=1313  1 → 3   metric above target          <-- 89 s after removing the pod
```

`llm_d_epp_inflight_tokens` is an **instantaneous** gauge and the query applies no time
averaging, so at `pollingInterval: 15` most samples read exactly **0** — a prefill leg only
occupies ~2 s of each request's life. Observed values were quantised on multiples of ~1.2
(0, 1.2, 2.35, 3.6, 4.8, 6.0, 7.2, 8.4, 9.6, 10.76): one leg ≈ 0.6 s of backlog, so the
trigger effectively asks *"were ≥3 prefill legs in flight at the instant I looked?"*

With `scaleDown.stabilizationWindowSeconds: 300`, a five-minute run of zeros is enough to
retire a replica **while load is increasing**. The pod retired at t=1224 had become Ready at
t≈1083 — it served roughly **2.5 minutes** after a ~5 minute, 65 GB weight download.

**Fix:** wrap the numerator in `avg_over_time(...[1m])` so the signal reflects prefill work
queued over the interval rather than at the sampling instant. That addresses the flap and the
wasted pod builds together.

## Finding 2: 35% of the run was spent paying for pods that were not serving

Of 239 timeline samples, **83 had a replica desired but not Ready** — the guide provisions no
model PVC, so every new pod downloads its own ~65 GB copy of the weights (~5 min). Combined
with `scaleUp` of 1 pod / 180 s, capacity consistently arrives late: the decode pod added at
t=2370 became Ready around t≈2600 and was retired at t≈2748, by which time load had already
fallen to rate 0.25.

**Fix:** a shared RWX PVC for the HF cache, so only the first pod downloads. This also makes
the ~5 min lag stop dominating the autoscaler's response time.

## Finding 3: scale-down killed in-flight requests (8 failures, 0.3%)

All 8 failures cluster in one ~10 s window, and one is explicit:

```
503 "The decode node is not ready. Please check that the vLLM service is running
     and the port configuration is correct."
```

The other 7 are `ClientPayloadError` / `TransferEncodingError` — truncated response streams.
Failure timestamps are relative to the first request (~45–60 s after the sampler's t=0), so
request-time t≈2690 maps to timeline t≈2735–2750 — the window in which decode went 2/2 → 1/1.
The scale-down removed a decode pod that still had streaming requests attached.

**Fix:** a `terminationGracePeriodSeconds` long enough to drain a generation (OSL 2048 at
~20 ms/token is ~40 s), and/or `preStop` draining, so retiring a decode replica does not cut
live streams.

## What this does NOT establish

There is **no paused baseline arm**, so this shows the autoscaler *reacting* to changing
traffic — not that autoscaling improved latency or throughput versus a fixed fleet. Throughput
did rise with the fleet (423 → 2692 tok/s), but rate rose at the same time, so the two are not
separable from this run alone. To measure the benefit, replay this identical profile with
`./run-benchmark.sh --workload-file workloads/pd-autoscaling-ramp.yaml --pause-autoscaling`
and compare per-stage TTFT/throughput.

Also note TTFT mean reached 1.5 s at stage 2 — the same magnitude as the trigger's 1.5 s
threshold, which the summary frames as *the share of the TTFT budget spent queueing*. By the
time this trigger fires, total TTFT is already well past that budget on this hardware.

## Artifacts

| file | what |
|---|---|
| `autoscaling-timeline.csv` | replicas + both trigger values every ~15 s (239 samples) |
| `*/results/*/summary_lifecycle_metrics.json` | aggregate latency/throughput |
| `*/results/*/stage_N_lifecycle_metrics.json` | per-stage metrics (the table above) |
| `*/results/*/per_request_lifecycle_metrics.json` | 2490 records incl. the 8 failures |
| `epp-metrics-{before,after}.txt` | EPP counters bracketing the run |
| `llmdbenchmark.log`, `run-benchmark.log` | full logs |
