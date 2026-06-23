# DeepEP-vs-Alternative — measured campaign data (calculator source of truth)

All numbers MEASURED on the efa-gda campaign: 2× p5en.48xlarge, AWS EFA (16 NICs/node,
25 GB/s/NIC effective per PR#612, 400 GB/s/node theoretical, ~2.2% NIC util = issue-bound),
Qwen3-30B-A3B-FP8 EP16 unless noted. SRD: no atomics, no ordering, CQ-local, 1-SGE write.
Sanitized — no IPs / ECR account IDs / hostnames.

## Serving A/B — DeepEP vs same-framework dense (the apples-to-apples comparators)

| framework | DeepEP backend | dense backend | regime | DeepEP agg tok/s | dense agg tok/s | DeepEP/dense |
|---|---|---|---|---|---|---|
| TRT-LLM 0.21 | ElasticBuffer GIN-V2 | NotEnabled (allgather) | conc4/ISL120/EP16 | 64.24 | 106.36 | **0.60×** |
| TRT-LLM 0.21 | DeepEP + EP_NUM_COMMS=8 | dense | conc4/ISL120/EP16 | 68.7 | 105.66 | **0.65×** |
| TRT-LLM 0.21 | DeepEP-mc8 | dense | conc32/ISL120/EP16 | 545.04 | 794.56 | **0.69×** |
| TRT-LLM 0.21 | DeepEP-mc8 | dense | conc16/ISL2048/EP16 | 284.22 | 424.67 | **0.67×** |
| vLLM | deepep_v2 (EPNC8) | allgather_reducescatter | conc32/decode/EP16 | 296 | 304 | **0.97× (TIE)** |
| vLLM | deepep_v2 (EPNC8) | allgather_reducescatter | conc64/prefill/EP16 | 18201 | 22797 | **0.80×** |
| vLLM | deepep_v2 | allgather_reducescatter | EP32 prefill (4-node) | — | — | **1.02× (TIE)** |
| vLLM | deepep_v2 | allgather_reducescatter | EP32 decode | — | — | **1.00× (TIE)** |

**Per-user tok/s (the decode-step tax signature):** DeepEP flat 19.4-19.7 (σ=0.13) vs dense flat
26.9-28.1 (σ=0.51), across conc4→32 and ISL120→2048 ⇒ fixed +14.7 ms/token CPU-proxy tax.

## DeepEP vs UCCL (same framework SGLang, ≥2-rep, decode var <1%)
| conc | DeepEP-V2 | UCCL | DeepEP/UCCL |
|---|---|---|---|
| 32 | 263.9 | 254.2 | 1.038× |
| 256 | 2173.8 | 2105.4 | 1.032× |
AIPerf standardized: DeepEP 1.03× agg, ~3% faster ITL/per-user. **TIE** — despite UCCL ~4× faster
transport microbench (47µs vs 186µs dispatch). All-to-all is NOT the serving critical path.

## Transport-layer (bytes — where DeepEP's O(topk) IS real)
| path | regime | DeepEP-V2 SO GB/s | comparator |
|---|---|---|---|
| dispatch | prefill, multi-comm K=8 | 33-35 | UCCL-published 22.77 → DeepEP **1.45×** |
| dispatch | decode | 10-12 | UCCL 40 (verified once) → UCCL 3.6× |
| dispatch | 128 experts | 36 | — |
| dispatch | 256 experts | 13 | more experts = SLOWER (0.36×) |

## Multi-comm scaling (how to RUN DeepEP best — the surpass knob)
K=1: ~8 SO · K=2: 12-13 (1.7×) · K=4: 21 · K=8: 33 (4.3×, peak). EPNC8 recovers vLLM from
142→296 tok/s (2.09× over single-comm) = parity with allgather (NOT beating it).

## WHERE DeepEP WINS vs LOSES
WINS: transport bytes (1.45× UCCL); CAPACITY (EP-itself, fits too-big model — allgather has it too);
multi-comm recovery to parity; 3-framework drop-in proven; ~3% over UCCL on AIPerf.
LOSES/TIES on SERVING tok/s: TRT dense 1.5× faster; vLLM allgather TIE; prefill 1.25× to dense;
EP32 dead tie; expert-count anti-scales. Root cause: a2a not the serving critical path + fixed
CPU-proxy per-step tax (2.2% NIC util, issue-bound) that never amortizes. Only GDAKI/IBGDA would
convert it — not available on EFA.

## Sources
docs/experiments/EXP-AB-TRTLLM-DEEPEP-VS-DENSE-2026-06-23.md, EXP-AB-DEEPEP-MULTICOMM-VS-DENSE-2026-06-23.md,
EXP-AIPERF-SGLANG-AB-2026-06-20.md, EXP-UCCL-SGLANG-LIVE-2026-06-19.md, EXP-LOOP2-BASELINE-2026-06-23.md,
docs/research/2026-06-19-MAKE-DEEPEP-BENEFICIAL-CLOSEOUT.md, evidence/FAIR-AB-MULTICOMM-20260623T120859Z/.
