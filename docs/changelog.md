# Changelog

## 2026-09-19

### Gemma 4 bake-off: not recommended

Ran a standalone test of Gemma 4 26B-A4B (MoE) and Gemma 4 31B (dense), both Q4_K_M, through the full 27-trial Hermes battery on the Moderate profile with a 600-second per-trial timeout, early stop disabled, and no speculative decoding, alongside a `gpt-oss-20b` re-run as a control. Both Gemma models scored 22/27 (the control scored 23/27). The 26B-A4B generated about 39 tokens per second with no timeouts. The 31B generated about 12 tokens per second, ran out of memory at the production 115,000 context (it was tested at 65,536 instead), and pushed Bowie past its 80°C limits. Neither beats production or `gpt-oss-20b`, so production is unchanged. A Granite 4.2 30B test was attempted but Crockett froze hard during its load, and Gemma 3 27B was not run. Full results are in [the Hermes model bake-off](hermes-model-bakeoff.md#gemma-4-bake-off-2026-09-19).

Operational note from the same run: downloading a model on Bowie while production is running got the downloader OOM-killed by the kernel, so stop production before downloading and use the plain HTTP download path (`HF_HUB_DISABLE_XET=1`).

## 2026-09-15

### Fan control hysteresis fix

A physical case modification caused audible fan cycling at idle. Root cause was a lack of hysteresis in the fan control curve combined with normal idle GPU temperature oscillation, worsened by the case change. Fixed by adding a temperature deadband to `bc250-fan-control` so the fan only changes speed once temperature crosses meaningfully past a curve point, instead of reacting to every small fluctuation. Confirmed stable on both Bowie and Crockett. The repo template and a new `bc250_fan_control_hysteresis_degrees` variable have been updated to match, on a separate branch pending PR and merge.

### RPC interconnect NIC binding fix

The direct point-to-point RPC link between Bowie and Crockett uses USB-to-RJ45 adapters, and the NetworkManager connection profile for each was matching by interface name rather than by the adapter's hardware MAC address. This caused the RPC link to fail to come up after a cold start whenever the USB adapter re-enumerated under a different interface name. Fixed on both nodes by binding the profile to the adapter's actual hardware MAC instead. Also added a second profile on each node matching the other node's adapter MAC, so if the two physical adapters are ever swapped between nodes, either one will be correctly recognized and configured automatically rather than requiring manual troubleshooting again.

### MTP speculative decoding enabled in production

Discovered that Qwen3.6-35B-A3B ships with an unused NextN/MTP head, a small extra model component that can draft its own next tokens for speculative decoding, verified by the main model. llama.cpp supports this via `--spec-type draft-mtp`, but there is a known compatibility bug on AMD Vulkan hardware where the default GPU-side drafting sampler crashes; worked around with `--no-spec-draft-backend-sampling`. Tested standalone at increasing context sizes up to the full production context of 115000 with no crashes and no RPC instability, then promoted to the live production `llama-server.service`. Result: generation speed improved from roughly 52 tokens per second baseline to roughly 66 to 72 tokens per second, confirmed across multiple real requests at full production context.
