# routeros_bootstrap

Bootstrap for MikroTik RouterOS 7.x devices. Renders a `.rsc` script per host, uploads it to the device via `ansible.netcommon.net_put`, and executes a factory reset via the RouterOS API with `run-after-reset` to apply the bootstrap configuration. The SSH public key is embedded directly in the bootstrap script.

The role runs **one host at a time**. Each host provides its own variables via `host_vars`.

All passwords (default admin and user passwords) are **always prompted at runtime** and never stored in variables or config files.

## What it does

- Configure a MGMT bridge with L2 port security (MAC whitelisting, ARP reply-only, static ARP, default-deny)
- Optionally configure a Local-OOB bridge with L2 port security (MAC whitelisting, dynamic ARP, default-deny)
- Create a single admin user (full group) with SSH key authentication
- Enable API SSL with a temporary self-signed TLS certificate
- Harden SSH (disable password auth, strong crypto, aes-gcm ciphers, ed25519 host key)
- Enable WinBox on management interfaces only
- Remove the default admin user

## Requirements

| Name | Version |
|------|---------|
| ansible | >= 2.10 |
| community.routeros | * |
| ansible.netcommon | * |

### Python libraries

| Name | Description |
|------|-------------|
| `librouteros` | Required by `community.routeros.api` for reset via RouterOS API |

## Host Variables (required)

These **must** be provided per host via `host_vars/`:

### `routeros_bootstrap_admin_interfaces`

```yaml
routeros_bootstrap_admin_interfaces:
  mgmt:
    bridge_name: "BR_MGMT"
    nic_name: "ether4"
    ip: "192.168.88.10"
    netmask: 32
    comment: "MGMT"
  # optional
  localoob:
    bridge_name: "BR_LOCALOOB"
    nic_name: "ether2"
    ip: "10.0.0.1"
    netmask: 30
    comment: "Local_OOB"
    allowed_mac_addrs:
      - "11:22:33:44:55:66"
      - "22:33:44:55:66:77"
```

### `routeros_bootstrap_bastion_host`

```yaml
routeros_bootstrap_bastion_host:
  ip: "192.168.88.32"
  mac_addr: "aa:bb:cc:dd:ee:ff"
  comment: "bastionHost"
```

### `routeros_bootstrap_admin_user`

```yaml
routeros_bootstrap_admin_user:
  name: "netadmin"
  ssh_pubkey: "ssh-ed25519 AAAAC3Nz... user@host"
```

The admin user is always created in the `full` group. Password is **prompted at runtime**.

Additional users should be managed by a separate role (e.g. `routeros_baseline` or `routeros_hardening`).

### Ansible connection variables

These should be defined in `host_vars/` so that subsequent roles can connect to the device:

```yaml
# Use localoob.ip, or mgmt.ip if localoob is not defined
ansible_host: "{{ routeros_bootstrap_admin_interfaces.localoob.ip }}"
ansible_user: "{{ routeros_bootstrap_admin_user.name }}"
```

If Local-OOB is defined, `ansible_host` should point to the Local-OOB IP; otherwise, use the MGMT IP. The bootstrap role itself overrides `ansible_host` internally: pre-reset tasks connect to the factory default IP (`192.168.88.1`), and post-reset tasks connect via the Local-OOB IP if defined, falling back to the MGMT IP.

## Role Defaults (`defaults/main.yml`)

| Variable | Description | Default |
|----------|-------------|---------|
| `routeros_bootstrap_admin_iface_list` | Interface list name for management interfaces | `"MGMT"` |
| `routeros_bootstrap_ca_crt` | Self-signed CA certificate settings | `{name: "selfCA", common_name: "Bootstrap CA", key_usage: "key-cert-sign,crl-sign"}` |
| `routeros_bootstrap_crt` | Self-signed server certificate settings | `{name: "selfCRT", key_usage: "tls-server", organization: "ACME Inc.", days_valid: 7}` |

## Internal Defaults (`vars/main.yml`)

These can be overridden via `set_fact` if needed but are not intended for regular user override:

| Variable | Default |
|----------|---------|
| `routeros_bootstrap_rendered_scripts_path` | `"./rendered_scripts"` |
| `routeros_bootstrap_script_suffix` | `"_bootstrap.rsc"` |
| `routeros_bootstrap_api_ssl_port` | `8729` |
| `routeros_bootstrap_crts_key_size` | `"prime256v1"` |
| `routeros_bootstrap_ssh_settings` | `{host_key_type: "ed25519", ciphers: "aes-gcm", port: 22}` |
| `routeros_bootstrap_default_admin` | `"admin"` |
| `routeros_bootstrap_min_password_length` | `16` |
| `routeros_bootstrap_min_password_categories` | `4` |
| `routeros_bootstrap_winbox_settings` | `{port: 8291, mac_ping: "no"}` |
| `routeros_bootstrap_default_ip` | `"192.168.88.1"` |
| `routeros_bootstrap_ssh_pubkey_temp_file` | `"{{ routeros_bootstrap_admin_user.name }}_ssh_pubkey"` |

## Usage

### Inventory (`inventory.ini`)

```ini
[router_hapac]
router_hapac

[router_hapac:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
ansible_ssh_private_key_file=~/.ssh/your_key_ed25519
```

### Host vars (`host_vars/router_hapac.yml`)

```yaml
routeros_bootstrap_admin_interfaces:
  mgmt:
    bridge_name: "BR_MGMT"
    nic_name: "ether4"
    ip: "192.168.88.10"
    netmask: 32
    comment: "MGMT"

routeros_bootstrap_bastion_host:
  ip: "192.168.88.32"
  mac_addr: "aa:bb:cc:dd:ee:ff"
  comment: "bastionHost"

routeros_bootstrap_admin_user:
  name: "netadmin"
  ssh_pubkey: "ssh-ed25519 AAAAC3Nz... user@host"

# Use localoob.ip, or mgmt.ip if localoob is not defined
ansible_host: "{{ routeros_bootstrap_admin_interfaces.localoob.ip }}"
ansible_user: "{{ routeros_bootstrap_admin_user.name }}"
```

### Playbook

```yaml
---
- name: Bootstrap RouterOS device
  hosts: router_hapac
  gather_facts: no
  become: no

  roles:
    - routeros_bootstrap
```

### Tags

| Tag | What runs |
|-----|-----------|
| `render` | Variables + template rendering only (no device interaction) |
| `deploy` | Full run: variables + rendering + upload + reset |

```bash
# Render only (test the template output)
ansible-playbook -i inventory.ini playbook.yml --tags render

# Full deploy
ansible-playbook -i inventory.ini playbook.yml --tags deploy
```

## Task Flow

1. **variables.yml** - Set internal defaults, prompt for passwords (default admin + new admin user), normalize MACs to uppercase, validate required host variables
2. **render_bootstrap.yml** - Render the `.rsc` template locally on the controller (`delegate_to: localhost`)
3. **reset_mikrotik.yml** - Upload rendered script to device, execute `/system/reset-configuration` via RouterOS API with `run-after-reset`, wait for device on Local-OOB IP (or MGMT IP if Local-OOB is not defined). The bootstrap script self-deletes from the device after execution
