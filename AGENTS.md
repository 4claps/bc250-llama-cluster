# Repository Guidelines

## Project Structure & Module Organization

This repository is an Ansible project for a two-node BC250 llama.cpp cluster. `site.yml` is the main entry point and imports numbered playbooks from `playbooks/` in deployment order. Reusable automation lives under `roles/<role>/`, with variables in `defaults/`, actions in `tasks/`, service notifications in `handlers/`, and Jinja templates in `templates/`. The tracked `inventories/example/` must remain safe and hardware-neutral. Real host settings belong only in the ignored `inventories/production/`. Operational guidance belongs in `docs/`.

## Build, Test, and Development Commands

Install required collections before validation:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory -i inventories/example/hosts.yml --graph
ansible-playbook -i inventories/example/hosts.yml site.yml --syntax-check
ansible-lint
```

The inventory command verifies the safe example host/group structure; syntax-check and `ansible-lint` catch structural issues without contacting nodes. Before deployment, explicitly select the ignored production inventory and preview changes one host at a time:

```bash
ansible-playbook -i inventories/production/hosts.yml site.yml --check --diff --limit bowie
```

Never run the example inventory against hardware. Run `ansible-playbook -i inventories/production/hosts.yml site.yml` only after required versions, checksums, NIC MACs, model settings, and allowed CIDRs are configured. Check mode cannot validate compilation, networking, Vulkan, or RPC behavior.

## Coding Style & Naming Conventions

Use two-space YAML indentation, `---` document markers, descriptive task names, and fully qualified module names such as `ansible.builtin.template`. Keep role variables prefixed by their role or subsystem (for example, `bc250_cu_live_*`). Preserve the numbered playbook convention (`30-network.yml`) and use snake_case for variables and task files. Put environment-specific values in inventory rather than role tasks.

## Testing Guidelines

For every change, run example-inventory, syntax, lint, and YAML checks. For host-affecting changes, explicitly select the production inventory, use `--check --diff --limit <host>`, then apply narrowly and run `playbooks/90-validation.yml`. Treat hardware-routing playbooks separately: always use their documented enable gates and tags. CU test, persistence, and rollback must hard-fail unless exactly one target host is selected.

## Known-Good Baseline Safety

The reference deployment is Fedora 44 with stock Fedora kernel and Mesa/RADV, persistent validated 40/40 CU routing on each board, the Ansible-managed Moderate profile, and the Ansible-managed TTM limits `pages_limit=3014656` and `page_pool_size=3014656`. Moderate means a 3500 MHz CPU target, scale `-22`, and a complete 500–1750 MHz GPU curve capped at 925 mV with 80°C throttling and 75°C recovery. The TTM limits expose 11.5 GiB of shared RAM as GTT plus 512 MiB hardware VRAM; they do not change UMA firmware settings. Do not change CU routing, CPU core count, UMA, TTM, kernel, Mesa, GFX1013 compute queues, clocks, model, context size, or llama.cpp revision as an incidental part of unrelated work. Hardware-routing operations must remain separately gated and must never target both boards simultaneously.

## Commit & Pull Request Guidelines

Use short, imperative summaries (for example, `Harden CU rollback safety`). Keep commits focused and explain operational or safety implications in the body. Pull requests should describe affected roles/playbooks, list validation commands and results, identify inventory or secret-handling impacts, and link relevant issues. Include logs or screenshots only when they clarify service, networking, or hardware behavior; never commit credentials, private keys, or model-download secrets.
