# Semaphore UI

Το [Semaphore](https://semaphoreui.com) είναι ένα web περιβάλλον για το Ansible. Αντί για `ansible-playbook` στο τερματικό:
* βλέπεις το project (playbooks, inventory) σε ένα UI
* τρέχεις τα playbooks με ένα κουμπί και βλέπεις την έξοδο live
* κρατιέται ιστορικό: ποιος έτρεξε τι, πότε, με ποιο αποτέλεσμα
* τα secrets (ssh keys, κωδικοί) αποθηκεύονται κρυπτογραφημένα

Το Semaphore τρέχει σε docker και "βλέπει" αυτό το repository όπως είναι στον δίσκο σου.
Ό,τι αλλάζεις στα αρχεία φαίνεται αμέσως στο επόμενο task, χωρίς git commit/push.

## Προαπαιτούμενα
* docker και docker compose
* τα Vagrant VMs σηκωμένα (δες [README.md](README.md#run-development-environment-with-vagrant)):
```bash
cd vagrant
vagrant up control
vagrant up
```

## 1. Εκκίνηση
Από τη ρίζα του project:
```bash
docker compose up -d
docker compose ps          # postgres: healthy, semaphore: Up
```
Άνοιξε το http://localhost:3000 και συνδέσου με **admin / changeme**.

Σταμάτημα:
```bash
docker compose down        # τα δεδομένα του Semaphore μένουν (volume semaphore-db)
docker compose down -v     # σβήνει και τα δεδομένα (projects, keys, ιστορικό)
```

### Τι κάνει το [docker-compose.yaml](docker-compose.yaml)
| Ρύθμιση | Γιατί |
|---|---|
| `postgres` | η βάση όπου το Semaphore κρατά projects, keys, ιστορικό |
| `network_mode: host` (και στα δύο) | ο semaphore πρέπει να φτάνει στα VMs (`192.168.56.x`). Από το δίκτυο του docker δεν φτάνει, γιατί το firewall του libvirt κόβει την κίνηση. Στο δίκτυο του host τα βλέπει όπως τα βλέπεις κι εσύ. |
| postgres στο `127.0.0.1:5433` | αφού είναι στο δίκτυο του host: μόνο τοπικά (όχι ανοιχτή στο δίκτυο) και σε άλλη πόρτα από μια τυχόν postgres του host |
| `extra_hosts` | το `hosts.yaml` έχει `ansible_host: appserver`, `dbserver`, `frontserver`. Μέσα στον container αυτά τα ονόματα δεν υπάρχουν, οπότε τα αντιστοιχίζουμε στις σταθερές IP του `Vagrantfile`. |
| `.:/opt/ansible-devops:ro` | το project μέσα στον container, μόνο για ανάγνωση |
| `SEMAPHORE_ACCESS_KEY_ENCRYPTION` | το κλειδί με το οποίο κρυπτογραφούνται τα secrets στη βάση. Για δική σου εγκατάσταση φτιάξε νέο: `head -c32 /dev/urandom \| base64` |
| `image: semaphoreui/semaphore:v2.19.17` | συγκεκριμένη έκδοση και όχι `latest`, ώστε όλοι να έχουν την ίδια |

> Οι κωδικοί (`changeme`, `semaphore_pass`) είναι για το εργαστήριο. Σε πραγματική εγκατάσταση μπαίνουν σε αρχείο `.env` που δεν ανεβαίνει στο git.

## 2. SSH key για τα VMs
Το Semaphore συνδέεται στα VMs με ssh, άρα χρειάζεται ένα private key που να το δέχονται **όλα** τα VMs.
Το Vagrant φτιάχνει διαφορετικό κλειδί για κάθε VM, οπότε φτιάχνουμε ένα δικό μας κλειδί για το Semaphore
και βάζουμε το public key στα VMs, με ansible (από τον host, όπου η σύνδεση ήδη δουλεύει):
```bash
ssh-keygen -t ed25519 -N "" -C semaphore -f ~/.ssh/semaphore

ansible appserver-vm,dbserver-vm,frontend-vm -m ansible.posix.authorized_key \
  -a "user={{ ansible_user_id }} key={{ lookup('file', '~/.ssh/semaphore.pub') }}"
```
Το `authorized_key` προσθέτει το key στο `~/.ssh/authorized_keys` του χρήστη `vagrant` μόνο αν δεν υπάρχει ήδη.
Αν ξαναφτιάξεις τα VMs (`vagrant destroy` / `vagrant up`), ξανατρέχεις την εντολή `ansible ...`.

## 3. Στήσιμο του project στο UI
Με αυτή τη σειρά, γιατί κάθε βήμα χρησιμοποιεί τα προηγούμενα.

**3.1 Project.** *New Project*, όνομα `ansible-devops`.

**3.2 Key Store** → *New Key*
* Name: `vagrant`
* Type: **SSH Key**
* Username (login): `vagrant`
* Private Key: το περιεχόμενο του `~/.ssh/semaphore` (`cat ~/.ssh/semaphore`), μαζί με τις γραμμές `-----BEGIN ...` / `-----END ...`

Υπάρχει ήδη και ένα key `None`. Το χρησιμοποιούμε όπου δεν χρειάζεται κλειδί.

**3.3 Repositories** → *New Repository*
* Name: `ansible-devops (local)`
* URL or path: `/opt/ansible-devops` (ο φάκελος από το `volumes:` του compose)
* Branch: κενό
* Access Key: `None`

Επειδή είναι τοπικός φάκελος και όχι git URL, το Semaphore δεν κάνει clone. Χρησιμοποιεί τα αρχεία όπως είναι.

**3.4 Inventory** → *New Inventory*
* Name: `hosts.yaml`
* User Credentials: `vagrant` (το key του 3.2)
* Type: **File**
* Repository: `ansible-devops (local)`
* Path: `hosts.yaml`

Επειδή το inventory είναι αρχείο μέσα στο repository, το Ansible βρίσκει δίπλα του και τα `group_vars/` και `host_vars/`, όπως όταν τρέχεις από το τερματικό.

**3.5 Variable Groups** (σε παλιότερες εκδόσεις: *Environment*) → *New Variable Group*
* Name: `default`
* Extra variables: `{}`

Εδώ μπορείς να βάλεις μεταβλητές που θα περνούν σαν `-e`, π.χ. `{"app_start_timeout": 600}`.

**3.6 Task Templates** → *New Template*, ένα για κάθε playbook. Για τα Vagrant VMs:

| Name | Playbook filename | CLI args |
|---|---|---|
| `postgres` | `playbooks/postgres.yaml` | `["-l", "dbserver-vm"]` |
| `spring` | `playbooks/spring.yaml` | `["-l", "appserver-vm"]` |
| `vuejs` | `playbooks/vuejs.yaml` | `["-l", "frontend-vm"]` |
| `docker` | `playbooks/docker.yaml` | `["-l", "frontend-vm"]` |
| `facts` | `playbooks/hostvars_and_facts.yml` | `["-l", "dbserver-vm"]` |

Σε όλα: Inventory `hosts.yaml`, Repository `ansible-devops (local)`, Variable Group `default`.

* Το `-l` χρειάζεται γιατί κάθε group του `hosts.yaml` έχει και hosts στο cloud (`devops-1`, `azure-db-server`, ...). Χωρίς αυτό, το playbook θα προσπαθούσε να συνδεθεί και σε αυτούς.
* Αν τσεκάρεις **Allow CLI args in Task** (ή **Prompt: CLI args**), κάθε φορά που πατάς *Run* μπορείς να δώσεις άλλα args, π.χ. `["-l", "devops-1"]` ή `["--check", "--diff"]`. Αλλιώς τα args που δίνεις στο task **αγνοούνται** και ισχύουν μόνο αυτά του template.
* Για έλεγχο χωρίς αλλαγές: ένα template με CLI args `["--list-hosts"]` ή `["--check", "--diff"]`.

## 4. Εκτέλεση
Πάτα *Run* στο template. Η έξοδος είναι ίδια με του `ansible-playbook` και μένει στο *History*.

Για όλο το stack, με τη σειρά: **postgres → spring → vuejs**.

## Τι δεν δουλεύει σε αυτό το στήσιμο
* `playbooks/postgres-from-role.yaml`: χρειάζεται το role `geerlingguy.postgresql`, που είναι εγκατεστημένο στον host (`~/.ansible/roles`) και όχι στον container.
* `playbooks/k8s-update-spring-deployment.yaml`: τρέχει στο `localhost`, δηλαδή μέσα στον container, και χρειάζεται kubeconfig.
* Οι cloud hosts (`devops-1`, `gcloud-app-server`, ...) χρειάζονται δικό τους SSH key στο Key Store και δικό τους inventory.

## Προβλήματα
| Σύμπτωμα | Αιτία / λύση |
|---|---|
| `UNREACHABLE ... Permission denied (publickey)` | Το key του Key Store δεν είναι στα VMs: ξανατρέξε την εντολή `ansible ... authorized_key` του βήματος 2. Έλεγξε ότι το Username του key είναι `vagrant`. |
| `UNREACHABLE ... Connection timed out` | Τα VMs δεν τρέχουν (`vagrant status`), ή άλλαξαν IP στο `Vagrantfile` και πρέπει να αλλάξουν και στο `extra_hosts`. |
| `Could not match supplied host pattern` | Λάθος όνομα στο `-l`. Τα ονόματα είναι αυτά του `hosts.yaml` (`dbserver-vm`), όχι του Vagrant (`dbserver`). |
| Το Semaphore δεν ανοίγει | `docker compose logs semaphore`. Αν η πόρτα 3000 είναι πιασμένη, άλλαξε το `SEMAPHORE_PORT`, π.χ. `":3001"`. |
| Windows / macOS (Docker Desktop) | Το `network_mode: host` χρειάζεται Docker Desktop 4.34+ με ενεργοποιημένο το *host networking* στις ρυθμίσεις. |
