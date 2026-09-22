# aapLearning

## Cisco IOS configuration compliance

[`config-compliance.yml`](config-compliance.yml) retrieves each device's running
configuration and checks it against [`compliance.txt`](compliance.txt). Add one
required IOS command per line to the compliance file. Blank lines and lines
beginning with `#` are ignored. Leading and trailing whitespace is ignored, but
each remaining line must otherwise exactly match a line in the running config.

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
