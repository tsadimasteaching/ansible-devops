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
ansible -m ping all <group-name>
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
