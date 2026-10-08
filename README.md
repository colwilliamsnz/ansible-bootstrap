# Ansible Bootstrap
Wattle Labs Ansible-Pull Bootstrap. Provisions base required config so that Ansible playbooks can target the system from a controller node.


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


### Run the playbook manually:

```bash
sudo ansible-playbook -i "localhost," -c local local.yml
```


### Run using ansible-pull:

```
# todo
```
