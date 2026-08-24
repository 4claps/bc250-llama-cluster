# Architecture

## Inference path

An existing external Hermes Agent machine is the API client; Hermes is not
installed on either BC250. It talks only to Bowie's OpenAI-compatible HTTP
endpoint:

```text
existing Hermes machine -> Bowie llama-server -> Bowie Vulkan
                                           `-> Crockett RPC -> Crockett Vulkan
```

Bowie runs the sole `llama-server`, bound to management address
`10.0.0.170:8080`. It loads the GGUF and uses its local `Vulkan0` device plus
Crockett's remote Vulkan device at `10.250.0.2:50052`. The initial split mode is
`layer`; llama.cpp assigns work in proportion to available memory unless an
explicit per-device tensor split is later validated.

Crockett runs `ggml-rpc-server`, selects `Vulkan0`, and enables its local tensor
cache. It binds only to its backend address. Bowie does not run a local RPC
server because its GPU is available directly to the coordinator.

The RPC backend is experimental and unauthenticated. It is never bound to a
management address or exposed beyond the dedicated link.

The validated llama-server deployment also has no API-key authentication.
Firewalld restricts TCP 8080 to explicitly configured trusted management
CIDRs; it must not be exposed to the internet. Do not invent a client API key
unless authentication is added deliberately at a reviewed reverse proxy or a
future supported server boundary.

## Inventory boundary

`inventories/example` is the only published inventory and is the default in
`ansible.cfg`. It contains documentation addresses, placeholder MACs, no WGP
table, disabled CU persistence, stock performance controls, and disabled TTM
management. Operators copy it to the gitignored `inventories/production` and
explicitly select that file for every hardware-facing command. This prevents a
fresh clone from inheriting another board's hardware routing.

## Networking

Ansible selects each backend NIC by its permanent MAC using `ethtool`, not by an
assumed interface name. NetworkManager creates `10.250.0.1/30` on Bowie and
`10.250.0.2/30` on Crockett with IPv6 disabled, no DNS, no gateway, and
`never-default`. Pre/post route checks protect the management default route.

Ports:

- TCP 22: existing management SSH policy.
- TCP 8080: Bowie management address, trusted management CIDRs only.
- TCP 50052: Crockett backend address, source `10.250.0.1/32` only.
- TCP 5201: temporary backend-only rule during bandwidth validation.

## Role ownership

- `base`: Fedora packages, updates policy, timezone, NetworkManager, firewalld.
- `bc250_hardware`: hardware baseline, PCI/amdgpu checks, Fedora/BLS TTM limits,
  performance governors, UMR/live-manager install, and separately gated
  hardware experiments.
- `network`: MAC-bound backend connection, route protection, link validation.
- `vulkan`: Fedora Mesa/RADV packages and GFX1013 validation.
- `llama_cpp`: pinned upstream source, Vulkan/RPC build, versioned install.
- `cluster`: users, model, services, RPC topology, and firewall policy.
- `validation`: hardware, read-only comparison of persisted versus live CU
  routing, TTM/Vulkan capacity, llama.cpp, services, routes, and health.
