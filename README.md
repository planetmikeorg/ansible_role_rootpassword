# ansible_role_rootpassword

An Ansible role that rotates the root password on **RHEL-family** hosts and stores the new password in **1Password** using the [onepassword.connect](https://galaxy.ansible.com/onepassword/connect) Ansible Collection.

> **No 1Password CLI client required.** All interaction with 1Password is done through the [1Password Connect Server](https://developer.1password.com/docs/connect/) REST API via the official `onepassword.connect` collection.

---

## Requirements

| Requirement | Details |
|---|---|
| Ansible | >= 2.14 |
| Ansible Collection | `onepassword.connect >= 2.0.0` (see [requirements.yml](requirements.yml)) |
| 1Password Connect Server | A running [1Password Connect Server](https://developer.1password.com/docs/connect/get-started/) reachable from the Ansible control node |
| 1Password Connect Token | A service-account token with **read/write** access to the target vault |

Target hosts must be in the RedHat OS family (`ansible_os_family == "RedHat"`).

Install the required collection before running the role:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `onepassword_connect_host` | `$OP_CONNECT_HOST` env var | URL of the 1Password Connect Server (e.g. `https://op-connect.example.com`) |
| `onepassword_connect_token` | `$OP_CONNECT_TOKEN` env var | API token for the Connect Server |
| `onepassword_vault` | `$OP_VAULT` env var or `Private` | Name of the 1Password vault to store the password item in |
| `onepassword_item_title` | `Root Password - {{ inventory_hostname }}` | Title for the 1Password item |
| `rootpassword_length` | `32` | Length of the generated root password |

---

## What the Role Does

1. **Generates** a cryptographically random password of the configured length.
2. **Hashes** the password with SHA-512.
3. **Applies** the hashed password to the `root` system account using `ansible.builtin.user`.
4. **Verifies** that the root password hash changed on the target host.
5. **Upserts** a Login item in 1Password (via `onepassword.connect.item`) containing:
   - `username` – `root`
   - `password` – the plain-text password (stored as a concealed field)
   - `hostname` – `{{ inventory_hostname }}`

All tasks that handle the plain-text password use `no_log: true` to prevent credential exposure in Ansible output or logs.

---

## Example Playbook

```yaml
---
- name: Rotate root passwords
  hosts: all
  become: true

  vars:
    onepassword_connect_host: "https://op-connect.example.com"
    onepassword_connect_token: "{{ vault_op_connect_token }}"   # store in Ansible Vault
    onepassword_vault: "Infrastructure"

  roles:
    - role: ansible_role_rootpassword
```

### Using Environment Variables

The Connect host and token can be supplied via environment variables instead of role variables:

```bash
export OP_CONNECT_HOST="https://op-connect.example.com"
export OP_CONNECT_TOKEN="<your-service-account-token>"
ansible-playbook site.yml
```

---

## Testing

Molecule tests are included under `molecule/default/`. The verify playbook confirms that the root password hash has been updated on the target host.

```bash
molecule test
```

---

## License

MIT

---

## Author Information

[planetmikeorg](https://github.com/planetmikeorg)
