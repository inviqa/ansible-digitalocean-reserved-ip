# Ansible role: DigitalOcean Reserved IP

Manage a DigitalOcean Reserved IP for an existing droplet and expose the
network metadata needed by callers to align routing with the target host.

This repository starts as a copy of the historical
`ansible-digitalocean-floating-ip` work, but it is now packaged and documented
as a Reserved IP role.

## Table of contents

- [Overview](#overview)
- [What it does](#what-it-does)
- [Current behavior notes](#current-behavior-notes)
- [Requirements](#requirements)
- [Installation](#installation)
- [Variables](#variables)
- [Exported facts](#exported-facts)
- [Outbound routing](#outbound-routing)
- [Examples](#examples)
- [Testing](#testing)
- [Development notes](#development-notes)
- [Continuous integration](#continuous-integration)
- [Release workflow](#release-workflow)
- [Repository guidance](#repository-guidance)
- [Maintainer](#maintainer)
- [Support](#support)
- [Repository](#repository)
- [License](#license)

## Overview

The role is intentionally narrow in scope:

- manage a Reserved IP for an existing droplet
- expose the reserved-address and anchor-routing metadata needed by callers
- avoid legacy host-level NAT while keeping Reserved IP default-route
  persistence explicit and opt-out

## What it does

- attaches an existing Reserved IP to a droplet, or creates one on the fly
- exposes the assigned address as Ansible facts
- retrieves the droplet anchor address and anchor gateway metadata
- configures outbound routing through the Reserved IP via the anchor gateway (when enabled)
- does not install legacy SMTP or general TCP `iptables` SNAT rules

## Current behavior notes

- Preferred terminology in this repo is **Reserved IP**.
- This role uses Reserved-IP-only variable names and exported facts.
- DigitalOcean exposes the current Reserved IP API through `/reserved_ips`
  REST endpoints and `reserved_ip` / `reserved_ips` response fields.
- The role assumes outbound traffic should follow the default route via the
  anchor gateway rather than host-level SNAT rules.

## Requirements

- Ansible Core $\ge 2.13$
- a DigitalOcean API token with permission to manage droplets and Reserved IPs
- `community.general` collection (used for RedHat-family NetworkManager
  gateway persistence via `community.general.nmcli`)
- gathered host facts when `digitalocean_reserved_ip_enable_outbound_routing` is enabled

## Installation

### Current repository role names

- Local development role name in this repository:
  `inviqa_digitalocean_reserved_ip`
- Intended Ansible Galaxy FQCN from the current repository metadata:
  `inviqa.digitalocean_reserved_ip`

For generic downstream playbook examples, this README uses the simple role name
`digitalocean_reserved_ip`.

If you install the role under a different local name or through a namespace,
use the role name or FQCN that matches your own installation method.

### Publication status

The repository is being prepared for publication. Until that release exists,
it may still be consumed from a local checkout during migration work.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `digitalocean_reserved_ip_api_base_url` | `https://api.digitalocean.com/v2` | Base DigitalOcean API URL. |
| `digitalocean_reserved_ip_api_token` | `{{ enc_do_v2_api_key }}` | API token used for DigitalOcean API requests. |
| `digitalocean_reserved_ip_droplet_id` | `{{ do.droplet.id \| mandatory }}` | Target droplet identifier. |
| `digitalocean_reserved_ip_address` | `""` | Preferred Reserved IP input. Leave empty to allocate one automatically. |
| `digitalocean_reserved_ip_metadata_anchor_ipv4_gateway_url` | `http://169.254.169.254/metadata/v1/interfaces/public/0/anchor_ipv4/gateway` | Metadata endpoint used to discover the anchor gateway. |
| `digitalocean_reserved_ip_metadata_public_ipv4_gateway_url` | `http://169.254.169.254/metadata/v1/interfaces/public/0/ipv4/gateway` | Metadata endpoint used to discover the original public gateway. |
| `digitalocean_reserved_ip_enable_outbound_routing` | `true` | Configure outbound routing through the Reserved IP via the anchor gateway. |

The role still accepts the older `digital_ocean_*` input names as fallback
aliases during migration, but new playbooks should use the
`digitalocean_reserved_ip_*` names above.

## Exported facts

The role exposes these Reserved-IP facts for callers:

- `do_droplet_reserved_ip_address`
- `reserved_ip_is_assigned_to_this_droplet`
- `anchor_ip_associated_to_droplet`
- `anchor_gateway_associated_to_droplet`

## Outbound routing

When `digitalocean_reserved_ip_enable_outbound_routing` is `true` (the default), the role
configures the droplet to route all outbound traffic through the Reserved IP
using the anchor gateway, following the
[official DigitalOcean Reserved IP documentation](https://docs.digitalocean.com/networking/reserved-ips/#reserved-ips-and-outbound-traffic).

The routing flow below shows the role-specific SSH handoff and operating-system
branch before the immediate and persistent route changes.

```mermaid
flowchart LR
  metadata["Retrieve anchor gateway"]
  handoff["Switch Ansible to Reserved IP"]
  os_family{"OS family?"}
  immediate["Replace default route now"]
  netplan["Persist route in netplan"]
  nmcli["Persist gateway with NetworkManager"]
  reboot["Reboot applies route"]
  verify["Verify egress IP"]

  metadata --> handoff
  handoff --> os_family
  os_family -->|Debian or Ubuntu| immediate
  immediate --> netplan
  os_family -->|Red Hat| nmcli
  nmcli --> reboot
  netplan --> verify
  reboot --> verify
```

The routing configuration is applied in two ways:

### Immediate routing

Before changing the default route, the role waits for SSH on the Reserved IP,
switches Ansible's management address to that Reserved IP, and resets the SSH
connection. To avoid host-key races during parallel runs, the handoff uses a
per-host temporary `known_hosts` file on the control machine.

On Debian and Ubuntu systems, the role then schedules the route change via
`ip route` commands using the detected primary interface:

```bash
ip route del 0/0
ip route add default via <anchor-gateway> dev <primary-interface>
```

This takes effect immediately on the target droplet while keeping the control
connection aligned with the Reserved IP.

On Red Hat-family systems, the role does not attempt the same in-band route
cutover over SSH. Instead, it switches Ansible management to the Reserved IP
first, persists the gateway through NetworkManager, and lets the reboot apply
path complete the routing change safely.

### Persistent routing

The role also applies persistent routing configuration through the system's
network configuration to ensure routing survives reboot:

- **Debian/Ubuntu**: Updates `/etc/netplan/50-cloud-init.yaml` with routes
  section pointing the default route for the detected primary interface to the
  anchor gateway
- **CentOS/RHEL**: Uses `nmcli` to update the active NetworkManager connection
  for the detected primary interface with `ipv4.gateway=<anchor-gateway>` and
  reboots when that persisted gateway changes, following the safer
  CentOS/RHEL-compatible workflow validated in the live tests

### Verification

To verify that outbound routing is working correctly, query your droplet's
outbound public IP:

```bash
curl -4 https://icanhazip.com/
```

The returned IP should match your Reserved IP address.

### Disabling routing

Set `digitalocean_reserved_ip_enable_outbound_routing: false` to skip routing configuration.
The role will still allocate, attach, and expose the Reserved IP facts, but
will not modify the droplet's routing table.

## Examples

```yaml
---
- name: Attach an existing Reserved IP
  hosts: digitalocean_droplets
  vars:
    reserved_ip_role_name: digitalocean_reserved_ip
  tasks:
    - name: Manage a Reserved IP with your installed role name
      ansible.builtin.include_role:
        name: "{{ reserved_ip_role_name }}"
      vars:
        digitalocean_reserved_ip_api_token: "{{ lookup('env', 'DIGITAL_OCEAN_API_TOKEN') }}"
        digitalocean_reserved_ip_droplet_id: "{{ digitalocean_droplet.id }}"
        digitalocean_reserved_ip_address: "203.0.113.10"

- name: Allocate a Reserved IP on the fly
  hosts: digitalocean_droplets
  vars:
    reserved_ip_role_name: digitalocean_reserved_ip
  tasks:
    - name: Allocate and attach a Reserved IP with your installed role name
      ansible.builtin.include_role:
        name: "{{ reserved_ip_role_name }}"
      vars:
        digitalocean_reserved_ip_api_token: "{{ lookup('env', 'DIGITAL_OCEAN_API_TOKEN') }}"
        digitalocean_reserved_ip_droplet_id: "{{ digitalocean_droplet.id }}"
```

If you install the published role under its current namespace, replace
`digitalocean_reserved_ip` with `inviqa.digitalocean_reserved_ip`.

## Testing

The current test workflow is documented in [docs/testing.md](docs/testing.md).
It covers Workspace commands, DigitalOcean live tests, Jenkinsfile lint,
cleanup, the Workspace CLI install command, and direct Ansible execution with
`--limit`.

## Development notes

- `tests/` contains the copied integration harness and is being refreshed for
  the new role name.
- `tests/README.md` points to the maintained testing documentation in
  `docs/testing.md`.
- `workspace.yml` provides the preferred local test and release command surface:
  `ws ansible-lint`, `ws syntax`, `ws test-live all`, and the release
  preflight commands documented below.
- The repo is being prepared for Galaxy publication, so the metadata and
  documentation are intentionally kept explicit.

## Continuous integration

- `Jenkinsfile` defines the private Jenkins CI entrypoint for this role.
- `docs/jenkins-ci.md` documents the Jenkins build parameters, where
  maintainers set them with **Build with Parameters**, required credentials,
  Workspace environment, validation stages, live DigitalOcean test stage,
  release preflight, and cleanup behavior.

## Release workflow

- `docs/ansible-galaxy-release.md` documents the GitHub release and Ansible
  Galaxy import flow.
- Workspace commands keep local and Jenkins release behavior aligned:
  `ws github release check`, `ws github release publish`,
  `ws ansible-galaxy check-token`, `ws ansible-galaxy info`, and
  `ws ansible-galaxy publish`.
- Galaxy publishing reads `ANSIBLE_GALAXY_TOKEN` or
  `ansible.galaxy.token` from `workspace.override.yml`; Jenkins uses the
  `ansible-roles-galaxy-token` Secret text credential.

## Repository guidance

- `AGENTS.md` defines repository-specific editing, linting, and documentation
  expectations for AI coding agents working in this role.
- `.ansible/` is treated as generated or vendored content and should not be
  hand-edited; update the repository source files instead.

## Maintainer

- Maintainer: Marco Massari Calderone `<marco.massari-calder@inviqa.com>`
- Copyright holder: Inviqa UK Ltd

## Support

- For current maintenance and publication work, contact the maintainer above.
- If and when the role is published publicly, issue-tracking and support paths
  should be documented alongside the published source.

## Repository

- Repository URL: <https://github.com/marcomc/ansible-digitalocean-reserved-ip>
- Publication status: pending Ansible Galaxy publication

## License

MIT
