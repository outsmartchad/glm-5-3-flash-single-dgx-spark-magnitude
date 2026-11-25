# Serving bench

Host: spark-6154 · Cortex-X925 + Cortex-A725 · 121.688 GiB RAM · Linux 7.0.0-1025-nvidia

Status: **completed**

```sh
magnitude-bench job --inference /home/outsmartchad/.cache/magnitude-evaluation/outputs/d5bb3a407c23d17f-0bd87f38c1ce149e/src --model glm-5.3-flash:gguf:iq2 --passes 2 --fresh-tuning --magnitude-device cuda --magnitude-binary /home/outsmartchad/.cache/magnitude-evaluation/outputs/d5bb3a407c23d17f-0bd87f38c1ce149e/target/release/magnitude-engine --magnitude-cache-dir /home/outsmartchad/.cache/magnitude-evaluation/runs/20261125T084019Z-bench-6bb9e2/kernels --harness /home/outsmartchad/.cache/magnitude-evaluation/outputs/d5bb3a407c23d17f-0bd87f38c1ce149e/target/release/magnitude-render --template /home/outsmartchad/.cache/magnitude-evaluation/runs/20261125T084019Z-bench-6bb9e2/inputs/f67f31e10d8a2109.json --artifacts /home/outsmartchad/.cache/magnitude-evaluation/runs/20261125T084019Z-bench-6bb9e2/artifacts.json --output /home/outsmartchad/.cache/magnitude-evaluation/runs/20261125T084019Z-bench-6bb9e2/bench --build-identity /home/outsmartchad/.cache/magnitude-evaluation/runs/20261125T084019Z-bench-6bb9e2/bench/build.json
```

Recorded 2 / 2 planned measurements.

Outcomes: {'valid': 2}

| Target | Section | Concurrency | Template | Actual prompt tokens | Eligible / recorded | TTFT ms | Completion ms | Prefill tok/s | Decode tok/s | Drafts accepted |
| --- | --- | ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| magnitude | context | 1 | repeat:64k | 64363 | 2 / 2 | 100,591.6 | 118,820.9 | 641.5 | 14.0 | — |

Latencies and phase rates above are medians; nearest-rank p95 latency is in summary.json.

Moby Dick repeat: each request repeats a Moby Dick passage back; no answer-quality scoring. EOS or reaching the output budget ends a valid measurement; actual lengths are in summary.json.

Truncation, protocol errors, timeouts and cancellation are excluded from latency and rates, and retained in outcome counts and accuracy.

MLX-VLM generation timing measures server token emission. Its phase rates are shown for inspection and must not be interpreted as native model-service ratios against other engines.

product comparison of each engine's serving of one catalog model

disabled; shared session history does not imply retained prefix reuse

Raw request streams, per-process memory samples, source identities and engine logs are retained alongside this report.

Process-tree RSS (includes loading; not isolated GPU allocation):

| Target | Pass | Ready GiB | Peak GiB |
| --- | ---: | ---: | ---: |
| magnitude | 1 | 95.68 | 98.30 |
| magnitude | 2 | 96.44 | 97.14 |

Elastic expert bank (experts' slots, from the engine's memory observations while serving):

| Target | Pass | Slots first → last | Range | Floor | Ceiling | Resizes seen | Latest resize |
| --- | --- | ---: | ---: | ---: | ---: | --- | --- |
| magnitude | block-0 | 287 → 249 | 247–287 | 16 | 288 | experts Grew 2 | experts Grew 247 → 249 |
| magnitude | block-1 | 285 → 253 | 247–285 | 16 | 288 | experts Grew 2 | experts Grew 247 → 253 |

## Tuning health

> **WARNING: High tuning fallback rate** — 124/130 (95.4%) units fell back to canonical default (6/130 qualified, 44 expired).

Worst `default` vs. `tuned` gaps (>1.25×; 1 total):

| Unit | Default µs | Best µs | Ratio | Status |
| --- | ---: | ---: | ---: | --- |
| `dense_expand [NW=bf16,GW=q5k@mma16,UW=q5k@mma16,A=bf16,function=0,M=65..]` | 498.9 | 338.1 | 1.48× | qualified |

Slowest untuned / fallback units:

| Unit | Reason | Median µs | Weight | Weighted ms |
| --- | --- | ---: | ---: | ---: |
| `attention_prefill [A=bf16,M=1..17;codec=CodecSpec { key: Dense { dtype: BF16 }, key_width: 512,…` | Canonical | 1,146.0 | 117.4 | 134.56 |
| `attention_project [NW=bf16,QW=q5k@mma16,GW=q8g32s@mma16,KW=q8g32s@mma16,VW=q8g32s@mma16,A=bf16,…` | Expired | 1,583.7 | 68.8 | 108.93 |
| `kimi_delta_chunk [RN=bf16,A=bf16]` | Canonical | 254.8 | 356.0 | 90.73 |
| `routed_gate_up_part [EGW=iq2xxs@mma16,EUW=iq2xxs@mma16,A=bf16,M=4..17]` | Canonical | 545.4 | 136.0 | 74.16 |
| `routed_gate_up [EGW=iq2xxs@mma16,EUW=iq2xxs@mma16,A=bf16,M=4..17]` | Canonical | 543.2 | 136.0 | 73.86 |
