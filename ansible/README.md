# ansible — MAGI system provisioning

Ansible playbook for bootstrapping a fresh Debian install into `nerv`,
the host running the MAGI stack.

## Structure

```
ansible/
├── site.yml                        # Main playbook
├── hosts.example                   # Inventory template
├── ansible.cfg                     # Ansible settings
├── group_vars/all/
│   ├── vars.yml                    # Non-secret variables
│   └── vault.yml.example           # Secret variables template (encrypt before use)
└── roles/
    ├── system/      # Timezone, apt upgrade, essential packages
    ├── users/       # Create shinji and gendo
    ├── tailscale/   # AT-Field (Tailscale) installation and auth
    ├── firewall/    # UFW rules (SSH only via AT-Field)
    ├── ssh/         # SSH hardening, ListenAddress on tailscale0
    ├── docker/      # Docker Engine + Compose plugin, add gendo to docker group
    ├── storage/     # mergerfs pool (/mnt/disk1 + /mnt/disk2 → /mnt/data)
    └── security/    # fail2ban, unattended-upgrades
```

## Prerequisites

On your control machine:

```bash
pip install ansible
ansible-galaxy collection install community.general ansible.posix
```

## First-time setup

**1. Create the inventory file:**

```bash
cp hosts.example hosts
# Edit hosts — set ansible_host to nerv's LAN IP for the initial run
```

**2. Create and encrypt the vault:**

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# Edit vault.yml and set real passwords for gendo, shinji, and become
ansible-vault encrypt group_vars/all/vault.yml
```

**3. Review storage variables:**

Open `group_vars/all/vars.yml` and confirm `disk1_device` and `disk2_device`
match the actual block devices on nerv. Verify with:

```bash
# On the target host:
lsblk
```

If the drives need to be formatted (fresh disks), set `format_drives: true` in
`vars.yml`. **This is destructive — it erases all data on both devices.**
Reset it to `false` after the first run.

**4. Run the playbook:**

```bash
ansible-playbook site.yml --ask-vault-pass
```

The playbook will pause at the Tailscale step if `tailscale_auth_key` is not
set. When prompted, SSH into nerv on a separate terminal and run:

```bash
sudo tailscale up
```

Follow the authentication URL, then return to the playbook and press Enter.

## Important: access changes after provisioning

The `firewall` and `ssh` roles restrict SSH to the AT-Field (Tailscale)
interface (`tailscale0`). After the playbook completes:

- SSH from the LAN IP will be **refused**
- Update `hosts` to use the Tailscale IP (`tailscale ip -4` on nerv)
- All future connections must be through the AT-Field

## Re-running the playbook

The playbook is idempotent. It can be re-run safely to apply config changes.
To run only specific roles, use tags or `--start-at-task`.

To run a single role:

```bash
ansible-playbook site.yml --ask-vault-pass --tags docker
```

Each role name doubles as its tag.

## Unattended Tailscale auth

To skip the interactive pause, add a Tailscale auth key to the vault:

```yaml
# In group_vars/all/vault.yml
tailscale_auth_key: "tskey-auth-..."
```

Generate one at https://login.tailscale.com/admin/settings/keys.

## Enabling password-less SSH

The SSH config deploys with `PasswordAuthentication yes` by default.
To harden further, once SSH keys are on all your devices:

1. Add your public key to `~/.ssh/authorized_keys` for both users on nerv
2. In `roles/ssh/templates/sshd_config.j2`, uncomment `PasswordAuthentication no`
   and comment out `PasswordAuthentication yes`
3. Re-run the playbook
