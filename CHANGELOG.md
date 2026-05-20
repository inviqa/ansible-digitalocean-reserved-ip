# Changelog

## [Unreleased]

### Changed

- Switched Jenkins live tests to the shared
  `digitalocean-ansible-roles-oauth-token` credential used by DigitalOcean
  Ansible role repositories.

## [0.1.0] - 2026-05-13 First release

### Added

- Initial release of the `inviqa.digitalocean_reserved_ip` Ansible role for
  managing a DigitalOcean Reserved IP attached to an existing droplet.
- Reserved-IP-only role interface, including dedicated variables and exported
  facts for the assigned Reserved IP, anchor IP, and anchor gateway metadata.
- Optional outbound routing through the Reserved IP, enabled by default with
  `enable_reserved_ip_outbound_routing`.
- Debian and Ubuntu routing support with immediate route updates and persistent
  netplan configuration.
- CentOS-family routing support through NetworkManager via
  `community.general.nmcli`.
- SSH handoff support so Ansible reconnects through the Reserved IP before
  default route changes are applied.
- Per-host temporary `known_hosts` handling for live test runs to avoid
  host-key races during parallel execution.
- Live DigitalOcean integration test harness covering Debian 13, Ubuntu 24.04
  LTS, and CentOS Stream 10.
- Test provisioning and cleanup through the maintained `digitalocean.cloud`
  collection, with cleanup compatibility for older interrupted test resources.
- Outbound routing verification in the live tests, asserting that droplet
  egress matches the assigned Reserved IP.
- Test configuration support through environment variables or the gitignored
  `tests/test_variables.yml` file.
- Repository documentation covering installation, variables, exported facts,
  routing behavior, examples, maintainer details, and live test operation.
- Jenkins pipeline for the private CI job, using the shared Inviqa Ansible
  Docker image to install Ansible collections, run syntax checks, execute live
  tests, and always run cleanup.
- Jenkins credential placeholders for the DigitalOcean API token, DigitalOcean
  SSH key IDs, live-test private key, and Slack notification token.
- Jenkins live-test inventory selection restricted to the documented repository
  inventories.
- Failure-only Jenkins notifications to the `ops-integrations` Slack channel.
- Local validation policy and helper documentation for Markdown, YAML,
  Ansible, and repository-maintenance checks.
