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

The play succeeds for compliant devices. For a noncompliant device, the failed
assertion lists every required line that was not found.
