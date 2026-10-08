# Ansible Bootstrap
Wattle Labs Ansible-Pull Bootstrap. Provisions minimal base config so that an Ansible controller node can
deploy playbooks.

- Create an ansible service account with a locked password, and deploy the controller's public key to it.
- Add a scoped sudoers drop-in in /etc/sudoers.d/, validated with visudo -cf.
- Make sure python3 and any modules your roles assume (e.g. python3-apt) are present.
- Set the hostname/FQDN, DNS resolvers and chrony NTP.



### Pre-reqs on target server:

Packages:

- git
- ansible-core

SSH private key:

- 'git-ansible-bootstrap' repo key copied from 1Password to ~/ssh/github-ansible-bootstrap

SSH config:

Append the following to ~/.ssh/config

```ini
Host github-ansible-bootstrap
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-ansible-bootstrap
  IdentitiesOnly yes
```


### Clone the repo:

```bash
git clone git@github-ansible-bootstrap:colwilliamsnz/ansible-bootstrap.git
```

### Run using ansible-pull:

```
ansible-pull -U https://git.example/bootstrap.git \
    -e "bootstrap_hostname=svr-vm-newhost bootstrap_domain=wattle.nz" \
    bootstrap.yml
```

### Run the playbook manually:

```bash
sudo ansible-playbook -i "localhost," -c local bootstrap.yml
```
