# Troubleshooting

Validate in dependency order:

- `lspci` reports `1002:13fe` with `amdgpu` bound.
- A render node exists and the service user belongs to `render` and `video`.
- `vulkaninfo --summary` reports RADV GFX1013 and never llvmpipe.
- `llama-cli --list-devices` reports `Vulkan0`.
- The backend permanent MAC resolves to the expected NIC at 2500 Mbit/s.
- The peer route uses the backend source/interface and the management default
  route remains unchanged.
- Crockett listens on `10.250.0.2:50052`, not its management address or all
  addresses.
- Bowie logs show both Vulkan and RPC initialization with no CPU-only fallback.
- Kernel logs contain no GPU reset, ring timeout, page fault, or MCE.

If a live WGP candidate destabilizes the GPU, run the rollback playbook or
reboot while persistence is disabled. If persistent restore is implicated,
disable the service and restore stock dispatch before further testing.
