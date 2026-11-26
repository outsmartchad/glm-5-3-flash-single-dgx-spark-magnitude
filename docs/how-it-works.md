# How GLM 5.3 Flash runs on one DGX Spark

This recipe serves Z.ai's GLM 5.3 Flash model using Unsloth's **UD-IQ2_XXS GGUF** and Magnitude's CUDA engine. The model has **321B total parameters** and about **18B active per token**. It fits this single Spark configuration by combining sparse computation, quantized weights and an engine that can place part of the expert bank on disk. [GLM model card](https://huggingface.co/zai-org/GLM-5.3-Flash) · [Magnitude catalog](https://magnitude.dev/benchmarks)

## 1. What happens to a prompt

The chat template assembles the messages. A tokenizer turns the text into IDs, and an embedding maps each ID to a numerical vector. A sequence of decoder blocks lets each token incorporate earlier information before scoring the next token.

**Prefill** processes the whole input and builds the internal state needed for generation. **TTFT** is the time until the client receives its first generated token. **Decode** advances that state and selects subsequent tokens. Large prompts can take much longer to process than a short completion, even when the decode throughput is unchanged.

## 2. The model's 45 main decoder blocks

Each block has a mixer that combines information across tokens and a feed-forward sublayer that transforms each token. The specific published Flash configuration has 45 main blocks, 34 with linear attention and 11 with sparse attention. The first three feed-forward blocks are dense. The remaining blocks route into **8 of 288 routed experts**, with a shared expert also available on the routed blocks. [Z.ai configuration](https://huggingface.co/zai-org/GLM-5.3-Flash/blob/main/config.json)

| Component | Role |
| --- | --- |
| Kimi Delta Attention | Maintains a recurrent summary of the prefix. Local convolutions and learned decay gates help control what enters and stays in the state. |
| Sparse multi-head latent attention | Compresses each history entry into a 512-wide latent vector. Attention reads selected latent entries and reconstructs the contributions the heads need. |
| Pooled history indexer | Scores groups of four prior tokens and selects work for attention up to a 2,048-token budget, with a tail path for recent tokens. |
| Hyper-connections | Four residual streams carry information between blocks. Learned read and write operations mix the streams around each sublayer. |
| Mixture of experts | A router scores experts for each token. The chosen networks transform the token and their weighted outputs combine with the shared expert. |

A recurrent summary gives cheap incremental updates, while the indexed attention blocks can retrieve specific elements of the past. Latent history reduces the context memory per attention token. The model's 18B active count remains an approximation of the weights touched per token. Across a long prompt, different tokens can need most experts in a layer. [Magnitude's GLM family implementation](https://github.com/magnitudedev/magnitude/blob/78c92e4f21e38f9528f21e09c5e43936a04b6b0d/inference/engine/families/glm5next/src/lib.rs)

## 3. Quantized weights and embedded draft layer

- **GGUF** packages the tensor values and their formats alongside tokenizer and architecture metadata. The catalog package has four target shards at one pinned Unsloth revision.
- **UD-IQ2_XXS** stores much of the model using a very low bit quantization format. Unsloth's dynamic quantization uses different representations where helpful. The shorthand “2-bit” describes a quant family. Actual file size includes metadata, scales and tensors held at other precisions.
- **Embedded MTP** adds a trailing prediction block to the main decoder's results. It combines the target's hidden representation with a candidate token embedding to predict a further token. Magnitude asks the target to verify these proposals. Accepted proposals can reduce the number of expensive sequential target steps.

The native run requested MTP and the embedded head reported draft counters in both warmups. The timed passes lacked those counters. The result therefore documents a configured serving profile and achieved throughput, with no measured acceptance rate or isolated MTP gain. [Unsloth quant](https://huggingface.co/unsloth/GLM-5.3-Flash-GGUF) · [Raw timings](../raw/bench/results.jsonl)

## 4. How a 102 GB checkpoint loads on a 128 GB Spark

GB10 shares a physical memory pool between CPU and GPU. CUDA allocations, the OS, model pages and other processes all compete for it. The GGUF file size only describes the checkpoint on SSD. During inference, the engine also needs space for active experts, prompt/cache state, temporary activations, draft weights and compute buffers.

Magnitude maintains an **elastic expert bank**. Shared weights and experts that fit stay resident. When the model needs an expert outside that bank, the runtime can fetch and stage its weights. Keeping more in memory can save reads. A larger prompt and its temporary work can make the engine shrink the bank, so a workload that fits at load time can still be tight during inference.

The successful run sampled **247 to 287 expert slots** out of a 288-slot ceiling and recorded nonzero disk-backed expert weights. Slot counts describe bank capacity. They do not give SSD transfer volume or prove what limited decode throughput. The **98.30 GiB** result is peak sampled process-tree RSS. Linux reported **121.69 GiB** for the whole machine. These figures describe different things. [Allocation samples](../raw/bench/allocations.jsonl) · [RSS footprints](../raw/bench/footprints.jsonl)

## 5. Why we chose this launch profile

The official model configuration allows positions up to 1,048,576 tokens. Allocating and serving that many tokens depends on the runtime, quant and available memory. An earlier stock CLI load selected the 1M profile and failed on a local chat request. The native engine and **64,768-token context** in this recipe match the completed run. The server accepts text, with the vision projector disabled to match the test setup.

`prepare` pins the Magnitude source, acquires the checked catalog artifacts, and stages NVIDIA NVRTC 12.9 for local CUDA kernel compilation. `serve` starts one text server on loopback. The native benchmark owns its own loopback process, uses zero input cache hits and times each of two 256-token completions after ingesting 64,363 tokens. This is a single-request repeat workload. Our published data includes both passes and the record of which kernel decisions fell back to defaults. [Setup](setup.md) · [Timed results and previous failures](results.md) · [Troubleshooting](troubleshooting.md)
