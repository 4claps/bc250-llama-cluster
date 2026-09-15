# Changelog

## 2026-09-15

### Fan control hysteresis fix

A physical case modification caused audible fan cycling at idle. Root cause was a lack of hysteresis in the fan control curve combined with normal idle GPU temperature oscillation, worsened by the case change. Fixed by adding a temperature deadband to `bc250-fan-control` so the fan only changes speed once temperature crosses meaningfully past a curve point, instead of reacting to every small fluctuation. Confirmed stable on both Bowie and Crockett. The repo template and a new `bc250_fan_control_hysteresis_degrees` variable have been updated to match, on a separate branch pending PR and merge.

### RPC interconnect NIC binding fix

The direct point-to-point RPC link between Bowie and Crockett uses USB-to-RJ45 adapters, and the NetworkManager connection profile for each was matching by interface name rather than by the adapter's hardware MAC address. This caused the RPC link to fail to come up after a cold start whenever the USB adapter re-enumerated under a different interface name. Fixed on both nodes by binding the profile to the adapter's actual hardware MAC instead. Also added a second profile on each node matching the other node's adapter MAC, so if the two physical adapters are ever swapped between nodes, either one will be correctly recognized and configured automatically rather than requiring manual troubleshooting again.

### MTP speculative decoding enabled in production

Discovered that Qwen3.6-35B-A3B ships with an unused NextN/MTP head, a small extra model component that can draft its own next tokens for speculative decoding, verified by the main model. llama.cpp supports this via `--spec-type draft-mtp`, but there is a known compatibility bug on AMD Vulkan hardware where the default GPU-side drafting sampler crashes; worked around with `--no-spec-draft-backend-sampling`. Tested standalone at increasing context sizes up to the full production context of 115000 with no crashes and no RPC instability, then promoted to the live production `llama-server.service`. Result: generation speed improved from roughly 52 tokens per second baseline to roughly 66 to 72 tokens per second, confirmed across multiple real requests at full production context.
