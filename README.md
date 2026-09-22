# aapLearning

## Cisco IOS configuration compliance

[`config-compliance.yml`](config-compliance.yml) retrieves each device's running
configuration and checks it against [`compliance.txt`](compliance.txt) in a
single playbook run. Add one required IOS command per line to the compliance
file. Blank lines and lines beginning with `#` are ignored. Leading and
trailing whitespace is ignored, but each remaining line must otherwise exactly
match a line in the running config.

Example inventory:

```yaml
---
all:
	hosts:
		access-switch-01:
			ansible_host: 192.0.2.10
			netbox_tags:
				- ios-xe
			ansible_user: automation
			ansible_password: "{{ vault_network_password }}"
			ansible_become: true
			ansible_become_method: enable
			ansible_become_password: "{{ vault_enable_password }}"
```

Run the check with:

```shell
ansible-playbook -i inventory.yml config-compliance.yml --ask-vault-pass
```

Only hosts in the AAP inventory groups `tags_ios` or `tags_ios-xe` are checked.
The play succeeds for compliant devices. For a noncompliant device, the failed
assertion lists every required line that was not found.

### Split collect/check workflow (recommended in AAP)

[`collect-config.yml`](collect-config.yml) and
[`compliance-check.yml`](compliance-check.yml) split the same logic into two
playbooks so an AAP workflow can tell you *where* a host failed: unreachable or
a command error during collection, versus a real compliance gap found in the
config. The two playbooks pass the running config between them through
Ansible's fact cache instead of connecting to the device twice:

1. `collect-config.yml` connects over `network_cli` and caches the parsed
   running-config lines with `set_fact: cacheable: true`.
2. `compliance-check.yml` runs with `connection: local` and never touches the
   device — it reads `running_config_lines` back out of the fact cache, diffs
   it against `compliance.txt`, and asserts/reports exactly like the combined
   playbook.

To wire this up as an AAP workflow:

1. Create a **Collect Config** job template pointing at `collect-config.yml`.
2. Create a **Compliance Check** job template pointing at `compliance-check.yml`.
3. On **both** job templates, check **Enable Fact Storage** so the cache is
   written by the first and read by the second.
4. Set a short **Cache Timeout** (Job Settings) so a stale cached config from a
   previous run can't silently satisfy a host that failed to collect this
   time — the compliance check only trusts facts newer than this window.
5. Create a workflow template with two nodes: **Collect Config**, then a
   **Compliance Check** node linked on the **On Success** path.

With this wired up, a host that fails to connect or errors out shows up as a
failure on the Collect Config node, and a host that connects fine but is
missing required config shows up as a failure on the Compliance Check node —
distinguishable at a glance in the workflow view, without SSH-ing into any
device twice.

## Sync interface descriptions and VLANs to NetBox

[`netbox-sync.yml`](netbox-sync.yml) gathers Cisco IOS interface, layer 2
interface, and VLAN resource facts, then writes them into NetBox. It creates the
device if needed, creates VLANs, updates interface descriptions, and applies
access/trunk VLAN assignments when those facts are available from IOS.

Install the required Ansible collections:

```shell
ansible-galaxy collection install -r requirements.yml
```

Set the NetBox connection details before running the playbook:

```shell
export NETBOX_URL=https://netbox.example.com
export NETBOX_TOKEN=your-token
export NETBOX_SITE=Default
```

Run the sync with:

```shell
ansible-playbook -i inventory.yml netbox-sync.yml --ask-vault-pass
```

The NetBox device name defaults to the Ansible `inventory_hostname`. Adjust the
inventory hostnames or NetBox device names so they match the records you want to
update.
