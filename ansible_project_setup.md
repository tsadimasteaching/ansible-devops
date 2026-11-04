# Στήσιμο Ansible project

Ο οδηγός περιγράφει πώς φτιάχνουμε από την αρχή ένα Ansible project σαν αυτό το repository:

1. τη δομή των φακέλων,
2. το `ansible.cfg`,
3. το inventory (`hosts.yaml`),
4. τη σύνδεση σε ένα VM στην Azure, με δύο τρόπους:
   - **Επιλογή Α:** όλες οι παράμετροι σύνδεσης γράφονται στο inventory,
   - **Επιλογή Β:** οι παράμετροι σύνδεσης γράφονται στο `~/.ssh/config` και το inventory κρατά μόνο ένα alias.

---

## 1. Προαπαιτούμενα

Στο μηχάνημα από το οποίο τρέχουμε το Ansible (ο **control node**):

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install ansible

# εναλλακτικά με pip
python3 -m pip install --user ansible

ansible --version
```

Στα μηχανήματα που διαχειριζόμαστε (οι **managed nodes**) χρειάζονται μόνο:

* πρόσβαση με **SSH**,
* **Python 3** (το Ubuntu το έχει ήδη εγκατεστημένο),
* ένας χρήστης με δικαίωμα `sudo`, αν θέλουμε `become: yes`.

---

## 2. Δομή του project

```text
ansible-devops/
├── ansible.cfg              # ρυθμίσεις του Ansible για αυτό το project
├── hosts.yaml               # inventory: ποια μηχανήματα και σε ποια groups
├── group_vars/              # μεταβλητές ανά group
│   ├── all.yaml             #   ισχύουν για όλους τους hosts
│   ├── appservers.yaml      #   ισχύουν για το group appservers
│   └── dbservers.yaml       #   ισχύουν για το group dbservers
├── host_vars/               # μεταβλητές ανά host (το όνομα αρχείου = όνομα host)
│   ├── appserver-vm.yaml
│   └── dbserver-vm.yaml
├── files/                   # templates (.j2) και αρχεία προς αντιγραφή
├── playbooks/               # τα playbooks
└── vagrant/                 # τοπικό περιβάλλον δοκιμών
```

Το Ansible διαβάζει αυτόματα τους φακέλους `group_vars/` και `host_vars/`, αρκεί να βρίσκονται δίπλα στο inventory. Δεν χρειάζεται να τους δηλώσουμε κάπου.

```bash
mkdir -p ansible-devops/{group_vars,host_vars,playbooks,files}
cd ansible-devops
```

---

## 3. `ansible.cfg`

Το Ansible ψάχνει το αρχείο ρυθμίσεων με την εξής σειρά και κρατά το **πρώτο** που βρίσκει:

1. τη μεταβλητή περιβάλλοντος `ANSIBLE_CONFIG`,
2. το `./ansible.cfg` στον τρέχοντα φάκελο,
3. το `~/.ansible.cfg`,
4. το `/etc/ansible/ansible.cfg`.

Γι' αυτό τρέχουμε πάντα τις εντολές `ansible` / `ansible-playbook` **από τη ρίζα του project**.

Το ελάχιστο που χρειαζόμαστε (αυτό που υπάρχει στο repo):

```ini
[defaults]
inventory = ./hosts.yaml
allow_world_readable_tmpfiles = true
```

Μια πιο πλήρης εκδοχή, με σχόλια:

```ini
[defaults]
# ποιο inventory χρησιμοποιείται αν δεν δώσουμε -i
inventory = ./hosts.yaml

# μην ρωτάς "Are you sure you want to continue connecting (yes/no)?"
# βολικό για VMs που ξαναδημιουργούνται συχνά (Vagrant, cloud)
host_key_checking = false

# χρειάζεται όταν κάνουμε become_user σε μη-root χρήστη (π.χ. postgres)
allow_world_readable_tmpfiles = true

# default python στους remote hosts (εναλλακτικά στο group_vars/all.yaml)
interpreter_python = auto_silent

# πιο ευανάγνωστο output
stdout_callback = yaml

# πόσοι hosts εκτελούνται παράλληλα
forks = 10

# πού βρίσκονται τα roles (π.χ. από ansible-galaxy)
roles_path = ./roles:~/.ansible/roles

# δεν δημιουργούνται αρχεία .retry
retry_files_enabled = false

[ssh_connection]
# επαναχρησιμοποίηση της SSH σύνδεσης, για ταχύτερη εκτέλεση
pipelining = true
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

Για να δούμε ποιο `ansible.cfg` χρησιμοποιείται τελικά:

```bash
ansible --version          # η γραμμή "config file = ..."
ansible-config dump --only-changed
```

---

## 4. Inventory (`hosts.yaml`)

Το inventory λέει στο Ansible **ποια μηχανήματα** διαχειρίζεται και σε **ποια groups** ανήκει το καθένα.

```yaml
---
<group>:                  # όνομα group
  hosts:                  # οι hosts του group
    <inventory_hostname>: # το όνομα με το οποίο ο host είναι γνωστός στο Ansible
      <μεταβλητή>: <τιμή> # μεταβλητές του host (π.χ. ansible_host)
  vars:                   # μεταβλητές κοινές για όλο το group
    <μεταβλητή>: <τιμή>
  children:               # (προαιρετικά) υπο-groups
    <άλλο_group>:
```

Βασικά σημεία:

* Το **`inventory_hostname`** (π.χ. `azure-db-server`) είναι ένα λογικό όνομα. Με αυτό το όνομα ψάχνει το Ansible το αρχείο `host_vars/<όνομα>.yaml` και αυτό δίνουμε στο `-l` / `--limit`.
* Το **`ansible_host`** είναι **πού** θα συνδεθεί το SSH: μια IP, ένα DNS όνομα ή ένα alias του `~/.ssh/config`.
* Ένας host μπορεί να ανήκει σε **περισσότερα από ένα groups**. Στο repo, για παράδειγμα, ο `azure-db-server` είναι και στο `appservers` και στο `dbservers`.
* Υπάρχουν πάντα δύο έμμεσα groups: το `all` (όλοι οι hosts) και το `ungrouped`.

Το inventory του repo:

```yaml
---
control:
  hosts:
    local-control:
      ansible_connection: local     # εκτέλεση τοπικά, χωρίς SSH
    control-vm:
      ansible_host: control

appservers:
  hosts:
    appserver-vm:
      ansible_host: appserver
    frontend-vm:
      ansible_host: frontserver
    gcloud-app-server:
      ansible_host: devops-gcloud
    azure-db-server:
      ansible_host: devops-azure

dbservers:
  hosts:
    dbserver-vm:
      ansible_host: dbserver
    azure-db-server:
      ansible_host: devops-azure
```

Εδώ όλα τα `ansible_host` είναι **aliases του `~/.ssh/config`** (Επιλογή Β παρακάτω).

### Συνηθισμένες μεταβλητές σύνδεσης

| Μεταβλητή | Τι ορίζει | Παράδειγμα |
|---|---|---|
| `ansible_host` | IP, DNS ή ssh alias προς σύνδεση | `20.86.12.34` |
| `ansible_port` | θύρα SSH (default 22) | `22` |
| `ansible_user` | χρήστης SSH (παλαιότερο όνομα: `ansible_ssh_user`) | `azureuser` |
| `ansible_ssh_private_key_file` | ιδιωτικό κλειδί | `~/.ssh/devops-azure_key.pem` |
| `ansible_ssh_common_args` | επιπλέον ορίσματα στο `ssh` | `-o StrictHostKeyChecking=no` |
| `ansible_connection` | τύπος σύνδεσης | `ssh` (default), `local` |
| `ansible_python_interpreter` | python στον remote host | `/usr/bin/python3` |
| `ansible_become` | χρήση `sudo` | `true` |
| `ansible_become_password` | κωδικός sudo, αν χρειάζεται (καλύτερα σε Vault) | — |

---

## 5. Δημιουργία VM στην Azure (γρήγορα)

Από το Azure Portal → **Virtual machines** → **Create**:

* **Image:** Ubuntu Server 22.04 LTS ή 24.04 LTS
* **Authentication type:** *SSH public key*
* **Username:** π.χ. `azureuser`
* **SSH public key source:**
  * *Generate new key pair*: η Azure δίνει για download ένα `.pem` (π.χ. `devops-azure_key.pem`), ή
  * *Use existing public key*: επικολλάμε το `~/.ssh/id_ed25519.pub`
* **Inbound ports:** SSH (22). Για όσες υπηρεσίες θα στήσουμε ανοίγουμε ports στο Network Security Group, π.χ. `5432` για PostgreSQL, `8080` για Spring, `80` για nginx.

Μετά τη δημιουργία κρατάμε τη **Public IP** του VM.

Αν η Azure έδωσε `.pem`:

```bash
mv ~/Downloads/devops-azure_key.pem ~/.ssh/
chmod 600 ~/.ssh/devops-azure_key.pem   # αλλιώς το ssh απορρίπτει το κλειδί
```

Πρώτα δοκιμάζουμε ότι το απλό ssh δουλεύει:

```bash
ssh -i ~/.ssh/devops-azure_key.pem azureuser@20.86.12.34
```

> Αν δεν δουλεύει το απλό `ssh`, δεν θα δουλέψει ούτε το Ansible. Πάντα ελέγχουμε πρώτα αυτό.

Στα παραδείγματα που ακολουθούν:

* Public IP: `20.86.12.34`
* User: `azureuser`
* Key: `~/.ssh/devops-azure_key.pem`

---

## 6. Επιλογή Α: παράμετροι σύνδεσης στο inventory

Όλη η πληροφορία σύνδεσης γράφεται μέσα στο `hosts.yaml`.

```yaml
---
dbservers:
  hosts:
    azure-db-server:
      ansible_host: 20.86.12.34
      ansible_port: 22
      ansible_user: azureuser
      ansible_ssh_private_key_file: ~/.ssh/devops-azure_key.pem
      # προαιρετικά: μην κάνεις έλεγχο host key (αν δεν το έχουμε στο ansible.cfg)
      ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

Αν έχουμε **πολλά VMs στην Azure με τον ίδιο χρήστη και το ίδιο κλειδί**, βάζουμε τις κοινές τιμές στα `vars` του group:

```yaml
---
azure:
  hosts:
    azure-db-server:
      ansible_host: 20.86.12.34
    azure-app-server:
      ansible_host: 20.86.12.35
  vars:
    ansible_user: azureuser
    ansible_ssh_private_key_file: ~/.ssh/devops-azure_key.pem

dbservers:
  hosts:
    azure-db-server:

appservers:
  hosts:
    azure-app-server:
```

Οι ίδιες μεταβλητές μπορούν να μπουν και σε αρχεία, για πιο καθαρό inventory:

```yaml
# host_vars/azure-db-server.yaml
ansible_host: 20.86.12.34
ansible_user: azureuser
ansible_ssh_private_key_file: ~/.ssh/devops-azure_key.pem
```

```yaml
# group_vars/azure.yaml
ansible_user: azureuser
ansible_ssh_private_key_file: ~/.ssh/devops-azure_key.pem
```

**Υπέρ**

* Όλα βρίσκονται μέσα στο project και φαίνονται σε όποιον το ανοίξει.
* Δεν εξαρτάται από ρυθμίσεις του υπολογιστή μας.

**Κατά**

* Το απλό `ssh` δεν ξέρει αυτές τις ρυθμίσεις, οπότε για χειροκίνητη σύνδεση γράφουμε όλη την εντολή με `-i ... user@ip`.
* Οι διαδρομές κλειδιών και οι IPs είναι προσωπικές και καταλήγουν στο git.
* Αν αλλάξει η IP (π.χ. VM χωρίς static IP μετά από stop/start), την αλλάζουμε στο inventory.

---

## 7. Επιλογή Β: παράμετροι στο `~/.ssh/config` (αυτή που χρησιμοποιεί το repo)

Οι ρυθμίσεις σύνδεσης γράφονται **μία φορά** στο `~/.ssh/config`, κάτω από ένα alias. Το Ansible χρησιμοποιεί το OpenSSH, άρα διαβάζει αυτόματα το `~/.ssh/config`.

### 7.1 `~/.ssh/config`

```ssh-config
Host devops-azure
    HostName 20.86.12.34
    User azureuser
    Port 22
    IdentityFile ~/.ssh/devops-azure_key.pem
    IdentitiesOnly yes
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host devops-gcloud
    HostName 35.189.109.16
    User rg
    IdentityFile ~/.ssh/google_compute_engine
```

```bash
chmod 600 ~/.ssh/config
ssh devops-azure          # πρέπει να συνδεθεί χωρίς άλλα ορίσματα
```

### 7.2 `hosts.yaml`

Στο inventory μένει μόνο το alias:

```yaml
---
dbservers:
  hosts:
    azure-db-server:
      ansible_host: devops-azure
```

Ακόμα πιο απλά: αν το `inventory_hostname` είναι το ίδιο το alias, δεν χρειάζεται καν `ansible_host`:

```yaml
dbservers:
  hosts:
    devops-azure:
```

### 7.3 Τα Vagrant VMs με τον ίδιο τρόπο

Τα τοπικά VMs του repo (`appserver`, `dbserver`, `frontserver`) μπαίνουν στο `~/.ssh/config` με μία εντολή:

```bash
cd vagrant
vagrant up
vagrant ssh-config >> ~/.ssh/config
```

Η εντολή γράφει entries σαν το παρακάτω:

```ssh-config
Host dbserver
  HostName 127.0.0.1
  User vagrant
  Port 2201
  IdentityFile /path/to/.vagrant/machines/dbserver/virtualbox/private_key
  StrictHostKeyChecking no
  ...
```

Γι' αυτό το inventory έχει απλώς `ansible_host: dbserver`.

**Υπέρ**

* Το ίδιο alias δουλεύει παντού: `ssh devops-azure`, `scp`, VS Code Remote-SSH και Ansible.
* Το inventory είναι καθαρό και δεν περιέχει προσωπικά paths κλειδιών ή IPs.
* Αν αλλάξει η IP, την αλλάζουμε σε ένα μόνο σημείο.

**Κατά**

* Το project δεν είναι πια αυτόνομο: κάθε μέλος της ομάδας πρέπει να έχει τα **ίδια aliases** στο δικό του `~/.ssh/config`.

> **Μπορούμε να τα συνδυάσουμε.** Οι μεταβλητές του inventory (`ansible_user`, `ansible_ssh_private_key_file`, ...) μετατρέπονται σε ορίσματα της εντολής `ssh` και **υπερισχύουν** του `~/.ssh/config`. Έτσι μπορούμε π.χ. να κρατάμε το alias στο `~/.ssh/config` και να αλλάζουμε μόνο τον χρήστη από το inventory.

---

## 8. Μεταβλητές σε `group_vars` / `host_vars`

Μεταβλητές που δεν αφορούν τη σύνδεση (ρυθμίσεις εφαρμογής, βάσης κ.λπ.) τις κρατάμε έξω από το inventory:

```yaml
# group_vars/all.yaml  -> για όλους τους hosts
ansible_python_interpreter: /usr/bin/python3
```

```yaml
# group_vars/dbservers.yaml -> για το group dbservers
db:
  name: "appdb"
  user: "appuser"
  password: "apppassword"
```

```yaml
# host_vars/azure-db-server.yaml -> μόνο για αυτόν τον host
db:
  name: "appdb"
  user: "appuser"
  password: "{{ vault_db_password }}"   # το μυστικό στο Ansible Vault (βλ. README)
```

Προτεραιότητα, από τη χαμηλότερη στην υψηλότερη (απλοποιημένα):
`group_vars/all` → `group_vars/<group>` → `host_vars/<host>` → `vars` του playbook → `-e` στη γραμμή εντολών.

> Αν ένα dictionary (π.χ. `db`) οριστεί και στο `group_vars` και στο `host_vars`, κρατιέται **ολόκληρο** το πιο ειδικό. Τα πεδία των δύο δεν συγχωνεύονται.

---

## 9. Έλεγχος ότι όλα δουλεύουν

```bash
# δομή του inventory
ansible-inventory --graph

# όλες οι μεταβλητές ενός host, όπως τις βλέπει το Ansible
ansible-inventory --host azure-db-server

# ping (το module ping του Ansible, όχι ICMP) σε όλους / σε group / σε host
ansible all -m ping
ansible dbservers -m ping
ansible azure-db-server -m ping

# ad-hoc εντολή
ansible azure-db-server -a "uname -a"
ansible azure-db-server -b -a "apt list --upgradable"   # -b = become (sudo)

# εκτέλεση playbook σε συγκεκριμένο host
ansible-playbook -l azure-db-server playbooks/postgres-16.yaml

# facts και hostvars ενός host
ansible-playbook -l azure-db-server playbooks/hostvars_and_facts.yml
```

Επιτυχημένο ping:

```text
azure-db-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 10. Συνηθισμένα προβλήματα

| Σύμπτωμα | Πιθανή αιτία / λύση |
|---|---|
| `UNREACHABLE ... Permission denied (publickey)` | λάθος `ansible_user` ή κλειδί. Δοκιμάζουμε πρώτα `ssh -v devops-azure` |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | `chmod 600 ~/.ssh/<key>.pem` |
| `Connection timed out` | λάθος IP, το VM είναι σταματημένο ή το NSG της Azure δεν επιτρέπει την port 22 |
| `Host key verification failed` | το VM ξαναδημιουργήθηκε με την ίδια IP. `ssh-keygen -R 20.86.12.34` ή `host_key_checking = false` |
| `Missing sudo password` | ο χρήστης θέλει κωδικό για sudo: `--ask-become-pass` (`-K`) |
| `Failed to set permissions on the temporary files ... become_user` | `allow_world_readable_tmpfiles = true` στο `ansible.cfg` (ή εγκατάσταση του `acl` στον host) |
| Τρέχει με λάθος inventory | δεν τρέχουμε από τη ρίζα του project, άρα δεν διαβάστηκε το `./ansible.cfg` |
| Δεν βρίσκει το alias `devops-azure` | λείπει από το `~/.ssh/config` αυτού του χρήστη/υπολογιστή |

---

## Links

* [How to build your inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
* [Connection variables (behavioral inventory parameters)](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#connecting-to-hosts-behavioral-inventory-parameters)
* [Ansible configuration settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
* [Variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)
* [Connect to a Linux VM in Azure with SSH](https://learn.microsoft.com/en-us/azure/virtual-machines/linux-vm-connect)
* `man ssh_config`
