# Changelog

## [0.2.0] - 2026-05-20

### CI

- Corrected Jenkins credential IDs for shared DigitalOcean live-test
  credentials.
- Reduced duplicate Jenkins credential bindings and standardized release
  credentials on the expanded `GITHUB_TOKEN`, `DIGITAL_OCEAN_API_TOKEN`, and
  `DIGITAL_OCEAN_SSH_KEYS` environment names.
- Kept `DO_OAUTH_TOKEN` as a backward-compatible local input fallback for
  DigitalOcean API credentials.
- Simplified the Workspace console Dockerfile requirement-copy paths to generic
  temporary filenames shared across sibling role repositories.

### Changed

- Switched Jenkins live tests to the shared
  `digitalocean-ansible-roles-oauth-token` credential used by DigitalOcean
  Ansible role repositories.
- Moved Jenkins validation and release publication onto the Workspace-backed
  flow shared with sibling Ansible role repositories.
- Added Workspace commands for DigitalOcean live testing, GitHub release
  checks, GitHub publication, Ansible Galaxy token checks, Galaxy status, and
  Galaxy import.
- Required explicit Workspace live-test targets with `ws test-live all`,
  `ws test-live debian`, `ws test-live centos`, or `ws test-live ubuntu`.
- Required the same explicit target shape for cleanup with
  `ws cleanup-live all`, `ws cleanup-live debian`, `ws cleanup-live centos`, or
  `ws cleanup-live ubuntu`.
- Replaced direct single-family Ansible inventory guidance with `--limit`
  examples against the canonical `tests/inventory`.
- Removed the duplicate single-family inventory files in favor of filtering
  `tests/inventory` by group.
- Simplified Jenkins configuration by replacing build parameters with fixed
  top-level environment defaults.
- Renamed the role input variables to the `digitalocean_reserved_ip_*` prefix
  while keeping fallback support for the previous `digital_ocean_*` names.
- Moved test harness guidance into `docs/testing.md` and left
  `tests/README.md` as a pointer to the maintained documentation.
- Aligned the testing documentation with the JumpCloud role structure by adding
  Jenkinsfile lint guidance and a live-test flow diagram.
- Documented the Ansible Galaxy release workflow and the Jenkins credentials
  used for GitHub and Galaxy publication.
- Sanitized the tracked live-test variable file so real credentials are read
  from Workspace overrides, environment variables, or Jenkins credentials.

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
