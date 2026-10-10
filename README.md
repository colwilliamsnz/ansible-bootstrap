# Ansible Bootstrap

Wattle Labs ansible-pull bootstrap. Applies a minimal base config to a new host so the Ansible controller can manage it with normal playbooks.

Run once per host, by hand. It is not scheduled. After the bootstrap, the controller manages the host.

Supported targets: Debian and Ubuntu. The play asserts this and fails on anything else.

## What it does

- Installs base packages (`base_packages`). Packages are installed if missing, not upgraded (patching is the controller's job). Note that a number of useful utilities are installed over and above what Ansible needs to run.
- Sets the system timezone (`/etc/localtime`, `/etc/timezone`).
- Generates and sets the default locale (`/etc/locale.gen`, `/etc/default/locale`).
- Creates the `svc_ansible` service account with a locked password and writes the controller's public key to its `authorized_keys`. The file is authoritative: any other key is removed.
- Grants the service account passwordless sudo via a drop-in in `/etc/sudoers.d/`. See [Sudo access](#sudo-access).
- Enables UFW: deny inbound by default, allow SSH only from `ssh_allowed_cidrs`. See [Firewall behaviour](#firewall-behaviour).
- Ensures the SSH service is enabled and running.

## Out of scope

Everything else is applied by the controller's playbooks after the bootstrap, including:

- Hostname, DNS resolvers and NTP (Physical and VMs only, Proxmox manages these for LXCs).
- SSH daemon hardening (password auth, root login).
- Package upgrades.

## Repo visibility: private or public

The repo contains no secrets. The controller public key is public by nature, and everything else is configuration. The one thing it does reveal is internal network layout (`ssh_allowed_cidrs`, `controller_from`). Either visibility works; the choice changes the prerequisites and the run command.

|                | Private repo                                             | Public repo                               |
|----------------|----------------------------------------------------------|-------------------------------------------|
| Transport      | SSH with a read-only deploy key                          | HTTPS, no credentials                     |
| Target setup   | Deploy key from 1Password plus `/root/.ssh/config` entry | None beyond `git` and `ansible-core`      |
| Exposes        | Nothing                                                  | Network layout in `vars/main.yml`         |
| Integrity      | Branch protection, signed commits                        | Branch protection, signed commits (same)  |

Repo integrity matters equally in both cases. See [Security model](#security-model).

## Prerequisites (target host)

### Both variants

Install packages:

```bash
sudo apt update && sudo apt install git ansible-core
```

### Private repo only

1. Copy the repo deploy key `git-ansible-bootstrap` from 1Password to `/root/.ssh/git-ansible-bootstrap`, then set its mode:

    ```bash
    sudo chmod 0600 /root/.ssh/git-ansible-bootstrap
    ```

2. ansible-pull runs as root, so append this to `/root/.ssh/config`:

    ```ini
    Host git-ansible-bootstrap
      HostName github.com
      User git
      IdentityFile ~/.ssh/git-ansible-bootstrap
      IdentitiesOnly yes
    ```

### Public repo only

Nothing further. The host needs outbound HTTPS to `github.com`, which the bootstrap does not restrict (UFW default outbound policy is allow).

## Configure

All variables live in `vars/main.yml`. Review them before the first run. In particular, confirm `ssh_allowed_cidrs` covers the network you are connecting from, or you will lock yourself out.

|Variable|Purpose|
|---|---|
|`ansible_svc_user`|Service account the controller connects as|
|`ansible_svc_home`|Home directory of the service account|
|`ansible_svc_shell`|Login shell of the service account|
|`controller_pubkey`|Controller's SSH public key (full line: type, key, comment)|
|`controller_from`|Optional source IP restriction for the key (`from=` option). Empty means none|
|`controller_key_options`|Options prepended to the key in `authorized_keys`. Default `restrict`. Empty means none|
|`grant_sudo`|Whether the service account gets passwordless sudo|
|`sudoers_dropin_path`|Path of the sudoers drop-in|
|`base_packages`|Packages installed if missing|
|`ssh_service_name`|SSH service unit name (`ssh` on Debian/Ubuntu)|
|`ssh_port`|Port the UFW SSH rules apply to|
|`ssh_allowed_cidrs`|Networks allowed to reach SSH. Authoritative: other allow/limit rules for `ssh_port` are removed|
|`system_timezone`|IANA timezone name, e.g. `Pacific/Auckland`|
|`system_locale`|Default locale, e.g. `en_NZ.UTF-8`|

`restrict` disables port forwarding, agent forwarding, X11 forwarding and pty allocation for the key. Ansible needs none of these (it runs modules over the SSH exec channel and `become` uses passwordless sudo). Clear `controller_key_options` if you need an interactive shell as the service account.

## Run

ansible-pull runs `local.yml` by default. Pick the command that matches the repo visibility.

### Private repo (SSH, deploy key)

```bash
sudo ansible-pull \
    -U git@git-ansible-bootstrap:colwilliamsnz/ansible-bootstrap.git \
    -C main \
    -i localhost, \
    --accept-host-key \
    --purge
```

`--accept-host-key` adds GitHub's SSH host key to root's `known_hosts` on first contact. Without it the clone stops at the host key prompt.

### Public repo (HTTPS)

```bash
sudo ansible-pull \
    -U https://github.com/colwilliamsnz/ansible-bootstrap.git \
    -C main \
    -i localhost, \
    --purge
```

No `--accept-host-key`: HTTPS verifies GitHub through the system CA bundle (`ca-certificates`), not `known_hosts`.

### Common options

`--purge` deletes the checkout (`/root/.ansible/pull/<hostname>`) after the run. The bootstrap is one-shot, so nothing needs it afterwards.

The playbook is safe to re-run. A re-run reapplies the same config and reports no changes if nothing has drifted.

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

## Sudo access

The controller connects as `svc_ansible`, an unprivileged account. Almost everything a playbook does (install packages, write to `/etc`, manage services) needs root, which Ansible gets with `become: true`. On Debian/Ubuntu `become` runs `sudo` under the hood, and there is no one at the keyboard to type a password. The sudoers drop-in is what makes that work.

The play writes one line to `/etc/sudoers.d/99-ansible`:

```
svc_ansible ALL=(ALL) NOPASSWD: ALL
```

Read left to right: on any host, `svc_ansible` may run any command as any user without a password.

Why a drop-in rather than editing `/etc/sudoers`:

- `/etc/sudoers` is owned by the distro package. Editing it invites merge conflicts on upgrade and makes the change hard to find later.
- A drop-in is a single file with one owner and one purpose. Removing it (or setting `grant_sudo: false`) removes the grant cleanly.
- The `99-` prefix sorts it after the distro's own drop-ins, so it is applied last.

Why it is safe to write with Ansible:

- The file is passed through `visudo -cf` before it is put in place. A syntax error in sudoers can lock everyone out of `sudo` on the host. `visudo -cf` rejects the file and the task fails instead.
- Mode is `0440`, owner `root:root`. `sudo` ignores drop-ins with looser permissions.

Why `NOPASSWD: ALL` and not a narrower command list:

- Ansible modules run as arbitrary Python scripts in temp paths, so a command allow-list cannot be expressed meaningfully. Any attempt to restrict it would either break Ansible or be trivially bypassed.
- The real boundary is who can authenticate as `svc_ansible`. That is controlled by the `authorized_keys` file (single key, `restrict`, `from=`) and UFW. See [Security model](#security-model).

Set `grant_sudo: false` if the controller will only ever run tasks that do not need root on this host. The play then removes the drop-in if it exists.

## Firewall behaviour

- Allow rules are added before UFW is enabled, so SSH from permitted networks is never interrupted.
- The managed rule form is `allow from <cidr> to any port <ssh_port> proto tcp`. Networks removed from `ssh_allowed_cidrs` have their rule deleted.
- Any other `allow` or `limit` rule that references `ssh_port` is deleted. This covers `allow 22/tcp`, `limit 22/tcp`, `allow OpenSSH`, `allow ssh`, `allow from any to any port 22`, and interface-scoped forms such as `allow in on eth0 to any port 22`. Rules for other ports are left alone.
- Default policies (`DEFAULT_INPUT_POLICY="DROP"`, `DEFAULT_OUTPUT_POLICY="ACCEPT"`) are the UFW defaults. The play enforces them in `/etc/default/ufw` in case they have been changed.
- **Lockout risk:** if you run the bootstrap over SSH from a network not in `ssh_allowed_cidrs`, your next connection will be refused. The controller's IP (and `controller_from`, if set) must fall inside a permitted network.
- Rules are IPv4 only when the CIDRs are IPv4. Inbound IPv6 SSH is then denied by the default policy.

## Security model

The controller public key is not a secret. Repo integrity is what matters. Anyone who can push to the pulled branch decides which key is authorised, and so gets root on every host that pulls. This is true whether the repo is private or public: visibility controls who can read, not who can write.

- Protect the branch and restrict push access.
- Sign commits, and pull with `--verify-commit` once a signing key is in place on targets.
- Set `controller_from` to the controller's IP.
- Keep `controller_key_options` at `restrict` unless there is a reason not to.

A private repo additionally hides the network layout in `vars/main.yml` from the public. A public repo removes the deploy key handling from every new host. Pick based on how much the CIDRs and controller IP matter to you; the play itself is the same either way.
