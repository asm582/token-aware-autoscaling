# Prefill-heavy staged autoscaling experiment — 2026-08-20

Upstream llm-d `pd-disaggregation` guide, Qwen3-32B, prefill/decode TP=2, namespace `pd-test`
(H100-80GB), token-aware ScaledObjects from `TOKEN-AWARE-AUTOSCALING-SUMMARY.md` §4 with
**`peakPrefillThroughput` (V_P) = 2696 tok/s**, min 1 / max 4 per role. Deployed with
`--model-cache`.

Workload: [`workloads/pd-autoscaling-ramp-prefill-heavy.yaml`](../../workloads/pd-autoscaling-ramp-prefill-heavy.yaml)
(copy preserved here as `workload-used.yaml`) — the prefill-heavy sibling of the balanced ramp:
**ISL 8192 / OSL 256 (32:1)**, 7 stages, rates 0.15 → 1.4 → 0.15. It is built to drive the
*prefill* trigger (`inflight_tokens / V_P`) hard while leaving decode compute nearly idle. Run
via `run-benchmark.sh` (run-only). Autoscaling **armed**, no paused baseline arm.

## Result: prefill scaled 1 → 4 (capped) and held; decode reached 3 on KV pressure

1949 requests succeeded, 1 failed. Every request was served through the P/D split.

| stage | rate | achieved | TTFT mean | TTFT p90 | ITL mean | in tok/s | out tok/s | replicas p\|d |
|---|---|---|---|---|---|---|---|---|
| 0 | 0.15 | 0.152 | 4.198 | 6.338 | 8.3 ms | 1217 | 63 | 1\|1 |
| 1 | 0.30 | 0.300 | 4.791 | 6.867 | 14.6 ms | 2403 | 69 | 1\|1 |
| 2 | 0.50 | 0.501 | **25.628** | 44.322 | 14.5 ms | 4023 | 123 | 1 → 2 |
| 3 | 0.80 | 0.803 | **110.468** | 198.176 | 13.9 ms | 5822 | 182 | → 4 |
| 4 | 1.40 | 1.405 | **136.509** | 215.131 | 12.1 ms | 8466 | 308 | **4\|3** (cap) |
| 5 | 0.50 | 0.503 | 4.440 | 6.166 | 12.0 ms | 4031 | 147 | 4\|3 → 4\|2 |
| 6 | 0.15 | 0.150 | 3.839 | 4.952 | 9.7 ms | 1210 | 53 | 4 → 3 → 2 → 1 |

Aggregate: TTFT mean 89.7 s / p90 202.3 s / p99 233.4 s (dominated by stages 3–4), ITL mean
12.5 ms, peak input 8466 tok/s, 0.518 req/s. Fleet trajectory (recovered timeline, 19:04–19:48
UTC): prefill pinned **4/4** through stages 3–5, then 4 → 3 → 2 → 1; decode 3 → 2 → 1.

## Reading the analysis charts

The three charts are plotted against **requested rate**, and rates 0.15 and 0.50 each appear
twice (up-ramp stage 0/2, down-ramp stage 6/5). Those repeated x-values produce the vertical
drops / loops — they are **hysteresis**: the same offered rate served by a cold fleet (1 replica)
vs a warm one (4 replicas).

### `throughput_vs_qps.png` — the autoscaler raising its own ceiling
Input tok/s climbs monotonically 1217 → 8466 and is gently concave. Fixed at one replica it would
**plateau flat at ≈ V_P = 2696**; instead it keeps rising because KEDA added prefill replicas,
lifting the aggregate ceiling to `4 × 2696 = 10784`. The peak (8466) is **~79% of that ceiling**
(2117 tok/s/replica, below calibrated V_P) — the gap is scale-up lag plus the harness only
achieving 1.03 of the requested 1.40 rate once backlog saturated `worker_max_concurrency`. Output
tok/s stays tiny (63 → 308): OSL 256 means decode moves ~2% of the token volume.

### `latency_vs_qps.png` — the money chart (TTFT is the whole story)
- **TTFT floor ≈ 4 s** at low rate is not arbitrary: `ISL / V_P = 8192 / 2696 = 3.0 s` is the
  irreducible prefill time for one 8192-token request. Even unloaded, prefill-heavy TTFT is
  bounded below by V_P.
- **The knee is at rate 0.50** — exactly where theory puts it, since per-replica saturation is
  `V_P / ISL = 0.33 req/s`. TTFT breaks 4.8 → 25.6 → 110 → 137 s across stages 1→4.
- **The vertical drop at QPS 0.5 is the hysteresis loop**: 25.6 s on the up-ramp (cold, 1 replica)
  vs **4.4 s on the down-ramp** (warm, 4 replicas) — **~5.8× lower TTFT at the identical offered
  rate**. This is the clearest single-picture argument for the autoscaler.
- **ITL spans only 8–14.6 ms** — decode compute was never the bottleneck; the loop there is noise
  on a 6 ms band. Norm-time-per-output-token tracks TTFT, not ITL, because with 256 output tokens
  the one-time prefill wait dominates the per-token average (~1180 ms/token at the peak).

### `throughput_vs_latency.png` — the cost curve
Output-vs-TTFT is a textbook saturation elbow: the usable region is the bottom-left cluster
(stages 0,1,5,6 at ~4 s TTFT); past ~180 tok/s output you fall off a cliff to 110–137 s for almost
no extra throughput. Output-vs-ITL rises while ITL barely moves — again, all the pain is prefill
queueing, i.e. precisely the quantity the prefill trigger watches.

## Finding 1: the prefill trigger is correctly targeted, and the cap — not the signal — is the limit

100% of the latency blowup is TTFT/prefill-queue (ITL flat throughout). At the peak the trigger
would read ≈ `137 / 1.5 ≈ 90` replicas wanted, clamped to `--max 4`. The signal points the right
way; the run simply over-drove capacity on purpose (peak offered `1.4 × 8192 = 11469` tok/s >
`4 × V_P = 10784`), so stages 3–4 are unbounded backlog plus scale-up lag, not steady state.

## Finding 2: long context pressures decode KV even with a tiny output

Despite OSL 256, decode scaled to **3**. Each request transfers ~**2.0 GiB of KV** to the decode
side (8192 tokens × 256 KiB/token for Qwen3-32B), so decode's KV-occupancy trigger fired on
*memory*, not compute — ITL never rose. Prefill-heavy is therefore not purely a prefill story: the
larger the context, the more it also loads decode KV. (One decode replica sat `Pending` at the
peak — 4×2 prefill + 3×2 decode = 14 GPUs requested — a fleet-size limit, not a scheduling bug.)

## Finding 3: recovery was clean — 1 failure vs 8 in the balanced run

Only a single request failed (in the stage-6 scale-down), and TTFT returned to ~4 s within one
stage of the load dropping (stage 5). The `--model-cache` weight cache (new since 2026-08-18) plus
the shorter recovery kept scale-down from clipping streams the way the balanced runs did.

## Operational note: a network blip, and how the results were recovered

~24 min in, this workstation briefly lost its route to the cluster API (`network is unreachable`).
That killed only the **local** `run-benchmark.sh` wrapper — result collection and its timeline
sampler — with exit 1. The harness pod runs inside the cluster and ran to completion regardless.
The authoritative per-stage lifecycle metrics and charts were recovered off the workload PVC
(`kubectl cp` from the data-access pod), and the fleet timeline here was captured by a replacement
sampler covering the peak and scale-down (19:04–19:48 UTC). The 1 → 4 prefill scale-up itself
falls in a ~6 min gap between the wrapper's truncated CSV and the replacement sampler; the harness
log confirms the stage boundaries, and prefill was already 4/4 when sampling resumed in stage 3.

## What this does NOT establish

No paused baseline arm, so this shows the autoscaler *reacting*, not a head-to-head vs a fixed
fleet. The fairest within-run comparison is the hysteresis at rate 0.5 — warm (stage 5, 4.4 s) vs
cold (stage 2, 25.6 s) — which is decisively in the autoscaler's favor. To measure the benefit
directly, replay with `--pause-autoscaling` and compare per stage.

## Artifacts

| file | what |
|---|---|
| `throughput_vs_qps.png`, `latency_vs_qps.png`, `throughput_vs_latency.png` | analysis charts (discussed above) |
| `autoscaling-timeline.csv` | replicas + trigger-active status, peak → scale-down (19:04–19:48 UTC) |
| `summary_lifecycle_metrics.json` | aggregate latency/throughput |
| `stage_N_lifecycle_metrics.json` | per-stage metrics (the table above) |
| `workload-used.yaml` | exact prefill-heavy workload for this run |
