# How GLM 5.3 Flash runs through Magnitude

This guide explains the model and the measured serving path. It distinguishes sparse model computation from the engine’s method for placing weights in memory.

## 1. Sparse mixture of experts

Magnitude’s catalog describes GLM 5.3 Flash as **321B total parameters, about 18B active per token, 288 routed experts, and 8 selected experts per token**. The total count is the full parameter inventory. At a routed feed-forward layer, a learned router scores candidate experts and activates only a small top-k set for the current token. The selected expert outputs are combined; shared layers still run for every token.

This sparsity reduces compute per token relative to running every expert densely. It does not mean the entire checkpoint fits in GPU memory: this quantized model is about 102 GB on disk, and the runtime must account for shared weights, expert working sets, activations, context state, and operating-system memory.

## 2. The loaded checkpoint and quantization

This recipe uses the Magnitude catalog entry backed by Unsloth’s **Dynamic IQ2_XXS** GGUF checkpoint. IQ2_XXS is an importance-aware, very-low-bit quantization format; “2-bit” is a format family name, not a promise that every tensor is stored at exactly two bits including metadata. Quantization reduces storage and memory traffic while approximating the original model weights. It can affect output quality; this throughput benchmark did not score quality.

Magnitude lists an MTP layer embedded in the model file rather than a separate drafter artifact. The measured run’s result data contains no separate draft proposal/acceptance counter. We therefore do not claim an MTP speedup for this run.

## 3. Token generation

For each step, the model consumes the prompt or previous tokens and computes the next-token distribution. The router chooses experts at routed layers; attention and the remaining layers update the context representation. A sampler then selects the next token. In this benchmark the sampler is greedy, so the highest-scoring token is selected. Thinking is off, and each valid request ends at 256 generated tokens.

The timed workload uses a repeated public-domain text passage to build a prompt around 64K tokens. It measures prompt ingestion (prefill), the time until the first generated token (TTFT), and subsequent token emission (decode). It does not measure normal prose quality, reasoning, coding, tool use, or answer correctness.

## 4. Magnitude’s memory tiers

Magnitude’s design keeps frequently used shared model data and experts in GPU-accessible memory, places additional experts in system RAM, and streams the rest from NVMe storage. The model’s router determines which experts are needed for a token. The runtime tries to predict upcoming expert usage and stage those weights ahead of use.

On DGX Spark, GB10 shares a unified 128 GB memory pool with the CPU. GPU-oriented allocations and host-side process memory still have different access paths and performance characteristics, but they are not two independent 128 GB pools. SSD-resident experts save memory at the cost of storage reads and staging latency. Expert locality, prefetch accuracy, bandwidth, and competing allocations affect throughput.

This benchmark recorded a process-tree RSS peak of 98.30 GiB. It is not a total machine memory snapshot or a dedicated GPU allocation metric. The near-cap peak is why this recipe keeps to the tested 64K prompt and does not assert safe capacity at 128K.

## 5. What the result says

The measured median was 14.0 tok/s decode with 641.5 tok/s prefill and 100.59 s TTFT. Magnitude’s published Spark figure is 28 tok/s decode. Our raw result shows the difference under one pinned local run. Fresh tuning left 95.4% of units on canonical fallback, so this result is not a tuned performance ceiling. The benchmark’s Moby Dick repeat workload is not a quality test.

## Sources

- [Magnitude benchmark and catalog](https://magnitude.dev/benchmarks)
- [Magnitude source repository](https://github.com/magnitudedev/magnitude)
- [Unsloth GLM 5.3 Flash GGUF checkpoint](https://huggingface.co/unsloth/GLM-5.3-Flash-GGUF/tree/main/UD-IQ2_XXS)
- [GLM-5 technical report](https://arxiv.org/abs/2602.15763)
