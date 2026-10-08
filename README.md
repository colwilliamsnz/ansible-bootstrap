# Ansible Bootstrap
Wattle Labs Ansible-pull Bootstrap for provisioning required config so that Ansible playbooks can target the system.

Required packages on target server:

git
ansible-core

private key: git-ansible-bootstrap


### Run the playbook manually:

```bash
sudo ansible-playbook -i "localhost," -c local local.yml
```


### Run using ansible-pull:

```
# todo
```
