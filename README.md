# Ansible Bootstrap

Wattle Labs Ansible-Pull Bootstrap. Provisions minimal base config so that an Ansible controller node can
deploy playbooks.

Supported targets: Debian and Ubuntu (the play asserts this).

Implemented:

- Install base packages.
- Set the system timezone (`/etc/localtime` and `/etc/timezone`).
- Generate and set the system default locale (`/etc/locale.gen`, `/etc/default/locale`).
- Create the `svc_ansible` service account with a locked password, and deploy the controller's public key to it.
- Add a sudoers drop-in scoped to the service account in /etc/sudoers.d/, validated with `visudo -cf`.
- Enable UFW: deny inbound by default, allow SSH only from `ssh_allowed_cidrs`.
- Ensure the SSH service is enabled and running.

Planned:

- Make sure python3 and any modules your roles assume (e.g. python3-apt) are present.
- Set the hostname/FQDN, DNS resolvers and chrony NTP.


### Variables

All variables live in `vars/main.yml`.

| Variable | Purpose |
|---|---|
| `ansible_svc_user` | Service account the controller connects as |
| `ansible_svc_home` | Home directory of the service account |
| `ansible_svc_shell` | Login shell of the service account |
| `controller_pubkey` | Controller's SSH public key (full line: type, key, comment) |
| `controller_from` | Optional source IP restriction for the key (`from=` option). Empty = none |
| `grant_sudo` | Whether the service account gets passwordless sudo |
| `sudoers_dropin_path` | Path of the sudoers drop-in |
| `base_packages` | Packages installed and kept at latest on every run |
| `ssh_service_name` | SSH service unit name (`ssh` on Debian/Ubuntu) |
| `system_timezone` | IANA timezone name, e.g. `Pacific/Auckland` |
| `system_locale` | Default locale, e.g. `en_NZ.UTF-8` |
| `ssh_port` | Port the UFW SSH rules apply to |
| `ssh_allowed_cidrs` | Networks allowed to reach SSH. Authoritative: other allow rules for `ssh_port` are removed |


### Firewall behaviour

- Allow rules are added before UFW is enabled, so SSH from the permitted networks is never interrupted.
- Any other UFW allow rule for `ssh_port` (e.g. `allow 22/tcp`, `allow OpenSSH`, or a network
  removed from `ssh_allowed_cidrs`) is deleted. Allow rules for other ports are left alone.
- Lockout risk: if you run the bootstrap over SSH from a network not in `ssh_allowed_cidrs`,
  your next connection will be refused. Make sure the controller's IP (and `controller_from`)
  falls inside one of the permitted networks.
- Rules are IPv4 only if the CIDRs are IPv4. Inbound IPv6 SSH is denied by the default policy.
- `--check` skips the `ufw` and `locale-gen` commands, so check mode does not preview those changes.


### Security model

The controller public key is not a secret. The repo's integrity is what matters: anyone who can push
to the pulled branch decides which key is authorised, and so gets root on every host that pulls.

- Protect the branch and restrict push access.
- Sign commits and pull with `--verify-commit` once a signing key is in place on targets.
- Set `controller_from` to the controller's IP.


### Pre-reqs on target server:

Packages:

- git
- ansible-core

SSH private key (repo deploy key):

- 'git-ansible-bootstrap' repo key copied from 1Password to `/root/.ssh/github-ansible-bootstrap` (mode 0600)

SSH config:

ansible-pull runs as root, so append the following to `/root/.ssh/config`

```ini
Host github-ansible-bootstrap
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-ansible-bootstrap
  IdentitiesOnly yes
```


### Run using ansible-pull:

ansible-pull runs `local.yml` by default.

```bash
sudo ansible-pull \
    -U git@github-ansible-bootstrap:colwilliamsnz/ansible-bootstrap.git \
    -C main \
    -i localhost, \
    --accept-host-key
```


### Run the playbook manually (from a clone):

```bash
sudo ansible-playbook -i "localhost," -c local local.yml
```

### Trust the target server host key:

Add the target server’s public host key on the Ansible control node, so the control node can verify later automated SSH connections

From the control node:

```
ssh-keyscan -H -t ed25519 target.example.com >> ~/.ssh/known_hosts
```

To trust every host in your inventory:

```
ansible all -i inventory.ini --list-hosts | tail -n +2 | xargs -I{} ssh-keyscan -H -t ed25519 {} >> ~/.ssh/known_hosts
```
