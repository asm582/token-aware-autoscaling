# Staged autoscaling experiment — 2026-08-20

Upstream llm-d `pd-disaggregation` guide, Qwen3-32B, prefill/decode TP=2, namespace `pd-test`
(H100-80GB), token-aware ScaledObjects from `TOKEN-AWARE-AUTOSCALING-SUMMARY.md` §4 with
**`peakPrefillThroughput` (V_P) = 2696 tok/s** — measured on this stack, full P/D path — min 1 /
max 4 per role. Deployed with `--model-cache` (shared RWX weight cache).

Workload: [`workloads/pd-autoscaling-ramp.yaml`](../../workloads/pd-autoscaling-ramp.yaml) (copy
preserved here as `workload-used.yaml`) — 7 stages, ISL 2048 / OSL 2048, 3420 s of load. Run via
`run-benchmark.sh` (run-only; the stack was not redeployed). Autoscaling **armed**, no paused
baseline arm.

## Result: both triggers fired, the fleet scaled 4 → 14 GPUs and back

2482 requests succeeded, 8 failed (0.32%). `llm_d_epp_disagg_decision_total` rose by **+2490** —
every request was served through the P/D split.

| stage | rate | achieved | TTFT mean | TTFT p90 | ITL mean | in tok/s | out tok/s | replicas p\|d |
|---|---|---|---|---|---|---|---|---|
| 0 | 0.25 | 0.251 | 1.089 | 1.604 | 15.2 ms | 467 | 466 | 1\|1 → 2\|1 |
| 1 | 0.50 | 0.507 | 1.189 | 1.851 | 17.4 ms | 934 | 894 | 2\|1 → **1\|1** → 2\|1 |
| 2 | 0.75 | 0.751 | 1.235 | 1.839 | 19.6 ms | 1415 | 1343 | 2\|1 → 3\|1 |
| 3 | 1.00 | 1.000 | 1.376 | 2.167 | 20.6 ms | 1924 | 1835 | 3\|1 → 4\|1 |
| 4 | 1.50 | 1.501 | **26.593** | **69.191** | 31.9 ms | 2811 | 2703 | 4\|1 (ramp-up transient) |
| 5 | 0.75 | 0.752 | 1.014 | 1.268 | 16.2 ms | 1345 | 1259 | 4\|1 → 4\|2 → 4\|3 |
| 6 | 0.25 | 0.251 | 0.939 | 1.051 | 15.0 ms | 494 | 465 | 4\|3 → 2\|2 → 1\|1 |

Peak trigger values: **prefill 131.7** (vs threshold 1.5 — it wanted `ceil(131.7/1.5)` = 88
replicas, clamped to `--max 4`) and **decode 1.891** (vs 0.8, crossed on multiple samples).
Aggregate: TTFT mean 10.42 s / p90 44.56 s / p99 83.79 s (all dominated by stage 4), ITL mean
23.3 ms, 1314 output tok/s, 0.672 req/s.

## Reading the analysis charts through V_P

The three charts in this directory (`throughput_vs_qps.png`, `latency_vs_qps.png`,
`throughput_vs_latency.png`) are the same data the calibration constant predicts in advance.

**V_P sets the saturation QPS, and the charts land where it says.** With one prefill replica and
ISL 2048, sustained-throughput saturation is at

```
QPS_sat = V_P / ISL = 2696 / 2048 ≈ 1.32 req/s per prefill replica
```

Every stage at or below that offered rate holds TTFT near ~1 s (stages 0–3: 1.09 → 1.38 s). The
first stage *above* it — stage 4 at rate 1.5 — is where TTFT detonates (26.6 s mean, 69 s p90).
On `latency_vs_qps.png` this is the TTFT knee between QPS 1.0 and 1.5; on `throughput_vs_qps.png`
it is where input tok/s stops tracking offered load and flattens toward the per-replica ceiling
(2811 tok/s ≈ V_P). The knee is not a surprise in the data — it is `V_P / ISL` drawn as a curve.

**The cliff is a prefill-queue cliff, which is exactly what the trigger watches.** ITL barely
moves across the whole run (15 → 32 ms), so decode is never the bottleneck — the latency blowup
is entirely time-to-first-token, i.e. prefill backlog. That is the quantity
`inflight_tokens / V_P` measures, and it is why the prefill metric spiked to 131.7 in stage 4
while the decode metric only reached 1.9.

**Caveat: stage 4's 26.6 s is a stage-average inflated by the scale-up transient, not a
steady-state number.** At a settled 4 prefill replicas, capacity is `4 × 2696 = 10784 tok/s`,
far above the 3072 tok/s that rate 1.5 offers — steady state would be comfortable. The backlog
built while the fleet was still climbing 1→4 and new pods were loading weights (see
`autoscaling-timeline.csv`). So the chart's stage-4 point reports *transition cost*, and the
right operating conclusion is: keep offered load per replica below `V_P / ISL`, and pre-provision
enough headroom that a rate step does not outrun the ~scale-up + load lag.

## Finding 1: the prefill trigger still flaps on the instantaneous gauge

```
t=97    1 → 2   metric above target
t=402   2 → 1   "All metrics below target"   <-- during rising load (stage 1, rate 0.5)
t=579   1 → 2   metric above target          <-- 177 s after removing the pod
```

`llm_d_epp_inflight_tokens` is an **instantaneous** gauge and the query applies no time
averaging, so at `pollingInterval: 15` many samples read 0 — a prefill leg occupies only ~2 s of
a request's life. As in the 2026-08-18 run, this retired a replica *while load was still rising*,
then re-requested it minutes later. **Fix:** wrap the numerator in `avg_over_time(...[1m])`.

## Finding 2: the model-weight cache removed the "pods not serving" tax, but scale lag remains

This run used `--model-cache`, so new pods no longer download their own ~65 GB copy — a direct
improvement over 2026-08-18, where 35% of samples had a replica desired-but-not-Ready. Pods still
take time to become Ready (weights load from the shared PVC into GPU memory), so a rate step that
outruns the `scaleUp` cadence still shows up as a TTFT transient (stage 4 above). The cache
shortens the lag; it does not remove the need for headroom.

## Finding 3: scale-down still clipped in-flight requests (8 failures, 0.32%)

All 8 failures fall in stages 5–6, the scale-down phase (7 in stage 5, 1 in stage 6) — not in the
saturated stage 4, which had zero failures. This matches 2026-08-18: retiring a decode replica
with streaming requests still attached truncates them. **Fix:** a
`terminationGracePeriodSeconds` long enough to drain a generation (OSL 2048 at ~20 ms/token ≈
40 s) and/or `preStop` draining.

## What this does NOT establish

There is **no paused baseline arm**, so this shows the autoscaler *reacting* to changing traffic —
not that autoscaling beat a fixed fleet. To measure the benefit, replay this identical profile
with `--pause-autoscaling` and compare per-stage TTFT/throughput.

## Artifacts

| file | what |
|---|---|
| `throughput_vs_qps.png`, `latency_vs_qps.png`, `throughput_vs_latency.png` | analysis charts (the V_P discussion above) |
| `autoscaling-timeline.csv` | replicas + both trigger values every ~15 s (257 samples) |
| `summary_lifecycle_metrics.json` | aggregate latency/throughput |
| `stage_N_lifecycle_metrics.json` | per-stage metrics (the table above) |
| `workload-used.yaml` | exact workload for this run |
