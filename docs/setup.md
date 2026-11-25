# Install and make a first local request

## Install

On the DGX Spark, install Magnitude using its [official CLI guide](https://docs.magnitude.dev/). Start the service in the foreground and leave it running:

```bash
magnitude serve
```

The local inference API defaults to `127.0.0.1:10100/inference/v1`. Keep it loopback-only. Do not run `magnitude server setup` for this local recipe.

In another terminal, pull and load the GLM catalog quant:

```bash
magnitude models pull glm-5.3-flash:gguf:iq2
magnitude models load glm-5.3-flash:gguf:iq2
```

Verify the binding:

```bash
ss -ltnp 'sport = :10100'
```

It should show `127.0.0.1:10100`, not `0.0.0.0:10100` or a LAN/Tailscale address.

## First chat smoke test

```bash
curl --fail-with-body http://127.0.0.1:10100/inference/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"glm-5.3-flash:gguf:iq2","messages":[{"role":"user","content":"Reply with exactly SPARK_OK."}],"temperature":0,"max_tokens":32}'
```

This tests local API reachability only. It is separate from the 64K throughput test and does not assess answer quality.

## Stop

Stop the foreground service with `Ctrl-C`. Do not expose the listener on a public interface for this recipe.
