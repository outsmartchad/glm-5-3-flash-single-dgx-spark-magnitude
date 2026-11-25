# Troubleshooting

## Download is too large or fails

The IQ2_XXS checkpoint is about 102 GB on disk. Confirm sufficient SSD space and a stable connection before starting. Keep the same checkpoint revision when comparing results.

## Severe memory pressure

The measured process tree peaked at 98.30 GiB on a Spark with about 121.7 GiB reported system memory. Close other model servers and avoid additional workloads during startup and inference. Do not infer safe context capacity from the checkpoint size alone. This recipe only reports the tested 64K workload; it did not test 128K.

## CUDA or NVRTC initialization fails

Check `nvidia-smi` and the Magnitude runtime installation. When building the pinned source, expose the Magnitude runtime directory containing `libnvrtc.so.12` through `SEISMIC_NVRTC_DIRECTORY`; `scripts/benchmark.sh` tries to locate it automatically.

## Local API does not respond

```bash
ss -ltnp 'sport = :10100'
curl --fail-with-body http://127.0.0.1:10100/inference/v1/models
```

Start `magnitude serve` if the listener is absent. Keep requests on loopback. Do not run `magnitude server setup` unless remote access is intentionally configured.

## Benchmark is slower than Magnitude’s 28 tok/s listing

Check the checkpoint, prompt token count, output budget, cache policy, engine revision, temperature, thermal state, and tuning report. Our measured median was 14 tok/s with 95.4% canonical fallback. The raw run is needed to diagnose the gap; neither number alone identifies its cause.

## A pass is invalid

Inspect the run’s `events.jsonl`, `logs/`, and `summary.json`. The harness retains invalid events but excludes them from timing. This published run had two valid measurements out of two planned.
