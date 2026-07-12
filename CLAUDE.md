# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repo currently contains only `SPEC.md` — no Ansible content (playbooks,
roles, inventory) has been built yet. `SPEC.md` is the authoritative plan;
read it in full before starting work. This file summarizes the parts Claude
needs on every task; SPEC.md has the full rationale.

## What this repo is

NetBox-driven Ansible network automation: consumes NetBox as the source of
truth (dynamic inventory), renders device configuration from Jinja2
templates, and applies it via playbooks and roles.

It is the companion to a sibling repo, `netbox-ai-ingest`, which populates
NetBox. The two integrate **only** over the NetBox REST API — this repo must
never import `netbox-ai-ingest` code.

## Build order — do not skip ahead

The SPEC defines a strict, tiered "fallback ladder." Work must start at Tier
1 and only advance once the current tier is proven and committed. Do not
jump to NetBox integration or a device lab before Slice 1 is done — the SPEC
explicitly calls this out as where time disappears.

- **Tier 1 / Slice 1 (start here — currently not yet built):** Pure Ansible,
  fully synthetic. Static inventory (`inventory/hosts.ini`), a `device_config`
  role with a Jinja2 template, a playbook that renders each device's config to
  `build/<hostname>.cfg`. No NetBox, no lab, no live devices.
- **Tier 2 / Slice 2 (next, the differentiator):** Swap the static inventory
  for the `netbox.netbox.nb_inventory` plugin pointed at a live NetBox
  (populated by `netbox-ai-ingest`). Requires a NetBox API token in `.env`
  (never committed).
- **Tier 3 / Slice 3 (optional, timeboxed):** Push rendered config to 1–2
  virtual devices (containerlab, Arista cEOS or Nokia SR Linux). Only attempt
  if Tiers 1–2 went fast; stop and ship Tiers 1–2 if device connectivity
  fights back.

The planned Slice 1 layout (see SPEC.md for the full spec and definition of
done):

```
ansible.cfg
inventory/hosts.ini
group_vars/all.yml
group_vars/<group>.yml
host_vars/<host>.yml
playbooks/render_config.yml
roles/device_config/tasks/main.yml
roles/device_config/templates/base.j2
roles/device_config/defaults/main.yml
README.md
```

## Hard rules

- **No real-world data, ever.** All devices, hostnames, IPs, topology, and
  configs are synthetic/fabricated. Never let real data enter the repo.
- **Credential custody.** NetBox API token and device credentials live in
  `.env` or Ansible Vault, never committed — enforced by `.gitignore`, which
  predates any secret in this repo. Slice 1 has no real secrets and does not
  need Vault; Vault arrives in Slice 2 with the NetBox token.
- **No hardcoded IPs/credentials** in playbooks or templates — reference vars
  (later: NetBox-sourced vars) instead.
- **Idempotent by design.** A second run against unchanged state must report
  no changes. Mirrors the idempotency contract used in `netbox-ai-ingest`.
- Rendering targets a local file (`build/<hostname>.cfg`), not a live device,
  keeping Slice 1 dependency-free and safe to run with `--check`.

## Environment

A `.venv` (gitignored) already has `ansible-core` 2.21.1 and the full
community collection bundle installed, including `netbox.netbox` (for Slice
2's `nb_inventory` plugin). Activate it before running any `ansible-*`
command:

```
source .venv/bin/activate
```

## Commands (once Slice 1 exists)

```
ansible-playbook playbooks/render_config.yml            # render all synthetic devices
ansible-playbook playbooks/render_config.yml --check    # check-mode dry run
```

There is no CI, linting, or test suite by design at this stage (SPEC.md
non-goals: no CI/CD, no molecule testing for Slice 1).
