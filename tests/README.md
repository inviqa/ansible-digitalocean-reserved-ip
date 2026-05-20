
# Test harness

This directory contains the live DigitalOcean integration tests for the role.

## Contents

- [Coverage](#coverage)
- [Setup](#setup)
- [Run the tests](#run-the-tests)
- [Clean up](#clean-up)
- [Outbound routing verification](#outbound-routing-verification)
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

The preferred local entrypoint is Workspace. Install the Workspace CLI if `ws`
is not already available:

```text
WS_VERSION=0.4.1
curl --output ./ws --location "https://github.com/my127/workspace/releases/download/${WS_VERSION}/ws"
chmod +x ws && sudo mv ws /usr/local/bin/ws
```

Live commands read local attributes from `workspace.override.yml`. Create it
from the example first:

```text
cp workspace.override.yml.example workspace.override.yml
```

Set `test.digitalocean.api_token` and `test.digitalocean.ssh_keys`. The SSH key
selector can be comma separated when it is passed through Workspace or Jenkins.
The selected DigitalOcean SSH key must match a private key loaded in the
forwarded SSH agent.

The playbooks still support the legacy local override file for direct Ansible
runs:

```text
cp tests/test_variables.example.yml tests/test_variables.yml
```

For direct Ansible runs, set your DigitalOcean API token and at least one SSH
key ID or fingerprint:

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

> **Note:** If `DIGITAL_OCEAN_API_TOKEN` is exported in the current shell, it
> takes precedence over `do_test_api_token` in
> `test_variables.yml`.

## Run the tests

From the repository root, run the full matrix with Workspace:

```text
ws test-live all
```

Run one family only:

```text
ws test-live debian
ws test-live centos
ws test-live ubuntu
```

Use `ws syntax` for syntax checks and `ws ansible-lint` for role linting.

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

## Clean up

Destroy all test droplets and Reserved IPs with Workspace:

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

## Outbound routing verification

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

1. Check that the droplet has network connectivity
2. Verify DNS resolution is working on the droplet
3. Confirm that the anchor gateway metadata was retrieved correctly
4. Check the droplet's routing table: `ip route show`
5. On the droplet, manually test the reserved IP endpoint:
   `curl -4 https://icanhazip.com/`

To skip routing configuration, add
`enable_reserved_ip_outbound_routing: false` to the playbook variables.

## Notes

- The harness provisions real droplets and Reserved IPs, so it incurs cost.
- No AWS credentials are required; only DigitalOcean credentials are used.
- The harness connects to test droplets as `root`.
- `workspace.override.yml` and `tests/test_variables.yml` must stay untracked
  because they may contain local credentials.
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
