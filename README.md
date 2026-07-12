# netbox-ansible-automation

**NetBox-driven Ansible network automation** — consumes NetBox as the network
source of truth (dynamic inventory), renders device configuration from Jinja2
templates, and applies it through idempotent playbooks and roles.

Companion to **[netbox-ai-ingest](https://github.com/craignetworking-dev/netbox-ai-ingest)**:
that project populates and structures the NetBox source of truth; this one
consumes it to drive device configuration. Together they form a complete
source-of-truth → config-automation pipeline:

```
netbox-ai-ingest            NetBox                  netbox-ansible-automation
(source-of-truth engine     (devices, roles,        (dynamic inventory +
 + AI ingestion)      ─────► sites, interfaces) ◄───  Jinja2 config rendering)
```

The two repositories integrate over the NetBox REST API — no shared code, a
clean interface boundary. This mirrors how modern network automation actually
works: a source of truth drives configuration generation, rather than
configuration living in static files.

## What this demonstrates

- **Source-of-truth-driven automation** — device inventory is pulled live from
  NetBox via the `netbox.netbox.nb_inventory` plugin and grouped by device role,
  rather than maintained as a static host list.
- **Idiomatic Ansible structure** — a reusable role (`tasks`/`templates`/`defaults`),
  `group_vars`/`host_vars` variable separation, and conditional Jinja2 templating.
- **Idempotency as a contract** — a second run against unchanged state makes zero
  changes (`changed=0`), the same discipline applied across both repositories.
- **Secure credential handling** — the NetBox API token is read from environment
  variables (via a git-ignored `.env`) and never committed. No secrets appear in
  any tracked file.

## How it works

1. **Dynamic inventory** (`inventory/netbox.yml`) — the NetBox inventory plugin
   queries NetBox live, returns device instances, and groups them by device role.
   Authentication is via the `NETBOX_API` and `NETBOX_TOKEN` environment
   variables; nothing sensitive is stored in the file.
2. **Variables** (`inventory/group_vars/`, `inventory/host_vars/`) — per-group and
   per-device configuration detail (domain, NTP, interfaces, VLANs) that joins
   onto the NetBox-sourced hosts.
3. **Role** (`roles/device_config/`) — a Jinja2 template (`templates/base.j2`)
   renders a device configuration, branching on interface mode
   (access / trunk / routed) to emit the correct switchport or routed-interface
   stanzas. The task renders each device's config to `build/<hostname>.cfg`.
4. **Playbook** (`playbooks/render_config.yml`) — applies the role across the
   inventory.

## Usage

Prerequisites: Python 3, Ansible, the `netbox.netbox` collection, and `pynetbox`
(plus `pytz`). A reachable NetBox instance with device instances defined.

```bash
# install dependencies
pip install ansible pynetbox pytz
ansible-galaxy collection install netbox.netbox

# configure credentials (never committed)
cp .env.example .env        # then edit with your NetBox URL and token
#   NETBOX_API=http://<your-netbox>:8000
#   NETBOX_TOKEN=<your-token>

# load env and render configs from NetBox-sourced inventory
set -a; source .env; set +a
ansible-playbook -i inventory/netbox.yml playbooks/render_config.yml

# rendered configs land in build/<hostname>.cfg
```

To run against the static synthetic inventory instead of NetBox (no NetBox
required):

```bash
ansible-playbook playbooks/render_config.yml   # uses inventory/hosts.ini
```

## Project status

Built in slices, each proven before the next:

- **Slice 1 — pure-Ansible synthetic config rendering.** Role + Jinja2 template +
  static synthetic inventory, rendering device configs to files. Proven
  idempotent. No external dependencies.
- **Slice 2 — NetBox dynamic inventory.** Devices pulled live from NetBox and
  grouped by role; the Slice 1 role renders configs from NetBox-sourced
  inventory, joined with `host_vars` config detail. Proven idempotent end to end.

See [`SPEC.md`](SPEC.md) for the full design, build order, and roadmap
(including the optional Slice 3: pushing rendered config to virtual devices).

## Data & safety

All device data in this repository is **synthetic** — fabricated hostnames, IPs
(RFC 1918), and configuration. No real network data, credentials, or topology
is included. The NetBox API token lives only in a git-ignored `.env`.

## Related

- **[netbox-ai-ingest](https://github.com/craignetworking-dev/netbox-ai-ingest)**
  — the NetBox source-of-truth provisioning engine and AI documentation-ingestion
  pipeline that this automation consumes.
