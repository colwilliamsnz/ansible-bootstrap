# Ansible Bootstrap

Wattle Labs Ansible-Pull Bootstrap. Provisions minimal base config so that an Ansible controller node can
deploy playbooks.

Implemented:

- Install base packages and set the system timezone.
- Create the `svc_ansible` service account with a locked password, and deploy the controller's public key to it.
- Add a sudoers drop-in scoped to the service account in /etc/sudoers.d/, validated with `visudo -cf`.
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
| `ssh_service_name` | `ssh` on Debian/Ubuntu, `sshd` on RHEL |
| `system_timezone` | IANA timezone name |


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
