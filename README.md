# GLM 5.3 Flash on one DGX Spark with Magnitude

Run the 321B-total / 18B-active GLM 5.3 Flash locally with Magnitude’s GPU/RAM/SSD expert paging. This is a separate recipe from Magnitude’s published result and from our other inference engines.

## Measured result

Measured on one NVIDIA DGX Spark (GB10, 128 GB unified memory) on 2026-11-25 with Magnitude’s CUDA engine, the official catalog IQ2_XXS checkpoint, and a fresh 64K-context benchmark process:

| Workload | Prompt | Output | Prefill | TTFT | Decode |
|---|---:|---:|---:|---:|---:|
| One request, cold 64K repeat prompt | 64,363 tokens | 256 tokens | **641.5 tok/s** | **100.59 s** | **14.0 tok/s median** |

Decode is the median of two valid requests (14.27 and 13.83 tok/s). Process-tree RSS peaked at **98.30 GiB**. The benchmark completed both passes, but this leaves little memory headroom for other workloads and is why this recipe does not claim a 128K result. Fresh tuning reported **124/130 (95.4%)** kernel units at canonical fallback; this is a measured baseline, not a fully tuned ceiling.

Magnitude’s published DGX Spark figure is 28 tok/s decode and 479 tok/s prefill. Our exact run measured lower decode and higher prefill. See [results and method](docs/results.md) and raw run artifacts before drawing conclusions from the difference.

## Quick start

Install Magnitude using its [official CLI installation guide](https://docs.magnitude.dev/). Start the service in the foreground:

```bash
magnitude serve
```

In another terminal, pull and load the official IQ2_XXS catalog model:

```bash
magnitude models pull glm-5.3-flash:gguf:iq2
magnitude models load glm-5.3-flash:gguf:iq2
```

Keep the service bound to loopback. Do **not** run `magnitude server setup` for this local-only recipe. See [first chat and local API](docs/setup.md).

## Reproduce the benchmark

From this repository on the Spark:

```bash
./scripts/benchmark.sh
```

The script checks out the pinned Magnitude inference source and runs the two-pass, single-request, cold `repeat:64k` CUDA benchmark with fresh tuning. GLM has no separate draft checkpoint in this Magnitude catalog entry; the harness reported no proposal acceptance counter. See [benchmark method](docs/results.md#reproduce).

## Guides and data

- [How it works](docs/how-it-works.md): sparse MoE, IQ2_XXS, memory tiers, and runtime behavior.
- [First chat and setup](docs/setup.md): install, loopback-only API smoke test, and request example.
- [Results and raw data](docs/results.md): methods, caveats, resource peaks, and provenance.
- [Troubleshooting](docs/troubleshooting.md): startup, memory pressure, API binding, and benchmark issues.
- [`raw/bench/`](raw/bench/): full report, raw request/measurement data, compressed tuning and thermal traces, process memory, and logs.

## Sources

- [Magnitude benchmark page](https://magnitude.dev/benchmarks)
- [Magnitude source and docs](https://github.com/magnitudedev/magnitude)
- [Unsloth IQ2_XXS checkpoint](https://huggingface.co/unsloth/GLM-5.3-Flash-GGUF/tree/main/UD-IQ2_XXS)
- [GLM-5 technical report](https://arxiv.org/abs/2602.15763)
