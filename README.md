# Project set up
* create an inventory file (e.g. hosts or hosts.yaml) that holds the remote hosts that ansible will handle.
* Example entry is
```yaml
webserver: # <-- group
  hosts: # <-- List of hosts in group
    gcloud_host: # <-- host number 1 in group
      ansible_host: 35.189.109.16
      ansible_port: 22
      ansible_ssh_user: rg
    app01:  # <-- host number 2 in group
      ansible_host: app01
    app02:  # <-- host number 3 in group
      ansible_host: app02
  vars:  # <-- common variables in this group
    ansible_python_interpreter: /usr/bin/python3
```
* to test if all hosts are accesible, run
```bash
ansible -m ping all
```
* to test if a group of hosts are accesible, run
```bash
ansible -m ping <group-name>
```

# Where to put variables
The same value can be written in many places. We follow a few simple rules, so that every value is defined **once**, in the place where it is easiest to find and change.

### The main idea
1. **Group hosts by role**, not by provider or location: `dbservers`, `appservers`, `frontservers`.
   A playbook targets a role (`hosts: appservers`) and `-l` only limits the run to some hosts of it.
   A host can have more than one role (e.g. `devops-1` is in `appservers` and `dbservers`), and then it gets the variables of both groups.
2. **Put a value as "high" as possible**: in the most general place that is correct for every host that uses it.
3. **Override only what differs**, as low as needed (in `host_vars`), and nothing more.
4. **Don't write what Ansible can find out**: values that depend on the OS, the architecture, the user etc. are computed from facts.
5. **Every value that appears twice, or that someone may want to change** (versions, paths, URLs, ports, repositories), becomes a variable.

### Which file? Ask these questions in order
| Question | If yes, put it in | Example in this project |
|---|---|---|
| Does it change on every run (e.g. a build number)? | `-e name=value` on the command line | `-e image_tag=abc123` (k8s playbook) |
| Is it a secret? | a file encrypted with `ansible-vault` (see [Vault](#vault)) | `db.password` in a real project |
| Can it be computed from facts? | a variable with a Jinja expression, or directly `ansible_facts` | `postgres_version`, docker `distribution_release` / `architecture` |
| Is it used only by one playbook and is the same on every host? | `vars:` of the play | `java_package`, `maven_version` in `spring.yaml` |
| Is it needed by **more than one group**? | `group_vars/all.yaml` | `db.*` (the db server creates the database, the app server connects to it), `app_port` (spring listens on it, the frontend nginx proxies to it) |
| Is it the same for **all hosts of one group**? | `group_vars/<group>.yaml` | `app_dir`, `app_jar` (appservers), `node_version` (frontservers), `db_allowed_cidr` (dbservers) |
| Is it different for **one host**? | `host_vars/<host>.yaml`, only that value | `db_host: localhost` for `devops-1` |

Rule of thumb: if you are about to copy the same line into a second `host_vars` file, it probably belongs in `group_vars`.
If you are about to copy it into a second `group_vars` file, it belongs in `group_vars/all.yaml`.

### Precedence: who wins
When the same variable is defined in more than one place, the more **specific** one wins (simplified, from low to high):
```
group_vars/all  <  group_vars/<group>  <  host_vars/<host>  <  play vars:  <  set_fact / register  <  -e
```
This is why defaults go in `group_vars` and exceptions in `host_vars`: e.g. `db_host` is `dbserver` for all appservers, but `localhost` for `devops-1`.
Check the final value with `ansible-inventory --host <host>` (see below).

### Dictionaries: build them from simple variables
A dictionary defined again in `host_vars` **replaces** the whole dictionary from `group_vars` (it is not merged).
So we do not override dictionaries; we build them from simple variables and override those:
```yaml
# group_vars/appservers.yaml
db_host: dbserver
app_env:
  server.port: "{{ app_port }}"
  spring.datasource.url: "jdbc:postgresql://{{ db_host }}:{{ db_port }}/{{ db.name }}"
```
```yaml
# host_vars/devops-1.yaml: one line, the rest of app_env stays as is
db_host: localhost
```

### Values computed from facts
Variables are evaluated when they are **used**, not when the file is read. So a variable can depend on facts of the host:
```yaml
# group_vars/dbservers.yaml
postgres_versions:
  "22.04": 14
  "24.04": 16
postgres_version: "{{ postgres_versions[ansible_facts['distribution_version']] }}"
postgres_conf_dir: "/etc/postgresql/{{ postgres_version }}/main"
```
The same playbook then works on Ubuntu 22.04 and 24.04, without `when:` and without hardcoded paths.
Such variables need facts (`gather_facts`, enabled by default). That is also why `ansible-inventory` shows them unrendered.

### Naming
* prefix the variable with the component it belongs to: `frontend_branch`, `frontend_dir`, `db_host`, `postgres_version`
* avoid generic names in shared files (`branch`, `dir`, `version`): a play `vars: branch: main` would silently hide the `branch: api` of `group_vars`. That is why the two branches are called `spring_branch` and `frontend_branch`.
* use the same name for the same thing everywhere (`app_port` in the spring config, in the nginx template and in `wait_for`)

### Where each variable is defined in this project
| File | Applies to | Contains |
|---|---|---|
| `group_vars/all.yaml` | every host | `db.*`, `db_port`, `app_port` (needed by more than one group) |
| `group_vars/dbservers.yaml` | `dbservers` | `postgres_version` (from the Ubuntu version), `postgres_conf_dir`, `db_allowed_cidr` |
| `group_vars/appservers.yaml` | `appservers` | spring backend: `spring_repo_url`, `spring_branch`, `app_dir`, `app_jar`, `db_host`, `app_env`, `spring_service` |
| `group_vars/frontservers.yaml` | `frontservers` | vue frontend: `frontend_*`, `node_version`, `nvm_version`, `backend_host`, `backend_server_url` |
| `host_vars/<host>.yaml` | one host | only what differs, e.g. `db_host` |
| `vars:` in a playbook | that play | values used only by that playbook, e.g. `java_package` in `spring.yaml` |
| `-e name=value` | that run | highest precedence, e.g. `-e image_tag=abc123` |


# Inspect inventory and variables
Before running a playbook it is useful to check **which hosts** it will run on and **which values** the variables will have.
Variables come from many places (`hosts.yaml`, `group_vars/`, `host_vars/`, play `vars:`, `-e`), so it is easy to end up with a value you did not expect.

### `ansible-inventory --host`: all variables of one host
```bash
ansible-inventory --host devops-1
```
Prints, as JSON, the **final** variables of the host after merging `hosts.yaml`, `group_vars/all.yaml`, `group_vars/<group>.yaml` for every group the host belongs to, and `host_vars/<host>.yaml`:
```json
{
    "ansible_host": "devops-vm-1",
    "app_env": {
        "server.port": "{{ app_port }}",
        "spring.datasource.url": "jdbc:postgresql://{{ db_host }}:{{ db_port }}/{{ db.name }}",
        ...
    },
    "app_port": 8080,
    "app_dir": "{{ ansible_user_dir }}/spring",
    "spring_branch": "api",
    "db": { "name": "appdb", "user": "appuser", ... },   <-- group_vars/all.yaml
    "db_host": "localhost",                              <-- host_vars/devops-1.yaml
    "postgres_version": "{{ postgres_versions[ansible_facts['distribution_version']] }}",
    ...
}
```
What to look for:
* a variable that is **missing**: the playbook will fail with `'...' is undefined`
* a variable with an **unexpected value**, e.g. which file "won". Here `db_host` is `dbserver` in `group_vars/appservers.yaml`, but `localhost` for `devops-1` because `host_vars` has higher precedence.
* a **dictionary** in `host_vars` **replaces** the whole dictionary with the same name from `group_vars`, it is not merged. That is why `app_env` is built from simple variables (`db_host`, `app_port`, `db.*`) and the `host_vars` files only change those.
* Jinja expressions (`{{ ... }}`) are shown **unrendered**. They are evaluated only when a play runs, and the ones that use facts (e.g. `ansible_user_dir`, `ansible_facts`) need a connection to the host.

### Other useful commands
* tree of groups and hosts (which host belongs to which group)
```bash
ansible-inventory --graph
```
* the same tree including the variables of each host
```bash
ansible-inventory --graph --vars appservers
```
* the whole inventory (all groups, hosts and variables) as JSON or YAML
```bash
ansible-inventory --list
ansible-inventory --list --yaml
```
* the **rendered** value of a variable (Jinja is evaluated, e.g. `{{ app_port }}` becomes `8080`)
```bash
ansible devops-1 -m debug -a "var=app_env"
ansible appservers -m debug -a "var=app_env"   # for every host of the group
```
Ad-hoc commands do **not** gather facts, so variables that depend on facts show an error, e.g. `spring_service` uses `app_dir`, which uses `ansible_user_dir`:
`"workingdir": "<< error 1 - 'ansible_user_dir' is undefined >>/spring"`.
For those, use a playbook (facts are gathered by default), e.g. `ansible-playbook -l appserver-vm playbooks/hostvars_and_facts.yml`.
* which hosts a playbook will run on (respecting `hosts:` and `-l`), without running it
```bash
ansible-playbook playbooks/spring.yaml --list-hosts
ansible-playbook playbooks/spring.yaml --list-hosts -l devops-1
```
* which tasks (and tags) a playbook contains
```bash
ansible-playbook playbooks/spring.yaml --list-tasks
ansible-playbook playbooks/spring.yaml --list-tags
```
* check the playbook syntax (YAML, module names, indentation)
```bash
ansible-playbook playbooks/spring.yaml --syntax-check
```
* dry run: show what **would** change without changing anything (`--diff` also shows the changes in files and templates)
```bash
ansible-playbook -l devops-1 playbooks/spring.yaml --check --diff
```
Tasks that use `command`/`shell` are skipped in check mode, so tasks that depend on their results may fail.
* the facts the host reports (OS, IPs, users, ...), filtered by name
```bash
ansible devops-1 -m setup -a "filter=ansible_distribution*"
ansible devops-1 -m setup -a "filter=ansible_default_ipv4"
```
* the settings of `ansible.cfg` that differ from the defaults
```bash
ansible-config dump --only-changed
```
* more output while a playbook runs, e.g. the variables of each task or the ssh connection (`-v` up to `-vvvv`)
```bash
ansible-playbook -l devops-1 playbooks/spring.yaml -vv
```

# Run development environment with Vagrant
* run testing environment
```bash
vagrant plugin install vagrant-hostmanager
vagrant plugin install vagrant-libvirt   # only for the libvirt provider (Linux)
cd vagrant
vagrant up control   # first: creates the ssh key the other VMs need
vagrant up
vagrant ssh-config >> ~/.ssh/config
```

### libvirt provider
The `Vagrantfile` uses **libvirt** (KVM) as the default provider instead of VirtualBox.
* host requirements: `libvirt`, `qemu`, `virtiofsd` (for the shared `/vagrant` folder) and the `vagrant-libvirt` plugin
* the VMs live in `qemu:///system`. To see them graphically use `virt-manager` (connect to *QEMU/KVM*, not *User session*), or from the CLI:
```bash
virsh -c qemu:///system list --all
```

### VirtualBox provider (alternative)
If you prefer VirtualBox (e.g. on Windows / macOS, or if you already use it), the same `Vagrantfile` works:
* install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (no extra Vagrant plugin is needed)
* pass the provider when creating the VMs:
```bash
cd vagrant
vagrant up control --provider=virtualbox
vagrant up --provider=virtualbox
```
or set it once for your shell, so that every `vagrant` command uses it:
```bash
export VAGRANT_DEFAULT_PROVIDER=virtualbox    # Windows PowerShell: $env:VAGRANT_DEFAULT_PROVIDER="virtualbox"
```
* the provider is chosen when a VM is created. To switch provider, first destroy the existing VMs with `vagrant destroy -f`
* VirtualBox and libvirt/KVM cannot run VMs at the same time on the same host

### "Network is unreachable" inside the VMs (Docker + libvirt)
If Docker runs on the host, it sets the iptables `FORWARD` policy to `DROP`, which also blocks traffic from the libvirt bridges (`virbr*`). The VMs then have no internet access and may not be able to reach each other.
* check:
```bash
sudo iptables -S FORWARD | head -3    # "-P FORWARD DROP" means this is the problem
```
* permanent fix (Docker >= 28): tell Docker not to change the policy
```bash
sudo mkdir -p /etc/docker
echo '{ "ip-forward-no-drop": true }' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
sudo iptables -P FORWARD ACCEPT       # resets the DROP policy that is already active
```
* quick fix until the next reboot (alternative):
```bash
sudo iptables -I DOCKER-USER -i virbr+ -j ACCEPT
sudo iptables -I DOCKER-USER -o virbr+ -j ACCEPT
```
* verify from inside a VM:
```bash
vagrant ssh control -c 'ping -c2 8.8.8.8'
```
* run a playbook
```bash
ansible-playbook -l dbserver-vm playbooks/postgres.yaml
```
* full stack on the Vagrant VMs (database -> spring backend -> vue frontend)
```bash
ansible-playbook -l dbserver-vm  playbooks/postgres.yaml    # group dbservers
ansible-playbook -l appserver-vm playbooks/spring.yaml      # group appservers
ansible-playbook -l frontend-vm  playbooks/vuejs.yaml       # group frontservers
```
Each playbook targets its own group (`hosts:`), so `-l` only limits the run to the Vagrant VMs and leaves out the cloud hosts of the same group.
Links:
* [Vagrant Quick start](https://learn.hashicorp.com/collections/vagrant/getting-started)

## Vault
* create a file that holds the **secret**
```bash
touch playbooks/vars/api_key.yml
```
* encrypt the file
```bash
ansible-vault encrypt playbooks/vars/api_key.yml
```
* run task that needs this file
```bash
ansible-playbook playbooks/use-api-key.yaml --ask-vault-pass
```
and you will be asked to provide the password
* edit the encrypoted file with
```bash
ansible-vault edit playbooks/vars/api_key.ym
```
* use stored password to decrypt
  create a file that holds the password with 600 permissions
```bash
vim ~/.ansible/vault_pass.txt
chmod 600 ~/.ansible/vault_pass.txt
```
```bash
ansible-playbook playbooks/use-api-key.yaml --vault-password-file  ~/.ansible/vault_pass.txt
```
Links:
* [Encrypting content with Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
## Ignore host_key_checking

add this line to ``ansible.cfg`` in [defaults] section
```ini
host_key_checking = false
```
Links:
* [Managing host key checking](https://docs.ansible.com/ansible/latest/user_guide/connection_details.html)


## Get host basic info
```bash
ansible-playbook -l <hostname> playbooks/hostvars_and_facts.yml
```

## postgres from ansible-galaxy
install postgresql role
```bash
ansible-galaxy install geerlingguy.postgresql
```

## Kubernetes

install python kubernetes
```bash
sudo apt install python3-kubernetes

```
## Links
* [apt module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
* [file module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
* [copy module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
* [service module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)
* [debconf module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debconf_module.html)
* [ansible postgres role](https://galaxy.ansible.com/geerlingguy/postgresql)
