# Operations

## Deployment order

The normal `site.yml` order is:

1. Read-only preflight.
2. Base Fedora configuration.
3. BC250 detection, pinned fan/sensor driver, fan telemetry, fail-safe fan
   control, and live cooling validation.
4. BC250 hardware baseline, read-only CU topology, TTM, and selected
   performance profile.
5. Dedicated backend networking.
6. Fedora RADV/Vulkan.
7. Pinned llama.cpp Vulkan/RPC build.
8. Crockett RPC service, then Bowie coordinator service.
9. Cluster validation and temporary bandwidth test.

Hardware and network plays use `serial: 1` where a failure could affect access.
Before deployment, run local syntax and lint checks. Check mode should then be
limited to one host at a time:

```bash
ansible-playbook -i inventories/production/hosts.yml site.yml \
  --check --diff --limit bowie
ansible-playbook -i inventories/production/hosts.yml site.yml \
  --check --diff --limit crockett
```

Check mode cannot prove compilation, link activation, Vulkan execution, RPC
operation, or hardware stability.

The Qwen3-Coder-Next fit plan is retained as historical methodology. The
completed 64K results and current Hermes recommendation are in
`docs/hermes-model-bakeoff.md`.

Cooling is a hard production prerequisite. The cluster role refuses to enable
inference services when cooling is bypassed or the configured fan channel is
not positively qualified, and both systemd inference units require the fan
controller. The explicit bypass is diagnostic and non-production only.

## Fedora update behavior

`base_apply_updates: true` applies normal Fedora updates; it does not freeze an
immutable operating-system image. `base_reboot_automatically: false` reports
changed packages but leaves reboot timing to the operator. Review kernel, Mesa,
firmware, and Vulkan changes, reboot one board at a time, and repeat the full
validation workflow before treating an updated system as known-good. Set
`base_apply_updates: false` only when intentionally controlling updates through
another documented process.

## CU live-manager operations

Normal `site.yml` never changes WGP routing. Status is read-only:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/21-cu-live-status.yml --limit bowie
```

Temporary candidate testing requires one host, an exact per-host WGP list, an
explicit variable gate, and the special tag:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/22-cu-live-test.yml --limit bowie \
  --tags cu_live_test -e bc250_cu_live_test_enabled=true
```

Rollback disables persistence and restores the factory dispatch table:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/23-cu-live-rollback.yml --limit bowie \
  --tags cu_live_rollback
```

Persistence requires the tested table to be copied to
`bc250_cu_live_validated_wgps`, a validation record, and all persistence gates:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/24-cu-live-persist.yml --limit bowie \
  --tags cu_live_persist
```

All three CU write paths hard-fail unless exactly one host is selected. `serial:
1` is not a substitute for `--limit`. Normal validation performs read-only
status calls and compares every active WGP against the approved per-host table
when persistence is enabled; it performs no MMIO writes.

## Hermes agent access boundary

Hermes runs on a separate existing client machine; this repository deploys
only its inference backend. The client connects to Bowie's OpenAI-compatible
API. It must not connect to Crockett's internal RPC service, and Hermes is not
installed on either BC250 node.

The validated client settings are:

- Base URL: `http://10.0.0.170:8080/v1`.
- Model identifier:
  `/var/lib/llama.cpp/models/Qwen3.6-35B-A3B-Q4_K_M.gguf`.
- Provider mode: custom OpenAI-compatible.
- Context size: 65,536 tokens.
- Recommended request timeout: 300 seconds.
- API authentication: none in the validated deployment; management-network
  firewall policy is the access-control boundary.
- For short deterministic answers or forced tool calls, pass
  `chat_template_kwargs: {"enable_thinking": false}` so the completion budget
  is not consumed by Qwen reasoning. Thinking can remain enabled for normal
  reasoning workflows when the client provides an adequate token budget.

Treat Hermes as an untrusted automation principal even when it runs on the
management LAN. Give it a dedicated SSH key and account rather than access to
the operator account or its private key. Keep its API access restricted to a
specific management CIDR in `llama_api_allowed_cidrs`; never expose either the
HTTP API or the llama.cpp RPC listener directly to the internet.

Do not grant the Hermes account unrestricted `sudo` or an unrestricted shell on
either BC250 node. If it must operate the cluster, expose a root-owned wrapper
with an allowlist of reviewed, non-interactive commands. The default allowlist
should be limited to read-only status and service inspection. Require a human
operator for:

- `site.yml` and any playbook that installs packages or changes networking.
- CU test, persistence, or rollback playbooks.
- Changes to firmware, boot configuration, GPU routing, or power limits.
- Edits to inventory, pinned source revisions, checksums, or firewall CIDRs.

Keep the agent's SSH private key, API credentials, and model-download
credentials outside this repository. Review the wrapper and its sudoers entry
after every allowlist change, and retain Ansible output or system journal logs
for unattended actions.
