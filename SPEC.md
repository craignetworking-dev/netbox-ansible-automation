# netbox-ansible-automation — SPEC

NetBox-driven Ansible network automation. Consumes NetBox as the source of truth
(dynamic inventory), renders device configuration from Jinja2 templates, and
applies it via playbooks and roles. Companion to `netbox-ai-ingest`: that repo
populates and structures the NetBox source of truth; this repo consumes it to
drive device configuration. The two integrate over the **NetBox API**, not by
sharing code.

Build slice by slice, prove each before the next, and do not let the NetBox
integration or a virtual-device lab turn into a rabbit hole (see the fallback
ladder).

## Why this repo exists

- **Closes a market-wide skills gap.** Ansible appears as a required/《basic》
  qualification across network-automation and AIOps roles (Itential NAE III and
  SDE II, most network-automation postings). A focused, working Ansible repo
  converts a hard "no" on screening questions into a demonstrable "yes."
- **Completes the portfolio story.** `netbox-ai-ingest` shows source-of-truth
  engineering + applied AI. This repo shows the config-automation layer that
  consumes it. Together: "I architect network automation as composable
  components with a clean NetBox-API seam" — a systems-design signal, not just
  a tool.
- **Mirrors the real world.** NetBox-as-source-of-truth → Ansible-as-config-push
  is the canonical modern network-automation pattern. Building it demonstrates
  fluency in how the ecosystem actually operates.

## The pairing (how the two repos work hand in hand)

```
netbox-ai-ingest          NetBox                 netbox-ansible-automation
----------------          ------                 -------------------------
provisioning engine  -->  source of truth  <--   dynamic inventory (nb_inventory)
AI ingestion pipeline     (devices, roles,       Jinja2 config templates
                           sites, interfaces)    playbooks + roles --> device config
```

The seam is the NetBox REST API. This repo reads NetBox; it never imports
`netbox-ai-ingest` code. Each repo's README links to the other and states the
relationship.

## Hard rules

- **No real-world data. Ever.** Synthetic devices, synthetic configs, synthetic
  inventory. Real hostnames/IPs/topology/credentials never enter the repo.
  (Real device credentials and the NetBox token live in `.env` / vault, already
  gitignored.)
- **Credential custody.** The NetBox API token and any device credentials live
  in `.env` or Ansible Vault — never committed. The `.gitignore` enforces this;
  it was the first commit, before any secret existed.
- **Reference by name/variable, never hardcode** IPs or credentials in playbooks
  or templates. Config comes from variables (later: from NetBox).
- **Idempotent by design.** Ansible modules should be idempotent; a second run
  against unchanged state reports no changes. (Mirrors the netbox-ai-ingest
  idempotency contract.)

## Fallback ladder (anti-rabbit-hole discipline)

Build tiers in order. Each tier is a complete, committable increment. Do NOT
start at tier 2 or 3 — the value is provable at tier 1, and tiers 2–3 are where
time disappears.

- **Tier 1 — pure Ansible, synthetic, no external deps (SLICE 1).**
  Playbook + role + Jinja2 template(s) + a static synthetic inventory, run in
  `--check` / mock mode. Proves you write real, idiomatic Ansible. No NetBox, no
  lab, nothing to get stuck on.
- **Tier 2 — NetBox dynamic inventory (SLICE 2, the differentiator).**
  Replace the static inventory with the `netbox.netbox.nb_inventory` plugin
  pointed at a live NetBox (the one netbox-ai-ingest populates). Proves the
  source-of-truth-driven pattern — the money shot.
- **Tier 3 — real device push (SLICE 3, optional, timeboxed).**
  Stand up 1–2 virtual devices (containerlab with Arista cEOS or Nokia SR Linux)
  and actually push rendered config. Only if tiers 1–2 went fast. Timebox hard;
  if device connectivity fights back, stop and ship tiers 1–2.

## Slice 1 scope (Tier 1 — pure Ansible on synthetic data)

Build exactly this, prove it, commit it. No NetBox, no lab.

### Structure
```
ansible.cfg                 # local config (inventory path, roles path, no host key check for lab)
inventory/hosts.ini         # static SYNTHETIC inventory (2–3 fake devices, grouped)
group_vars/all.yml          # shared synthetic vars (e.g. domain, ntp servers)
group_vars/<group>.yml      # per-group vars (e.g. access vs core settings)
host_vars/<host>.yml        # per-device synthetic vars (hostname, mgmt ip, interfaces)
playbooks/render_config.yml # the playbook: apply the role across inventory
roles/
  device_config/
    tasks/main.yml          # render the template to an output file (check-mode friendly)
    templates/base.j2       # Jinja2 device config template
    defaults/main.yml       # role default vars
README.md                   # what it is, how to run, link to netbox-ai-ingest
```

### What it does
- Defines 2–3 **synthetic** devices in a static inventory, grouped (e.g.
  `access`, `core`), with per-host/group variables (hostname, mgmt IP,
  interfaces, VLANs) — all fabricated.
- A `device_config` role with a **Jinja2 template** that renders a plausible
  device config (hostname, mgmt interface, a couple of interface/VLAN stanzas,
  NTP/domain) from those variables.
- A **playbook** that runs the role across the inventory, rendering each device's
  config to an output file (e.g. `build/<hostname>.cfg`) — no real device needed.
- Runs cleanly with `ansible-playbook playbooks/render_config.yml` and supports
  `--check`.

### Definition of done (Slice 1)
- [ ] `ansible-playbook playbooks/render_config.yml` runs green against the
      synthetic inventory with no errors.
- [ ] Each device's config renders correctly from its variables (spot-check the
      generated `build/<hostname>.cfg` — right hostname, IP, interfaces).
- [ ] Idempotent / re-runnable: a second run produces identical output.
- [ ] Real Ansible structure present: a **role** (not just a flat playbook),
      **Jinja2 templating**, **inventory + group_vars/host_vars** separation.
- [ ] No secrets, no real data; `git status` shows nothing that should be ignored.
- [ ] README explains what it is, how to run it, and links to netbox-ai-ingest.
- [ ] Committed: `Slice 1: pure-Ansible synthetic config rendering (playbook + role + Jinja2)`.

## Non-goals (do NOT do in Slice 1)

- No NetBox integration (that is Slice 2 — the dynamic inventory plugin).
- No live/virtual devices, no containerlab, no real config push (Slice 3).
- No Ansible Vault yet (no real secrets exist in Slice 1; add vault when Slice 2
  introduces the NetBox token — though the token goes in `.env`, not the repo).
- No real device credentials, IPs, hostnames, or topology.
- No CI/CD, no molecule testing (nice later, not now).

## Slice 2 preview (Tier 2 — the differentiator, next)

Replace `inventory/hosts.ini` with `inventory/netbox.yml` configuring the
`netbox.netbox.nb_inventory` plugin against the NetBox that netbox-ai-ingest
populates. Devices, roles, sites, and primary IPs flow from NetBox into Ansible
automatically. Requires the `netbox.netbox` collection and a NetBox API token in
`.env` (never committed). This is the slice that proves source-of-truth-driven
automation — the reason the two repos are a system, not two toys.

## Review checkpoints (where to slow down)

- **Real role structure, not a flat playbook.** The value signal is a proper
  role (tasks/templates/defaults), inventory with group_vars/host_vars, and
  Jinja2 templating — that is what "I write real Ansible" looks like. A single
  monolithic playbook does not demonstrate the same competence.
- **Idempotency.** Re-running renders identical output. Ansible's design supports
  this; confirm the tasks are written to honor it.
- **Synthetic-data hygiene.** Every host, IP, and credential is fabricated.
  Confirm nothing real slipped in.
- **Check-mode friendliness.** Rendering to a local file (not pushing to a
  device) keeps Slice 1 dependency-free and `--check`-safe.
