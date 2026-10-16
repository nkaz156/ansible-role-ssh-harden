# ssh_harden

Hardens the OpenSSH server on RHEL-family systems (built for Rocky Linux 10.2,
also works on EL9), following the
[Linuxize SSH hardening guide](https://linuxize.com/post/ssh-hardening-best-practices/):

- key-based authentication only (password and keyboard-interactive off)
- no root login
- access limited to groups with `AllowGroups`
- no empty passwords, 30 s login grace time, 3 authentication attempts
- X11, agent and TCP forwarding off
- only Ed25519 and RSA host keys offered
- idle sessions closed after 10 minutes
- a warning banner shown before login
- optionally, SSH allowed only from trusted addresses with firewalld

These parts of the guide are left out on purpose: the port stays at **22**,
access is limited with `AllowGroups` rather than `AllowUsers`, and Fail2Ban is not
installed. The guide's advice on client key types (Ed25519, RSA ≥ 4096) is
applied to the host keys the server offers.

## What the role changes

| File | How |
|------|-----|
| `/etc/ssh/sshd_config.d/01-ssh-harden.conf` | **Replaced** completely from a template |
| `/etc/ssh/banner` | **Replaced** when the banner is enabled |
| `/etc/ssh/sshd_config` | Only makes sure `Include /etc/ssh/sshd_config.d/*.conf` is present |
| `/etc/ssh/ssh_host_{ed25519,rsa}_key` | Generated only if missing |
| Groups in `ssh_harden_allow_groups` | Created; listed users are added to them |
| firewalld zone (opt-in) | Rich rules added, the `ssh` service removed |

sshd keeps the **first** value it reads for each setting. EL10 ships
`25-permitrootlogin.conf` (`PermitRootLogin yes`) and `50-redhat.conf`
(`X11Forwarding yes`), so the role's drop-in is named `01-…` so that it loads
before them. Distribution files are left alone. The settings in effect can be
checked with `sshd -T`.

## Requirements

- Ansible core 2.14 or later
- A RHEL-family 9 or 10 target, with `become: true`
- `ansible.posix`, only when `ssh_harden_firewall_manage: true`
  (`ansible-galaxy collection install -r requirements.yml`)
- Working SSH key login for an account that will be in an allowed group

## Role variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ssh_harden_password_authentication` | `false` | `PasswordAuthentication` |
| `ssh_harden_kbd_interactive_authentication` | `false` | `KbdInteractiveAuthentication` |
| `ssh_harden_pubkey_authentication` | `true` | `PubkeyAuthentication` |
| `ssh_harden_permit_root_login` | `"no"` | `PermitRootLogin`: `"no"` or `prohibit-password` |
| `ssh_harden_allow_groups` | `[sshusers]` | Groups allowed to log in (`AllowGroups`); must not be empty |
| `ssh_harden_create_allow_groups` | `true` | Create the allowed groups if they are missing |
| `ssh_harden_allow_group_members` | `{}` | Existing users to add, keyed by group, e.g. `{sshusers: [deploy]}`. Users are never created |
| `ssh_harden_lockout_check` | `true` | Fail before changing anything if no one could log in afterwards (see below) |
| `ssh_harden_permit_empty_passwords` | `false` | `PermitEmptyPasswords` |
| `ssh_harden_login_grace_time` | `30` | `LoginGraceTime`, in seconds |
| `ssh_harden_max_auth_tries` | `3` | `MaxAuthTries` |
| `ssh_harden_x11_forwarding` | `false` | `X11Forwarding` |
| `ssh_harden_allow_agent_forwarding` | `false` | `AllowAgentForwarding` |
| `ssh_harden_allow_tcp_forwarding` | `"no"` | `AllowTcpForwarding`: `no`, `yes`, `local`, `remote`, `all` |
| `ssh_harden_host_key_types` | `[ed25519, rsa]` | `HostKey` types offered; missing keys are generated (RSA at 4096 bits) |
| `ssh_harden_client_alive_interval` | `300` | `ClientAliveInterval`, in seconds |
| `ssh_harden_client_alive_count_max` | `2` | `ClientAliveCountMax` |
| `ssh_harden_banner_enabled` | `true` | Show the banner (`Banner none` when false) |
| `ssh_harden_banner_path` | `/etc/ssh/banner` | Banner file |
| `ssh_harden_banner_text` | `Authorized access only. …` | Banner text |
| `ssh_harden_extra_options` | `{}` | Extra directives for the drop-in, e.g. `{MaxSessions: 4}` |
| `ssh_harden_firewall_manage` | `false` | Allow SSH only from `ssh_harden_firewall_allowed_sources` |
| `ssh_harden_firewall_zone` | `public` | firewalld zone to manage |
| `ssh_harden_firewall_allowed_sources` | `[]` | Trusted IPv4/IPv6 addresses or CIDRs; must not be empty when the firewall is managed |

All variables are checked against `meta/argument_specs.yml` when the role starts.

### Lockout protection

With `AllowGroups`, any user outside the listed groups is refused, root included.
Before it changes anything, the role fails if:

- none of the allowed groups has a member (counting supplementary members,
  users whose primary group it is, and users in `ssh_harden_allow_group_members`), or
- Ansible connects over SSH as `ansible_user` and that user would be refused:
  not in an allowed group, or `root` while `PermitRootLogin no`.

The member check reads the local `passwd`/`group` databases. If accounts come from
SSSD/LDAP without enumeration, it may not see them; in that case set
`ssh_harden_lockout_check: false` once you have confirmed access.

Separately, and regardless of `ssh_harden_lockout_check`, the role always fails
before making changes if public key, password and keyboard-interactive
authentication are all disabled, since sshd would then accept no logins at all.
Key-only login (the default) is recommended.

## Example playbook

```yaml
- name: Harden SSH
  hosts: servers
  become: true
  roles:
    - role: nkaz156.ssh_harden
      vars:
        ssh_harden_allow_groups:
          - sshusers
          - wheel
        ssh_harden_allow_group_members:
          sshusers:
            - deploy
            - admin
        # Optional: allow SSH only from the office network and a bastion
        ssh_harden_firewall_manage: true
        ssh_harden_firewall_allowed_sources:
          - 203.0.113.0/24
          - 2001:db8::10
```

Keep an existing session or console access open the first time you apply the role,
and test a new login before closing it.

## Behaviour

- **Idempotent:** yes. A second run reports no changes.
- **Atomic:** no. If a run stops partway, earlier steps (groups, group
  membership, generated host keys, the banner) stay applied. sshd itself only
  changes after the complete configuration passes `sshd -t`, so it never runs a
  half-applied configuration.
- **Outputs:** none. The role sets no variables for later use; anything it
  sets (`__ssh_harden_*`, `getent_*` facts) is internal.
- **Check mode:** supported, including on a host the role has never touched.
  `--check --diff` shows the drop-in that would be written. If `openssh-server`
  (or `firewalld`, when managed) is not installed yet, check mode reports the
  install and skips the sshd (or firewalld) configuration, since there is
  nothing to compare against until the package exists.
- **Validation:** the drop-in and `sshd_config` are each checked with `sshd -t`
  before they are written. After a change, the full configuration is checked
  with `sshd -t` again, and sshd is **reloaded** only if that passes, so
  existing sessions stay open.
- **Rollback:** every replaced file is backed up next to the original
  (`<file>.<timestamp>~`). To undo everything, delete
  `/etc/ssh/sshd_config.d/01-ssh-harden.conf` and run `systemctl reload sshd`.
  Firewall changes are reverted with `firewall-cmd --permanent --zone=public
  --add-service=ssh --remove-rich-rule=…`, then `firewall-cmd --reload`.

## Testing

Lint checks run as pre-commit hooks (`.pre-commit-config.yaml`): yamllint,
ansible-lint on the production profile, gitleaks and basic file hygiene. The
ansible-lint hook needs Python 3.14 on the machine (it pins
`language_version: python3.14`). Install them with
`pip install pre-commit && pre-commit install`.

`.github/workflows/ci.yml` runs the same hooks, plus a gitleaks scan of the
full history. It then runs `molecule test` for each scenario on Rocky Linux
10.2 and 9, in systemd containers built from `molecule/resources/Containerfile`.

| Scenario | What it applies |
|----------|-----------------|
| `default` | The role defaults, with one user added to `sshusers` |
| `custom` | Every variable changed from its default: several groups, key-only root login, forwarding on, ECDSA host key, banner off, extra options, firewalld on |

All scenarios share their playbooks (`molecule/resources/`), and each scenario's
settings live in `molecule/<scenario>/inventory/group_vars/molecule.yml`.
`verify.yml` works out what to expect from those variables, so a new scenario
only needs a new `group_vars` file. It checks:

- every setting the role manages appears in `sshd -T` with the expected value.
  `AllowGroups` and `HostKey` must list exactly the configured groups and keys,
  and `AllowUsers` must be unset
- the role's drop-in loads before the distribution drop-ins, and
  `sshd_config` includes the drop-in directory before any other setting
- file ownership and permissions, the banner text, and that the host keys exist
- the groups exist and the listed users are in them
- sshd is enabled, running and listening on port 22
- real SSH logins inside the container: allowed users get in, users outside
  the allowed groups (and root, unless permitted) are refused, password and
  keyboard-interactive are not offered, and the banner shows only when enabled
- the firewalld rich rules are in place (runtime and permanent) and the open
  `ssh` service has been removed
- applying the role with an allowed group that has no members fails, and changes nothing

To run locally:

```sh
pip install -r requirements-dev.txt
molecule test -s default
SSH_HARDEN_TEST_IMAGE=docker.io/rockylinux/rockylinux:9 molecule test -s custom
```

Rocky Linux 10 images need an x86-64-v3 CPU, so on older hardware test locally with
the Rocky 9 image and leave Rocky 10 to CI.

## License

MIT

## Author

nkaz156
