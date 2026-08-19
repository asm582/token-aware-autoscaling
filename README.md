# Token-aware autoscaling on llm-d P/D

Five scripts that deploy the upstream [llm-d P/D disaggregation
guide](https://github.com/llm-d/llm-d/tree/main/guides/pd-disaggregation), measure the one
hardware-specific constant the design needs, arm token-aware KEDA autoscaling on both roles,
prove the trigger metrics actually reach KEDA, and benchmark the result — plus a staged
workload that exercises the autoscaler end to end.

The design is described in [TOKEN-AWARE-AUTOSCALING-SUMMARY.md](TOKEN-AWARE-AUTOSCALING-SUMMARY.md).
Read that first — it explains *why* prefill divides tokens by a token rate and decode does
not, which is the part that makes the thresholds mean something.

Everything here was run end to end on OpenShift 4.x / k8s 1.32, H100-80GB, Qwen3-32B at
prefill 1×TP2 + decode 1×TP2.

## Prerequisites

Asserted by the scripts, never installed by them — each fails with the exact command to run
if something is missing.

| Requirement | Why | Check |
|---|---|---|
| `kubectl`, `helm`, `kustomize`, `python3` (+ `pyyaml`), `git`, `envsubst` | client tooling | `deploy-pd-guide.sh` stage 0 |
| GAIE CRDs, `InferencePool` **v1** | the router chart renders an InferencePool | `kubectl get crd inferencepools.inference.networking.k8s.io` |
| Kubernetes **≥ 1.29** | the decode routing sidecar is a native sidecar (initContainer + `restartPolicy: Always`) | `kubectl version` |
| GPUs free **on one node per pod** | tensor parallelism is intra-node: a TP=N pod needs N free GPUs on a single node | `deploy-pd-guide.sh` bin-packs and reports |
| KEDA / Custom Metrics Autoscaler | runs the ScaledObjects | `kubectl get crd scaledobjects.keda.sh` |
| Prometheus user-workload monitoring + Thanos Querier | where the trigger queries run | `kubectl get pods -n openshift-user-workload-monitoring` |
| Permission to create a `ClusterRoleBinding` | KEDA→Thanos needs a `cluster-monitoring-view` binding | `kubectl auth can-i create clusterrolebinding` |
| `llmdbenchmark` CLI + the `llm-d-benchmark` checkout | only for `run-benchmark.sh`; also supplies the `config/` and `workload/` trees | `llmdbenchmark --version` |
| `oc` (OpenShift CLI) | `launch-scaledobjects.sh` uses it throughout; the Thanos/`service-ca` auth path is OpenShift-specific | `oc whoami` |

Install the benchmark CLI with:

```bash
curl -sSL https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/install.sh | bash
cd llm-d-benchmark && source .venv/bin/activate
```

`LLMD_DIR` controls where the llm-d checkout lives. By default the scripts use `./llm-d`,
falling back to `../llm-d` if that already exists, and cloning if neither does. Point it
anywhere: `LLMD_DIR=/path/to/llm-d ./deploy-pd-guide.sh`.

## Run order

```bash
# 1. deploy the upstream guide (clean namespace -> router -> model servers -> verify)
./deploy-pd-guide.sh                        # ~6 min; --render-only needs no cluster

# 2. enable monitoring — REQUIRED before step 4, see "Monitoring" below

# 3. measure peakPrefillThroughput on YOUR hardware, and apply it to the EPP
./calibrate-peak-prefill.sh --decompose     # measure + split prefill vs KV-transfer cost
./calibrate-peak-prefill.sh --apply         # measure, patch the EPP, verify it took

# 4. arm the two ScaledObjects
./launch-scaledobjects.sh --vp <measured> --max 4

# 5. prove the metrics reach KEDA
./test-metric-flow.sh                       # read-only health check
./test-metric-flow.sh --probe 180           # drive load, prove the values MOVE (may scale)

# 6. benchmark, or run the staged autoscaling experiment
./run-benchmark.sh                                              # guide's latency profile
./run-benchmark.sh --workload-file workloads/pd-autoscaling-ramp.yaml   # 57-min staged ramp
./run-benchmark.sh --workload-file workloads/pd-autoscaling-ramp.yaml --pause-autoscaling
                                                                # same load, fixed fleet (baseline)
```

Teardown: `./deploy-pd-guide.sh --teardown` (releases the GPUs) and
`./launch-scaledobjects.sh --delete`.

## The scripts

### `deploy-pd-guide.sh`
Deploys the guide by its own path — `helm install llm-d-router-standalone` plus a kustomize
overlay — not through llm-d-benchmark. Eight stage-gated stages, each asserting a
postcondition rather than trusting an exit code. Verifies disaggregation actually happened
by finding the decode pod's IP in the prefill pod's access log, because a P/D stack that
quietly serves everything from decode passes a plain `curl` check and is still broken.

Modes: `--render-only` (no cluster), `--dry-run`, `--verify-only`, `--teardown`,
`--namespace`, `--ref`. Topology via `MODEL PREFILL_REPLICAS PREFILL_TP DECODE_REPLICAS DECODE_TP`.

### `calibrate-peak-prefill.sh`
Wraps upstream `guides/recipes/router/calibration/calibrate.sh` unmodified, adding the
guards whose absence makes its output silently wrong: `CHUNK_SIZE` verified against the
effective `--max-num-batched-tokens` read from each prefill pod's own startup log; an idle
gate before and after (queue wait inside TTFT understates throughput); repeats with the
run-to-run spread reported. `--decompose` additionally measures the prefill pod directly
via upstream's `VLLM_ENDPOINT` override, splitting prefill compute from the P/D KV hop.
`--apply` rewrites `peakPrefillThroughput` (which lives inside a YAML string no `--set` can
reach), upgrades, restarts the EPP, and re-reads the live ConfigMap to confirm.

### `launch-scaledobjects.sh`
Creates the KEDA auth chain (metrics-reader SA + `cluster-monitoring-view` binding + token
Secret + `TriggerAuthentication`) and the two ScaledObjects from the summary's §4.
Deployment and InferencePool names are **discovered**, not hardcoded. `--decode-signal`
selects the summary's occupancy trigger (default) or the refused-admission variant; the
default threshold follows the signal because they are different units.

### `test-metric-flow.sh`
Reads the queries **out of the live ScaledObjects** rather than keeping its own copy, and
queries Thanos with the **metrics-reader ServiceAccount's own token** — the exact credential
KEDA uses. Both choices are deliberate: a test with its own copy of the PromQL passes while
KEDA runs something else, and a test using your `oc whoami -t` succeeds where the SA may
not. `--probe N` drives load and writes a timeline CSV proving the values move.

### `run-benchmark.sh`
Benchmarks the already-deployed stack with `llm-d-benchmark`, **run-only** — it never calls
`standup`/`teardown`, so it cannot redeploy or disturb the stack. Follows the guide's
documented path (`--endpoint-url` + `--gateway-class epponly`; without the latter the CLI
re-renders against the scenario's default topology and measures something else). Results land
in `./benchmark-results/<timestamp>/` with a `latest` symlink — never `~/data`.

Adds around the CLI: preflight (all pods Ready, endpoint answers `/v1/models`, model read from
the live Deployment); a replica + trigger-value timeline in `autoscaling-timeline.csv`;
`--pause-autoscaling` for a fixed-topology baseline, restored on exit even on Ctrl-C; EPP
counter snapshots asserting `llm_d_epp_disagg_decision_total` actually rose; and it raises
`--wait-timeout` to cover a long profile's own duration, since a harness killed mid-ramp
returns partial results that look complete.

### `workloads/` and `experiments/`
`workloads/pd-autoscaling-ramp.yaml` is a 7-stage ramp (rate 0.25 → 1.5 → 0.25, ISL/OSL 2048,
3420 s) built to cross both thresholds and then recover, so one run shows scale-up, the cap,
and scale-down. Traffic changes *within* a run via `load.stages` — the standard upstream
pattern. Pass it with `--workload-file`.

`experiments/` holds the report and the small artifacts from runs worth keeping. Start with
[`experiments/2026-08-18-staged-ramp/EXPERIMENT-REPORT.md`](experiments/2026-08-18-staged-ramp/EXPERIMENT-REPORT.md):
both triggers fired (prefill peaked 10.76 vs threshold 1.5, decode 0.994 vs 0.8), the fleet
went 4 → 12 GPUs and back — and it documents three defects found in the process.

## Monitoring is required, and is not part of the deploy

The guide treats monitoring as optional, but the ScaledObjects cannot work without it. The
EPP metrics endpoint answers **401** until it is enabled.

```bash
helm upgrade pd-disaggregation oci://ghcr.io/llm-d/charts/llm-d-router-standalone --version v0 \
  -f ${LLMD_DIR}/guides/recipes/router/base.values.yaml \
  -f ${LLMD_DIR}/guides/pd-disaggregation/router/pd-disaggregation.values.yaml \
  -f ${LLMD_DIR}/guides/recipes/router/features/monitoring.values.yaml \
  -n <namespace> --wait
kubectl apply -n <namespace> -k ${LLMD_DIR}/guides/recipes/modelserver/components/monitoring-pd
```

**If you have already run `calibrate-peak-prefill.sh --apply`**, add its override as a final
`-f` or this upgrade silently reverts `peakPrefillThroughput` to the guide's shipped value:

```bash
  -f .pd-guide-workspace/calibration/router-calibrated.values.yaml \
```

That file is generated (and git-ignored), so it exists only after a calibration run — on a
fresh clone there is nothing to add yet.

## Things that will bite you

**`peakPrefillThroughput` depends on which path you measure, and upstream's two published
values are not comparable.** Measured here: **15965 tok/s** against the prefill pod directly
(matching upstream's configuration-matrix value of 15928 to 0.2%), but **2619 tok/s** through
the full P/D path — because 84% of TTFT was the NIXL KV transfer, moving 2.00 GiB per
8192-token request (256 KiB/token for Qwen3-32B) at ~6.6 Gbps over TCP. Confirmed
bandwidth-bound by re-measuring at chunk 2048: predicted 0.65s, measured 0.666s. The
`pd-disaggregation` guide ships **33821**, measured on H200/gpt-oss-120b with a fast fabric.
Measure your own, and know which number you are holding. If RDMA works on your fabric, the
P/D figure moves a long way up.

**The denominator sets the trigger's aggressiveness, not just its units.** At V_P=2665 an
observed 26.9s backlog asks for `ceil(26.9/1.5)` = 18 replicas; the same load at 15928 reads
4.5s and asks for 3.

**`llm_d_epp_inflight_tokens` is registered lazily on the first dispatched request.** A
freshly restarted EPP has no such series, and the query's `or vector(0)` renders that as a
confident zero — indistinguishable from "no backlog". Send traffic before trusting the
trigger. `test-metric-flow.sh` reports *absent* separately from *zero* for this reason.

**The prefill trigger flaps under steady load.** `llm_d_epp_inflight_tokens` is an
*instantaneous* gauge and the query does no time-averaging, so at `pollingInterval: 15` most
samples read exactly 0 — a prefill leg occupies only ~2 s of a request's life. In the staged
run this retired a replica *while load was rising* ("All metrics below target"), then asked for
three replicas 89 s later. Consider `avg_over_time(...[1m])` around the numerator. See the
experiment report.

**`maxReplicaCount: 10`** (the summary's value) is 10 × TP GPUs per role. At TP=2 that is 40
GPUs across both roles. Use `--max` to match your fleet.

**On OpenShift, Thanos `:9091` answers unauthenticated queries with 401 — and KEDA
suppresses that error and serves `fallback` replicas**, so a broken trigger looks healthy.
This is why the auth chain exists and why the verifier tests it with the SA's own token.

**The guide provisions no model PVC**, so every pod downloads its own copy of the weights
(~65 GB for Qwen3-32B) to node ephemeral storage through the `emptyDir` at `/.cache`.
`deploy-pd-guide.sh --model-cache` fixes this the way `llm-d-benchmark` does: a shared
ReadWriteMany PVC, populated once by a Job, mounted read-only into every prefill/decode
pod (`vllm serve` is pointed at the local path and paired with `--served-model-name` so
client requests are unaffected). It is opt-in and scoped to `$NAMESPACE` — a full clean
run (the default) still deletes the namespace and the PVC with it; pass `--skip-clean`
to reuse a populated cache across reruns.

## Upstream gaps found while testing this (unreported as of 2026-08-18, llm-d @ main)

1. **`kubectl apply -k modelserver/gpu/vllm/base` cannot work.** It renders
   `image: REPLACE_MODEL_SERVER_IMAGE`, which the API server rejects, yet the README prints
   it with `INFRA_PROVIDER=base` as the default. `base/kustomization.yaml` omits the image
   component deliberately and every sibling overlay (`coreweave`, `aws`, `gke`) adds one;
   there is no generic or OCP overlay. `deploy-pd-guide.sh` generates the missing overlay
   and re-proves the gap on every run, so it will tell you when upstream fixes it.
2. **`calibrate.sh` is tracked as mode `100644`** while its sibling
   `calibrate-min-cached-token-delta.sh` is `100755`, so the README's documented
   `./calibrate.sh` fails with "permission denied" on a fresh clone.
   `calibrate-peak-prefill.sh` invokes it via `bash`.
