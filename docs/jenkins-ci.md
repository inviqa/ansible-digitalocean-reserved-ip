# Jenkins CI integration

This repository includes a `Jenkinsfile` for running the same Ansible
validation and live test flow used during local maintenance.

## Pipeline coverage

The pipeline runs inside a Jenkins-native Docker agent and executes:

- Ansible collection installation from `tests/requirements.yml`
- Syntax checks for `tests/playbook.yml` and `tests/playbook_cleanup.yml`
- The live DigitalOcean test matrix with `tests/playbook.yml`
- Cleanup with `tests/playbook_cleanup.yml`, even when the live test stage fails
- Failure notification to the `ops-integrations` Slack channel

The live test stage provisions real DigitalOcean droplets and Reserved IPs, so
it incurs provider cost while it runs.

Markdown, YAML, and `ansible-lint` checks are intentionally not part of the
Jenkins pipeline. They are local pre-handoff checks only, with Markdown using
the repository `.markdownlint.json` file.

## Jenkins Docker agent

The Jenkinsfile follows the Jenkins-native Docker workflow used by several
local projects. Jenkins runs the pipeline inside the shared Inviqa Ansible
image:

```text
quay.io/inviqa_images/ansible:2.15-python3.10-trixie
```

That image already provides Python and Ansible. The pipeline installs the
Ansible collections required by the live test harness, then runs the Ansible
commands directly.

This keeps the role aligned with the shared Inviqa Ansible image without
requiring a project-specific Dockerfile or `docker compose` inside the
Jenkinsfile.

The Jenkins job does not install extra Python, YAML lint, Ansible lint, or
Markdown tooling.

## Jenkins setup

Create a pipeline job that uses the repository `Jenkinsfile`.

Recommended Jenkins configuration:

- Agent label: `linux-amd64`
- Required tools on the agent:
  - Docker
  - `ssh-agent`
- Required Jenkins credential:
  - String credential containing the DigitalOcean API token
  - String credential containing the DigitalOcean SSH key IDs or fingerprints
    used for test droplets
  - SSH private key credential matching the DigitalOcean SSH key configured for
    test droplets
  - Slack token credential used for failure notifications

The credential ID placeholders are defined at the top of `Jenkinsfile`:

| Placeholder | Jenkins credential type | Purpose |
| --- | --- | --- |
| `digitalocean-ansible-roles-oauth-token` | Secret text | Shared DigitalOcean API token for Ansible role live tests. |
| `digitalocean-reserved-ip-ssh-key-ids` | Secret text | Comma or newline separated DigitalOcean SSH key IDs or fingerprints. |
| `digitalocean-reserved-ip-test-ssh-private-key` | SSH username with private key | Private key loaded for live test droplet access. |
| `inviqa-slack-integration-token` | Secret text | Slack token used for Jenkins failure notifications. |

## Parameters

| Parameter | Default | Purpose |
| --- | --- | --- |
| `RUN_LIVE_TESTS` | `true` | Enables the live DigitalOcean integration test stage. |
| `TEST_INVENTORY` | `tests/inventory` | Inventory choice used by the live test and cleanup playbooks. |

When `RUN_LIVE_TESTS` is enabled, all three Jenkins credentials above must
exist.

`TEST_INVENTORY` is restricted to these repository inventories:

- `tests/inventory`
- `tests/inventory-debian`
- `tests/inventory-centos`
- `tests/inventory-ubuntu`

The pipeline sends failure notifications only. Successful builds do not post to
Slack.

## Local equivalent

The Jenkinsfile mirrors this local sequence:

```text
docker run --rm -v "$PWD:/workspace" -w /workspace \
  -e ANSIBLE_COLLECTIONS_PATH=/workspace/.ansible/collections:/home/ansible/.ansible/collections:/usr/share/ansible/collections \
  -e ANSIBLE_ROLES_PATH=/workspace/tests/roles:/workspace/.ansible/roles:/home/ansible/.ansible/roles \
  quay.io/inviqa_images/ansible:2.15-python3.10-trixie \
  sh -c '
    ansible-galaxy collection install -r tests/requirements.yml -p .ansible/collections &&
    ansible-playbook --syntax-check -i tests/inventory tests/playbook.yml &&
    ansible-playbook --syntax-check -i tests/inventory tests/playbook_cleanup.yml &&
    ansible-playbook -i tests/inventory tests/playbook.yml &&
    ansible-playbook -i tests/inventory tests/playbook_cleanup.yml
  '
```
