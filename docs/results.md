# Benchmark results

## Measured on November 25, 2026

One NVIDIA DGX Spark, GB10 CUDA, 128 GB unified memory (121.69 GiB visible to Linux). These are two separate single-request passes of Magnitude's `repeat:64k` *Moby Dick* fixture. Both used **64,363 actual input tokens, zero cached input tokens and 256 generated tokens** with greedy decoding and thinking off.

| Eligible pass | Decode tok/s | Prefill tok/s | Client TTFT | Peak process-tree RSS |
| --- | ---: | ---: | ---: | ---: |
| 1 | 14.27 | 639.6 | 100.89 s | 98.30 GiB |
| 2 | 13.83 | 643.5 | 100.29 s | 97.14 GiB |
| Median | 14.05 | 641.5 | 100.59 s | — |

The two medians summarize the repeatability of the workload. The card uses the fastest valid **decode** pass and pairs it with that pass's prefill and TTFT. Full traces retain the warmups and individual completions. [Per-request records](../raw/bench/results.jsonl) · [Upstream summary](../raw/bench/summary.json) · [RSS by pass](../raw/bench/footprints.jsonl)

## What the benchmark measures

1. **Prompt and engine setup.** The native harness renders a repeat instruction using the model's chat template. The selected context class is 65,536 tokens. The harness served a 64,768-token capacity with one request at a time and disabled prompt-prefix reuse. The model files were already downloaded, and each pass launched a fresh native inference process and ran a warmup before the timed prompt.
2. **Prefill.** Magnitude reports input tokens processed divided by native prompt computation time. TTFT is measured by the client from sending the request until the first generated token arrives. TTFT also includes transport and processing outside the native prefill timer.
3. **Decode.** The rig divides its native `predicted_n` by `predicted_ms / 1000`. The stream's convenience field `predicted_per_second` uses a slightly different convention. Use the rig's formula to reproduce the table from `results.jsonl`.
4. **RAM.** RSS sums sampled resident memory of the engine process tree, including load. Some memory can be shared and some GPU, file-cache and system allocations are accounted for separately. RSS entries are suitable for comparing the recorded footprint under this workload. They are separate from installed capacity and a runtime allocation limit.

**MTP.** The benchmark requested `mtp`. The embedded head reported 5 accepted of 16 proposed in each warmup. Both measured responses omitted the draft counters, so a timed acceptance rate or MTP speedup cannot be calculated.

The run used a fresh tuning cache. The report recorded **124/130 (95.4%)** dispatch units falling back to default configurations. This documents one installation and workload. Kernel tuning, prompt, concurrency, quantization, or engine changes can move the numbers. [Native report and tuning breakdown](../raw/bench/report.md)

## Reproduce

Follow [the setup instructions](setup.md), then run `./scripts/benchmark.sh` on the Spark with only this inference workload active. Inspect `runs/<run-id>/bench/report.md` for **2 eligible timed passes**. The rig excludes invalid outcomes and marks captured warmups separately. Keep `summary.json`, `results.jsonl`, `requests.jsonl`, logs, tuning, memory and allocation traces with any measurement you share. [Raw archive map](../raw/bench/ARTIFACTS.md)

| Provenance | Observed setting |
| --- | --- |
| Engine source | [`78c92e4f21e38f9528f21e09c5e43936a04b6b0d`](https://github.com/magnitudedev/magnitude/tree/78c92e4f21e38f9528f21e09c5e43936a04b6b0d) |
| Catalog target | `glm-5.3-flash:gguf:iq2` at `a38483c8cd5df544f53d70fb281afe97369d5ab6` |
| Catalog target identity | `337870cefce5fb34b2aa3ade54a0fbf61676697f7f9a906670c20edde74561f4` |
| Drafter | Embedded MTP in the catalog model package |
| CUDA runtime compiler | NVIDIA NVRTC 12.9 |
| Mode | Loopback HTTP, text only, prefix cache disabled |
| Record | [`result.json`](../raw/bench/result.json) · [`host.json`](../raw/bench/host.json) · [`magnitude-runtime.json`](../raw/bench/magnitude-runtime.json) |

Magnitude's [published Spark benchmark](https://magnitude.dev/benchmarks) lists **28 tok/s decode** for this model with a 64K context and speculative decoding. The local passes are the numbers above. There is enough information to compare the workload class and quant, while the engine revision, tuning cache and exact run conditions can still differ. This workload measures inference throughput. It provides no answer-quality score or interactive prose or concurrency result for this quant.

## Earlier startup attempts

An earlier stock Magnitude 0.3.2 profile auto-selected a 1M-token context. GLM reached runtime ready on loopback, but its short chat requests failed with HTTP 500 `device_lost` after a CUDA illegal memory access. A separate 32K attempt on an earlier source build completed a warmup, then aborted during expert-bank resizing. These attempts yielded zero eligible timed measurements. The pinned 64K run above completed both passes. Use that profile for reproduction. A measured 32K/128K cold-prefill pair and five-user decode run are still unavailable for GLM.
