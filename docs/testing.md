# Test Harness

This document describes the DigitalOcean Reserved IP role test harness. Local
test operations run through Workspace, which owns the repository Docker Compose
environment.

## Table of Contents

- [Coverage](#coverage)
- [Setup](#setup)
- [Workspace Commands](#workspace-commands)
- [DigitalOcean Live Tests](#digitalocean-live-tests)
- [Jenkinsfile Lint](#jenkinsfile-lint)
- [Clean Up](#clean-up)
- [Outbound Routing Verification](#outbound-routing-verification)
- [Notes](#notes)

## Coverage

The default inventory currently exercises:

| Family | Image slug |
| --- | --- |
| Debian | `debian-13-x64` |
| CentOS family | `centos-stream-10-x64` |
| Ubuntu | `ubuntu-24-04-x64` |

Single-family runs use the `debian`, `centos`, and `ubuntu` inventory groups
through Ansible `--limit`.

## Setup

Install the Workspace CLI before running the test commands if `ws` is not
already available.

```bash
WS_VERSION=0.4.1
curl --output ./ws --location "https://github.com/my127/workspace/releases/download/${WS_VERSION}/ws"
chmod +x ws && sudo mv ws /usr/local/bin/ws
```

Live commands read local attributes from `workspace.override.yml`. Create it
from the example first:

```text
cp workspace.override.yml.example workspace.override.yml
```

Set `test.digitalocean.api_token`, `test.digitalocean.ssh_keys`, and
`test.digitalocean.project_name`. SSH key selectors can be IDs, fingerprints,
or names. In `workspace.override.yml`, `test.digitalocean.ssh_keys` is a list.
When passed through environment variables or Jenkins credentials, multiple
selectors can be comma or newline separated. The selected DigitalOcean SSH keys
must match private keys loaded in the forwarded SSH agent.
The harness validates that match before creating a Droplet and uses the
selected DigitalOcean public key to steer SSH agent authentication.

The playbooks still support the legacy local override file for direct Ansible
runs:

```text
cp tests/test_variables.example.yml tests/test_variables.yml
```

For direct Ansible runs, set your DigitalOcean API token and at least one SSH
key ID, fingerprint, or name:

```yaml
do_test_api_token: "dop_v1_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
do_ssh_keys:
  - "12345678"
```

To list available SSH key IDs and fingerprints with `doctl`:

```text
doctl compute ssh-key list --format ID,Name,FingerPrint
```

Load the matching private key into your SSH agent before running the playbooks:

```text
ssh-add ~/.ssh/<your-private-key>
```

The harness creates droplets with the `digitalocean.cloud` collection and
connects to them as `root`, so `do_ssh_keys` must reference a DigitalOcean SSH
key whose private key is available for root login.

If `DIGITAL_OCEAN_API_TOKEN` is exported in the current shell, it takes
precedence over `do_test_api_token` in `test_variables.yml`.
Likewise, `DIGITAL_OCEAN_SSH_KEYS` from Workspace or the shell takes precedence
over `do_ssh_keys` in `test_variables.yml`.

## Workspace Commands

The preferred local entrypoint is Workspace:

```text
ws
```

Useful commands:

```text
ws syntax
ws ansible-lint
ws lint-jenkinsfile
ws test-live all
ws test-live debian
ws test-live centos
ws test-live ubuntu
ws cleanup-live all
ws cleanup-live debian
ws cleanup-live centos
ws cleanup-live ubuntu
```

Both `ws console` and `ws ansible-playbook` load live-test environment values
from `workspace.override.yml` and forward them into the `console` container.
Other Workspace commands compose those entrypoints instead of repeating Docker
environment wiring.

Use `ws syntax` for syntax checks and `ws ansible-lint` for role linting.

## DigitalOcean Live Tests

Run the full DigitalOcean-backed end-to-end matrix:

```text
ws test-live all
```

Run one family only:

```text
ws test-live debian
ws test-live centos
ws test-live ubuntu
```

For direct Ansible runs, execute the playbooks from the repository root:

```text
ansible-playbook -i tests/inventory tests/playbook.yml
```

Run one family only:

```text
ansible-playbook -i tests/inventory tests/playbook.yml --limit debian
ansible-playbook -i tests/inventory tests/playbook.yml --limit centos
ansible-playbook -i tests/inventory tests/playbook.yml --limit ubuntu
```

`tests/inventory` disables local SSH proxy configuration for the temporary
DigitalOcean droplets so the live test connects directly to the provisioned
hosts.

The live playbook:

- creates one small DigitalOcean droplet for each target OS
- waits for SSH
- installs Python if the image needs it
- allocates or attaches the Reserved IP
- validates SSH agent access with the selected DigitalOcean SSH key
- verifies outbound routing through the Reserved IP

The live-test path creates real provider resources, validates them, and then
hands off to the cleanup playbook whether the validation succeeds or fails.

```mermaid
flowchart LR
  accTitle: DigitalOcean Reserved IP live-test flow
  accDescr: Shows the Workspace live-test sequence from setup through cleanup.
  setup["Load local test credentials"] --> provision["Create DigitalOcean droplets"]
  provision --> ssh["Wait for SSH and install Python"]
  ssh --> role["Run Reserved IP role"]
  role --> verify["Verify Reserved IP routing"]
  verify --> cleanup["Run cleanup playbook"]
  role -->|Failure| cleanup
  cleanup --> droplets["Delete droplets and Reserved IPs"]
```

## Jenkinsfile Lint

Validate the repository `Jenkinsfile` with the Workspace Jenkins lint controller
and the Jenkins Declarative Pipeline linter:

```text
ws lint-jenkinsfile
```

This command starts the Workspace `console` and `jenkins-lint` Compose services
and runs the helper inside the `console` container.

## Clean Up

DigitalOcean live tests run cleanup automatically. If a live run is interrupted,
destroy all test droplets and Reserved IPs with Workspace:

```text
ws cleanup-live all
```

Clean up one family only:

```text
ws cleanup-live debian
ws cleanup-live centos
ws cleanup-live ubuntu
```

For direct Ansible runs:

```text
ansible-playbook -i tests/inventory tests/playbook_cleanup.yml
```

If a single-family run fails mid-flight, clean up with the matching limit
before retrying:

```text
ansible-playbook -i tests/inventory tests/playbook_cleanup.yml --limit debian
```

## Outbound Routing Verification

The test playbooks verify that outbound routing through the Reserved IP is
working correctly by querying `https://icanhazip.com/` and asserting that the
returned outbound IP matches the assigned Reserved IP address.

During the SSH handoff, the role switches Ansible management to the Reserved IP
before the route cutover and uses a per-host temporary `known_hosts` file on
the control machine so parallel matrix runs do not race on shared host-key
state.

On CentOS-family test droplets, the role may reboot once after updating the
active NetworkManager connection with `nmcli connection modify ...
ipv4.gateway ...` so the persisted gateway change is applied cleanly.

If routing verification fails:

1. Check that the droplet has network connectivity.
2. Verify DNS resolution is working on the droplet.
3. Confirm that the anchor gateway metadata was retrieved correctly.
4. Check the droplet's routing table: `ip route show`.
5. On the droplet, manually test the Reserved IP endpoint:
   `curl -4 https://icanhazip.com/`.

To skip routing configuration, add
`digitalocean_reserved_ip_enable_outbound_routing: false` to the playbook
variables.

## Notes

- The harness provisions real droplets and Reserved IPs, so it incurs cost.
- No AWS credentials are required; only DigitalOcean credentials are used.
- The harness connects to test droplets as `root`.
- `workspace.override.yml` and `tests/test_variables.yml` must stay untracked
  because they may contain local credentials.
- Live-test droplets are assigned to the DigitalOcean project configured by
  `test.digitalocean.project_name` in `workspace.override.yml`.
- Test droplets are named `ansible-digitalocean-reserved-ip-<inventory-name>`
  and tagged with `ANSIBLE-TEST` for cleanup.
- Cleanup also recognises the previous `<inventory-name>-test-with-ansible`
  droplet names so interrupted older test runs can be removed.
- If a test run fails mid-flight, run the cleanup playbook before starting the
  next run.
- If you get `Permission denied (publickey)`, confirm the private key matching
  `do_ssh_keys` is loaded in your SSH agent.
- If you get a `401 Unauthorized` error, verify `do_test_api_token` in
  `test_variables.yml` or export a valid `DIGITAL_OCEAN_API_TOKEN`. The harness
  checks `/v2/account` before provisioning, so an invalid token fails early
  with a clear authentication message.
