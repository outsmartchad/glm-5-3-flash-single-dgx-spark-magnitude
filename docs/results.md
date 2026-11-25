# Benchmark results and method

## Summary

| Requests | Prompt tokens | Output tokens | Decode median | Prefill median | TTFT median |
|---:|---:|---:|---:|---:|---:|
| 1 at a time | 64,363 | 256 | **14.0 tok/s** | **641.5 tok/s** | **100.59 s** |

The individual decode results were 14.27 and 13.83 tok/s. Both requests were valid. Cold means a new benchmark engine process with prompt-cache reuse disabled; the model files themselves remain downloaded locally.

## Workload and caveats

- DGX Spark with NVIDIA GB10 and 128 GiB unified system memory; Magnitude engine used CUDA.
- Catalog model `glm-5.3-flash:gguf:iq2`, Unsloth Dynamic IQ2_XXS checkpoint (about 102 GB on disk).
- Official harness template `repeat:64k` rendered to 64,363 input tokens; two sequential, single-request measurements, each with 256 output tokens.
- Greedy output, thinking disabled, fresh tuning. This catalog model has an embedded MTP layer but no separate draft artifact; the run recorded no draft acceptance counter.
- No prose, coding, tool-call, Chinese, or answer-quality scoring was performed. Do not compare this value directly with TensorFold prose measurements or treat it as a quality comparison.
- Magnitude’s published DGX Spark row reports 28 tok/s decode and 479 tok/s prefill for GLM 5.3 Flash. This run measured 14.0 and 641.5 respectively. The difference is a real result under this pinned harness and environment, not proof that the publisher’s number is universally wrong.
- Fresh tuning ended with 124/130 units at canonical fallback (95.4%), 6 qualified, and 44 expired. Treat this as a fresh-tuning baseline, not the fully optimized limit.
- Process-tree RSS peaked at 98.30 GiB in pass one and 97.14 GiB in pass two. This is the captured process tree, not all system memory or GPU-only allocation. GLM leaves little spare memory on this Spark; no 128K run was attempted.

## Reproduce

The run used Magnitude inference source commit `78c92e4f21e38f9528f21e09c5e43936a04b6b0d`, two passes, a fresh tuning cache, CUDA, and the local host backend.

```bash
./scripts/benchmark.sh
```

The script pins the source revision and official workload. It requires Git, Rust/Cargo, `uv`, Python tooling declared by Magnitude, and the Spark CUDA/NVRTC runtime. It leaves the complete harness output under `results/<run-id>/bench/`.

## Provenance

- Benchmark date: 2026-11-25 UTC.
- Magnitude source revision: `78c92e4f21e38f9528f21e09c5e43936a04b6b0d`.
- Model revision: Unsloth repository snapshot `a38483c8cd5df544f53d70fb281afe97369d5ab6`.
- Target checkpoint: `UD-IQ2_XXS`, approximately 102 GB on disk.
- Raw report: [`../raw/bench/report.md`](../raw/bench/report.md); raw JSON: [`../raw/bench/result.json`](../raw/bench/result.json), [`../raw/bench/summary.json`](../raw/bench/summary.json).

## Published comparison

Magnitude’s [benchmark page](https://magnitude.dev/benchmarks) lists 28 tok/s decode and 479 tok/s prefill for GLM 5.3 Flash on a DGX Spark. It describes a 64K context, greedy output, one request, a short fixed output budget, fresh engine processes and native engine timing. Our run used the matching `repeat:64k` workload and records its actual prompt size and resources in raw artifacts. Engine revision, checkpoint files, compiler/tuning state, and thermal conditions may change results.
