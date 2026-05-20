# Changelog

## Unreleased

## [0.2.0] - 2026-05-20a

### CI

- Corrected Jenkins credential IDs for DigitalOcean live-test credentials.
- Reduced duplicate Jenkins credential bindings and standardized release
  credentials on the expanded `GITHUB_TOKEN`, `DIGITAL_OCEAN_API_TOKEN`, and
  `DIGITAL_OCEAN_SSH_KEYS` environment names.
- Standardized the Jenkins Ansible Galaxy credential ID on the shared
  `ansible-roles-galaxy-token` credential.
- Kept `DO_OAUTH_TOKEN` as a backward-compatible local input fallback for
  DigitalOcean API credentials.
- Simplified the Workspace console Dockerfile requirement-copy paths to generic
  temporary filenames.
- Reduced the Workspace destroy timeout so local test containers stop faster.
- Added a Workspace-provided DigitalOcean project name for live-test droplets
  and assigned created test droplets to that project.
- Documented the preferred `workspace.override.yml` live-test configuration
  path alongside `tests/test_variables.yml` for direct Ansible execution.

### Changed

- Switched Jenkins live tests to the
  `digitalocean-ansible-roles-oauth-token` credential.
- Moved Jenkins validation and release publication onto reusable Workspace
  commands.
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
- Kept Jenkins publication and live-test operator choices as build parameters
  while keeping credential bindings centralized in the Jenkinsfile environment.
- Renamed the role input variables to the `digitalocean_reserved_ip_*` prefix
  while keeping fallback support for the previous `digital_ocean_*` names.
- Moved test harness guidance into `docs/testing.md` and left
  `tests/README.md` as a pointer to the maintained documentation.
- Added Jenkinsfile lint guidance and a live-test flow diagram to the testing
  documentation.
- Resolved live-test DigitalOcean SSH key selectors through the API before
  Droplet creation so IDs, fingerprints, names, list values, and environment
  strings are accepted consistently.
- Made Workspace-provided DigitalOcean SSH key selectors take precedence over
  local `tests/test_variables.yml` values during live tests.
- Added live-test SSH agent validation against DigitalOcean MD5 fingerprints
  and SSH key selection before Ansible connects to the created Droplet.
- Set live-test SSH arguments so temporary DigitalOcean droplets bypass local
  SSH proxy configuration during direct Ansible runs.
- Reorganized the role task flow so the main task file and outbound routing
  setup delegate to focused task files.
- Avoided Ansible `reset_connection` conditional warnings during live outbound
  routing tests and waited for the post-cutover SSH connection before
  continuing Debian-family routing configuration.
- Clarified the route-cutover task names so TCP port readiness and Ansible SSH
  session readiness are distinct in live-test output.
- Documented when agents must use dynamic Ansible task includes to keep
  conditional logic away from `reset_connection` meta tasks.
- Extracted shared live-test override loading and DigitalOcean credential
  validation into reusable test task files.
- Documented the Ansible Galaxy release workflow and the Jenkins credentials
  used for GitHub and Galaxy publication.
- Kept the Ansible Galaxy release documentation aligned with Jenkins
  parameters and Workspace release checks.
- Clarified where Jenkins maintainers set per-build pipeline parameters.
- Clarified agent guidance for keeping Jenkins parameters, credential bindings,
  and Workspace commands documented consistently.
- Added a README diagram for the Reserved IP outbound routing handoff and
  operating-system-specific persistence flow.
- Clarified the README origin note without referencing future publication.
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
