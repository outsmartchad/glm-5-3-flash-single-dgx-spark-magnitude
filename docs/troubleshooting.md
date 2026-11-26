# Troubleshooting

## Preflight reports missing tools

Follow [setup](setup.md#1-check-the-spark) to install the build tools, rustup, and uv. Reopen the shell after installing Rust or uv, or add their user bin directories to `PATH`. Run `./scripts/recipe.py doctor` on the Spark. If `nvidia-smi` fails, fix the Spark's NVIDIA driver before building or loading.

## NVRTC cannot load

The pinned native CUDA engine opens `libnvrtc.so.12` and `libnvrtc-builtins.so.12.9` from the same directory. Run `prepare` to fetch and verify the ARM64 SBSA build, or set `SEISMIC_NVRTC_DIRECTORY` to a directory holding both files. A CUDA 13 driver and toolkit can coexist with this recipe's CUDA 12.9 runtime compiler. Renaming a CUDA 13 library does not supply the CUDA 12 ABI. [Magnitude CUDA compatibility](https://github.com/magnitudedev/magnitude/blob/78c92e4f21e38f9528f21e09c5e43936a04b6b0d/inference/docs/compatibility.md)

## Model acquisition fails

Check disk space on both the model store (`~/.magnitude/models`) and build cache. The catalog package contains 4 target GGUF shards plus the recorded auxiliary artifacts. Keep them together and rerun `prepare`. Its artifact check expects the pinned snapshot and the measured target and MTP identities. When comparing a replacement quant, collect a new baseline.

## The API is unavailable

Wait for the foreground server to announce its listener. Then run `./scripts/recipe.py status` and `./scripts/recipe.py smoke`. The wrapper uses the native engine at `/v1`. The separate Magnitude app and `magnitude serve` use `/inference/v1`. If port 10100 is already taken, choose a different `--port` and use it for each command. Run the commands on the Spark or forward its loopback port through SSH.

## The model cannot load or the client reports a device or transport error

Stop other model servers and retry the measured context profile. Check the foreground engine logs and `free -h`. The model file, process RSS, context state, temporary compute workspace and OS allocations describe different parts of the memory budget.

An earlier stock Magnitude 0.3.2 load selected a 1M-token context and returned `device_lost` on two localhost chat requests. An earlier source build also failed during expert-bank resizing. Start from this recipe's pinned revision and 64,768-token context. The successful runs here establish 64K repeat performance. A 128K timed result is unavailable.

## The numbers differ from the published benchmark

Match the model snapshot, engine revision, prompt, 256-token output cap, disabled prefix cache and draft method. Compare individual timed rows, never warmups. A fresh kernel-tuning run can fall back to default kernels. Check the eligible-pass count and error outcomes before using the rates. [Benchmark method and traces](results.md)
