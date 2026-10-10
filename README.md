# Ansible Bootstrap

Wattle Labs ansible-pull bootstrap. Applies a minimal base config to a new host so the Ansible controller can manage it with normal playbooks.

Run once per host, by hand. It is not scheduled. After the bootstrap, the controller manages the host.

Supported targets: Debian and Ubuntu. The play asserts this and fails on anything else.

## What it does

- Installs base packages (`base_packages`) and keeps them at latest on every run.
- Sets the system timezone (`/etc/localtime`, `/etc/timezone`).
- Generates and sets the default locale (`/etc/locale.gen`, `/etc/default/locale`).
- Creates the `svc_ansible` service account with a locked password and deploys the controller's public key to it.
- Adds a sudoers drop-in for the service account in `/etc/sudoers.d/`, validated with `visudo -cf`.
- Enables UFW: deny inbound by default, allow SSH only from `ssh_allowed_cidrs`. See [Firewall behaviour](https://claude.ai/cowork/cse_01SXJnYfovPorFQPiGihk2qN#firewall-behaviour).
- Ensures the SSH service is enabled and running.

## Out of scope

Everything else is applied by the controller's playbooks after the bootstrap, including:

- Hostname, DNS resolvers and NTP (Physical and VMs only, Proxmox manages these for LXCs).
- SSH daemon hardening (password auth, root login).

## Prerequisites (target host)

1. Install packages:
    
    ```bash
    sudo apt install git ansible-core
    ```
    
2. Copy the repo deploy key `git-ansible-bootstrap` from 1Password to `/root/.ssh/git-ansible-bootstrap`, then set its mode:
    
    ```bash
    sudo chmod 0600 /root/.ssh/git-ansible-bootstrap
    ```
    
3. ansible-pull runs as root, so append this to `/root/.ssh/config`:
    
    ```ini
    Host git-ansible-bootstrap
      HostName github.com
      User git
      IdentityFile ~/.ssh/git-ansible-bootstrap
      IdentitiesOnly yes
    ```
    

## Configure

All variables live in `vars/main.yml`. Review them before the first run. In particular, confirm `ssh_allowed_cidrs` covers the network you are connecting from, or you will lock yourself out.

|Variable|Purpose|
|---|---|
|`ansible_svc_user`|Service account the controller connects as|
|`ansible_svc_home`|Home directory of the service account|
|`ansible_svc_shell`|Login shell of the service account|
|`controller_pubkey`|Controller's SSH public key (full line: type, key, comment)|
|`controller_from`|Optional source IP restriction for the key (`from=` option). Empty means none|
|`grant_sudo`|Whether the service account gets passwordless sudo|
|`sudoers_dropin_path`|Path of the sudoers drop-in|
|`base_packages`|Packages installed and kept at latest on every run|
|`ssh_service_name`|SSH service unit name (`ssh` on Debian/Ubuntu)|
|`ssh_port`|Port the UFW SSH rules apply to|
|`ssh_allowed_cidrs`|Networks allowed to reach SSH. Authoritative: other allow rules for `ssh_port` are removed|
|`system_timezone`|IANA timezone name, e.g. `Pacific/Auckland`|
|`system_locale`|Default locale, e.g. `en_NZ.UTF-8`|

## Run

ansible-pull runs `local.yml` by default.

```bash
sudo ansible-pull \
    -U git@git-ansible-bootstrap:colwilliamsnz/ansible-bootstrap.git \
    -C main \
    -i localhost, \
    --accept-host-key
```

The playbook is safe to re-run. A re-run reapplies the same config. The only change to expect is `base_packages` being upgraded to latest.

Add `--check` for a dry run. It skips the `ufw` and `locale-gen` commands, so it does not preview firewall or locale changes.

## After the run: trust the host key on the controller

Add the target's host key to the controller's `known_hosts` so later automated SSH connections can verify it.

Single host:

```bash
ssh-keyscan -H -t ed25519 target.example.com >> ~/.ssh/known_hosts
```

Every host in the inventory:

```bash
ansible all -i inventory.ini --list-hosts | tail -n +2 | xargs -I{} ssh-keyscan -H -t ed25519 {} >> ~/.ssh/known_hosts
```

`ssh-keyscan` trusts whatever key answers. Compare the fingerprint against the target's console output (`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`) before relying on it.

## Firewall behaviour

- Allow rules are added before UFW is enabled, so SSH from permitted networks is never interrupted.
- Any other UFW allow rule for `ssh_port` is deleted. This includes `allow 22/tcp`, `allow OpenSSH`, and networks removed from `ssh_allowed_cidrs`. Allow rules for other ports are left alone.
- **Lockout risk:** if you run the bootstrap over SSH from a network not in `ssh_allowed_cidrs`, your next connection will be refused. The controller's IP (and `controller_from`, if set) must fall inside a permitted network.
- Rules are IPv4 only when the CIDRs are IPv4. Inbound IPv6 SSH is then denied by the default policy.

## Security model

The controller public key is not a secret. Repo integrity is what matters. Anyone who can push to the pulled branch decides which key is authorised, and so gets root on every host that pulls.

- Protect the branch and restrict push access.
- Sign commits, and pull with `--verify-commit` once a signing key is in place on targets.
- Set `controller_from` to the controller's IP.
