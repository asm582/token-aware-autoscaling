# Decode-heavy staged autoscaling experiment — 2026-08-20

Upstream llm-d `pd-disaggregation` guide, Qwen3-32B, prefill/decode TP=2, namespace `pd-test`
(H100-80GB), token-aware ScaledObjects from `TOKEN-AWARE-AUTOSCALING-SUMMARY.md` §4 with
**`peakPrefillThroughput` (V_P) = 2696 tok/s**, min 1 / max 4 per role. Deployed with
`--model-cache`.

Workload: [`workloads/pd-autoscaling-ramp-decode-heavy.yaml`](../../workloads/pd-autoscaling-ramp-decode-heavy.yaml)
(copy preserved here as `workload-used.yaml`) — the mirror image of the prefill-heavy sibling:
**ISL 256 / OSL 8192 (1:32)**, 7 stages, rates 0.10 → 0.80 → 0.10. Each request does trivial
prefill and a long generation, so it drives the *decode* trigger (KV-cache occupancy, threshold
0.8) while leaving the prefill token-backlog trigger (`inflight_tokens / V_P`) nearly idle. Run via
`run-benchmark.sh` (run-only). Autoscaling **armed**, no paused baseline arm.

## Result: decode scaled 1 → 4 (all Ready); prefill stayed at 1

1181 requests succeeded, 286 failed. `llm_d_epp_disagg_decision_total` rose by **+1467** — exactly
the 1181 + 286 total, so **every request was served through the P/D split** (even the ones that
later timed out had already been dispatched and disaggregated — the failures are generation
slowness, not routing rejection).

| stage | rate | achieved | TTFT mean | TTFT p90 | ITL mean | req latency | in tok/s | out tok/s | ok / fail | decode rep |
|---|---|---|---|---|---|---|---|---|---|---|
| 0 | 0.10 | 0.103 | 0.209 | 0.231 | 17.2 ms | 133.9 s | 18 | 533 | 30 / 0 | 1 |
| 1 | 0.20 | 0.200 | 0.226 | 0.253 | 20.8 ms | 166.5 s | 37 | 1139 | 84 / 0 | 1 |
| 2 | 0.35 | 0.352 | **2.571** | 5.417 | 27.9 ms | 227.0 s | 50 | 1516 | 170 / **40** | 1 (building) |
| 3 | 0.55 | 0.551 | 0.236 | 0.268 | 24.8 ms | 196.1 s | 109 | **3196** | 330 / 0 | 1 → 2 |
| 4 | 0.80 | 0.800 | **2.893** | 2.168 | 28.3 ms | 228.4 s | 97 | 2926 | 354 / **222** | 2 |
| 5 | 0.35 | 0.351 | 0.205 | 0.226 | 17.4 ms | 137.8 s | 61 | 1778 | 131 / 16 | 2 → 3 |
| 6 | 0.10 | 0.103 | 0.208 | 0.229 | 16.6 ms | 134.2 s | 21 | 647 | 82 / 8 | 3 → **4** → drain |

Aggregate: TTFT mean 1.36 s / p90 0.29 s / p99 35.7 s (the mean sits *above* p90 — a small heavy
tail of near-timeout admissions drags it up), ITL mean 24.4 ms, request latency mean 195.8 s,
1749 output tok/s, 0.228 req/s. Peak decode trigger value **2.976** (vs threshold 0.8 — it wanted
`ceil(2.976/0.8)` = 4 replicas, exactly `--max`). Prefill trigger stayed near 0 with brief
instantaneous-gauge spikes (transient max 8.26, never sustained).

## Fleet trajectory, and the decode-specific twist: occupancy lags load by a full generation

Decode climbed **1 → 2 → 3 → 4** and all four became Ready (2×1 prefill + 4×2 decode = 10 GPUs,
inside the 14-GPU fleet, so nothing sat Pending — unlike prefill-heavy). The decode trigger rose
monotonically stage over stage: **0.20 → 0.31 → 0.60 → 1.00 → 1.65 → 2.00 → 2.98**, crossing 0.8
during stage 3 and cresting near the end.

The striking part is *when* the fleet peaked. Decode reached its maximum of **4 replicas during
stage 6 — the rate-0.10 recovery stage** — not during the rate-0.80 peak. This is not a bug: KV
occupancy is set by *in-flight* requests, and each OSL-8192 generation lives ~230 s. The requests
sent at the rate-0.80 peak (stage 4, sent over 1920–2640 s) were still generating well into the
recovery window, so occupancy kept climbing after the send rate had already dropped. The autoscaler
correctly kept scaling **up** while offered load was falling, because the memory pressure it watches
peaks a full generation-time behind the traffic. Prefill, whose leg is ~0.1 s here (ISL 256), shows
none of this lag — it flapped 1↔2 on the instantaneous gauge and settled back to 1.

## Reading the analysis charts

The three charts are plotted against **requested rate**, and rates 0.10 and 0.35 each appear twice
(up-ramp stage 0/2, down-ramp stage 6/5). Those repeated x-values produce the vertical drops /
loops — they are **hysteresis**: the same offered rate served by a cold fleet vs a warm one.

### `throughput_vs_qps.png` — the ceiling is decode generation, and it rolls over
Output tok/s climbs 533 → 1139 → **3196 at rate 0.55**, then *declines* to 2926 at rate 0.80. That
roll-over is the decode fleet hitting its aggregate generation ceiling: past ~0.55 req/s, offering
more traffic buys no more output tokens — it only builds KV backlog (and failures, below). Input
tok/s stays tiny (18 → 109): ISL 256 means prefill moves ~3% of the token volume, the exact inverse
of the prefill-heavy run.

### `latency_vs_qps.png` — the work is all in generation, and the TTFT "V" is hysteresis
- **TTFT is near-flat at ~0.2 s** across the low/warm stages — ISL 256 makes prefill trivial, so
  time-to-first-token is essentially the network hop. Nothing like the prefill-heavy 3 s floor.
- **ITL spans 17–28 ms** and *this* is where load shows up: request latency runs 130–230 s because
  the 8192-token generation, not the prefill, is the whole cost. Norm-time-per-output-token peaks at
  rate 0.55 for the same reason the throughput curve rolls over there.
- **The TTFT "V" (2.57 s at 0.35 → 0.24 s at 0.55 → 2.89 s at 0.80) is the hysteresis loop.** Rate
  0.35 caught decode **cold** (1 replica, occupancy building) so new requests waited for KV
  admission — that admission delay lands in TTFT. Rate 0.55 was served **warm** (fleet had scaled)
  so TTFT fell back to floor. Rate 0.80 **saturated** even the scaling fleet. Note stage 4's TTFT
  mean (2.89 s) sits above its p90 (2.17 s): most requests were admitted fast, a tail waited ~77 s
  (p99), i.e. the same requests that eventually timed out.

### `throughput_vs_latency.png` — the cost curve
Output-vs-ITL is the usable envelope: the bottom-left cluster (stages 0,1,5,6) sits at ~17 ms ITL
and low throughput; the knee at ~3200 tok/s output is where ITL and norm-time climb steeply for no
extra tokens. This is a decode-bound saturation elbow — the counterpart to prefill-heavy's
TTFT-bound one.

## Finding 1: the decode trigger is correctly targeted, and it leads the failures it should

The decode metric climbed monotonically to 2.976 and asked for exactly `--max` 4 replicas; the
prefill metric stayed idle. Trigger isolation is clean and is the mirror image of the prefill-heavy
run (there prefill pegged at 4 and decode was the quiet one). The one operational subtlety, above,
is that KV occupancy **lags** offered load by a generation-time, so the fleet peaks after the
traffic does — worth knowing when tuning `scaleDown` stabilization so it does not retire replicas
that the still-draining backlog still needs.

## Finding 2: decode saturation surfaces as *failed* requests, not queued latency

This is the sharp asymmetry versus prefill-heavy (which had **1** failure). Here **262 of 286
failures** are `timeouterror`, every one pinned at request latency **300.0–301.0 s** — the
inference-perf client's per-request timeout. They occur exactly in the two under-provisioned stages:
40 in stage 2 (rate 0.35, decode still cold at 1 replica) and 222 in stage 4 (rate 0.80, offered
load past even the scaling fleet). The mechanism: a request needs `OSL × ITL ≈ 8192 × 28 ms ≈
229 s` of pure generation, so once decode is contended the total slips past the 300 s ceiling and
the client aborts a request that was still generating.

The reason this looks so different from prefill-heavy is structural. Prefill backpressure is a
**queue** — requests wait, TTFT rises, but they complete (hence 137 s TTFT with ~0 failures there).
Decode backpressure is **memory/stream** — there is no equivalent queue; a long generation under KV
contention simply runs out of wall-clock. So the same "over-drove the cap on purpose" experiment
that produced high-but-successful latency in prefill-heavy produces timeouts here. **Fix / caveat:**
the harness ran with `request_timeout: null` (library default 300 s); long-OSL workloads should set
`request_timeout` well above `OSL × steady-ITL` (e.g. 600 s) so decode saturation is measured as
latency rather than counted as failure.

## Finding 3: scale-down still clipped in-flight streams (24 failures)

The remaining 24 failures (16 in stage 5, 8 in stage 6) are `clientpayloaderror` — "Response
payload is not completed" — at partial request latencies (40–160 s). These fall entirely in the
scale-down phase, where decode retires 4 → 3 → 2 → 1 and cuts the long streaming generations still
attached to a departing pod. It is the same defect the balanced run flagged (Finding 3 there), and
it bites hardest here because decode-heavy streams are the longest in the suite. **Fix:** a
`terminationGracePeriodSeconds` long enough to drain a generation (OSL 8192 at ~24 ms/token ≈
200 s) and/or `preStop` draining.

## What this does NOT establish

No paused baseline arm, so this shows the autoscaler *reacting*, not a head-to-head vs a fixed
fleet. The fairest within-run comparison is the hysteresis at rate 0.35 — warm (stage 5, TTFT
0.21 s, 16 clip-failures) vs cold (stage 2, TTFT 2.57 s, 40 timeout-failures) — which favors the
warm fleet on both latency and success rate. To measure the benefit directly, replay with
`--pause-autoscaling` and compare per stage.

## Artifacts

| file | what |
|---|---|
| `throughput_vs_qps.png`, `latency_vs_qps.png`, `throughput_vs_latency.png` | analysis charts (discussed above) |
| `autoscaling-timeline.csv` | replicas + both trigger values every ~15 s (345 samples, full run incl. drain) |
| `summary_lifecycle_metrics.json` | aggregate latency/throughput |
| `stage_N_lifecycle_metrics.json` | per-stage metrics (the table above) |
| `workload-used.yaml` | exact decode-heavy workload for this run |
