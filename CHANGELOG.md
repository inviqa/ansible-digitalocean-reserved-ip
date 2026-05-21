# Changelog

## [0.2.0] - 2026-05-20

### CI and Workspace

- Moved Jenkins validation and release publication onto reusable Workspace
  commands for linting, syntax checks, DigitalOcean live testing, GitHub
  release checks and publication, and Ansible Galaxy token, status, and import
  actions.
- Namespaced helper commands under `ws ansible lint`, `ws ansible syntax`,
  `ws ansible playbook`, and `ws ansible galaxy <action>`, with grouped
  Workspace usage help for Ansible, Galaxy, config, GitHub, global, and secret
  command groups.
- Kept Ansible, Galaxy, GitHub, and provider CLIs inside the Workspace
  `console` boundary, including offline `ansible-lint`, isolated container
  Ansible cache paths, and simple-command rejection for quoted
  `ws console <command>` snippets.
- Standardized Jenkins credentials and release environment names on
  `GITHUB_TOKEN`, `DIGITAL_OCEAN_API_TOKEN`, `DIGITAL_OCEAN_SSH_KEYS`, the
  shared `ansible-roles-galaxy-token`, and the
  `digitalocean-ansible-roles-oauth-token` live-test credential, while keeping
  `DO_OAUTH_TOKEN` as a local DigitalOcean API fallback.
- Kept Jenkins publication and live-test operator choices as build parameters,
  with centralized top-level credential bindings, safe `full-cycle` live tests,
  and a second idempotent cleanup safety net.

### Role and Live Tests

- Renamed role input variables to the `digitalocean_reserved_ip_*` prefix while
  keeping fallback support for previous `digital_ocean_*` names.
- Reorganized the role task flow into focused files for Reserved IP assignment,
  outbound routing preparation, immediate route updates, Netplan,
  NetworkManager, SSH handoff, reporting, and cleanup.
- Resolved live-test DigitalOcean SSH key selectors through the API before
  Droplet creation so IDs, fingerprints, names, list values, and environment
  strings are accepted consistently, with Workspace-provided selectors taking
  precedence for live tests.
- Added live-test SSH agent validation against DigitalOcean MD5 fingerprints,
  made non-secret SSH selector diagnostics visible, and kept token-bearing API
  checks hidden behind `no_log`.
- Added optional Workspace-driven DigitalOcean project assignment for live-test
  Droplets, kept tracked examples inert by default, and set Jenkins to assign
  live-test Droplets to `Inviqa Sandbox`.
- Required explicit live-test phases:
  `ws test-live provision <target>`, `ws test-live cleanup <target>`, and
  `ws test-live full-cycle <target>`.
- Consolidated live-test targeting on `tests/inventory` plus `--limit`
  examples, removing duplicate single-family inventory files.
- Avoided `reset_connection` conditional warnings during outbound routing
  tests and waited for post-cutover SSH readiness before continuing
  Debian-family routing configuration, with clearer task names for TCP and SSH
  readiness checks.

### Documentation and Release Readiness

- Updated README, testing, Jenkins CI, and Ansible Galaxy release documentation
  for Workspace commands, Jenkins parameters, current credential IDs, release
  checks, direct Ansible variables, and live-test flow diagrams.
- Corrected the `0.2.0` changelog heading to a plain `YYYY-MM-DD` date and
  documented that release-prep changelog entries should stay compact in the
  latest concrete release section when no `Unreleased` section exists.
- Moved test harness guidance into `docs/testing.md`, left `tests/README.md` as
  a pointer, and documented all Workspace override attributes used by live
  tests and release commands.
- Clarified agent guidance for Jenkins parameters, credential bindings,
  Workspace commands, dynamic includes around `reset_connection`, and
  reviewer-readable changelog structure.
- Added a README diagram for the Reserved IP outbound routing handoff and
  operating-system-specific persistence flow.
- Sanitized the tracked live-test variable file so real credentials are read
  from Workspace overrides, environment variables, `tests/test_variables.yml`,
  or Jenkins credentials.

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
