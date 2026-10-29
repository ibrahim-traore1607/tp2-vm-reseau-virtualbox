# Compte rendu TP 2 — Deux machines virtuelles en réseau avec VirtualBox

**Cours :** Introduction au Cloud · L3 SI RIST & BDGL  
**Étudiant / trinôme :** TRAORE IBRAHIM — KOFFI BONI — DJANGORAN CHRISTIAN  
**Professeur :** Dr. Bakayoko

**Livrable :** commandes lancées, réponse de chaque commande, réponses aux questions.

> Invite hôte : `PS C:\...>`. Invite VM : `etudiant@web:~$` ou `etudiant@client:~$`.  
> Nom VirtualBox du modèle : `**ubuntu-s**` (nom d'hôte Linux d'origine : `ubuntu-srv`). Les commandes `VBoxManage` utilisent donc `ubuntu-s` à la place de `ubuntu-srv` de l'énoncé.

---

## Environnement

| Élément      | Valeur                                                          |
| ------------ | --------------------------------------------------------------- |
| Hôte         | Windows 11, PowerShell, utilisateur `HP`                        |
| Hyperviseur  | Oracle VirtualBox 7.2.20r175154 (type 2)                        |
| Modèle       | VM `ubuntu-s` du TP 1 (Ubuntu Server 26.04.1 LTS, user `etudiant`) |
| Clones       | `web` et `client` (1 vCPU, 1 Go de RAM chacun)                  |
| Réseau       | NAT Network `tp-cloud`, `10.10.0.0/24`, DHCP désactivé          |

### Architecture réalisée

| Machine          | Adresse IP      | Accès depuis l'hôte                | Rôle                           |
| ---------------- | --------------- | ---------------------------------- | ------------------------------ |
| `web`            | `10.10.0.10/24` | `ssh -p 2222 etudiant@127.0.0.1`   | Serveur Nginx, pare-feu `ufw`  |
| `client`         | `10.10.0.20/24` | `ssh -p 2223 etudiant@127.0.0.1`   | Poste de test (ping, curl)     |
| Routeur NAT      | `10.10.0.1`     | —                                  | Passerelle, DNS, redirections  |
| Navigateur hôte  | —               | `http://127.0.0.1:8080`            | Redirigé vers `web:80`         |

---

## Partie 1 — Créer les VM par clonage

### 1.1 Cloner l'image et créer le réseau

**Commande**

```powershell
$env:Path += ";C:\Program Files\Oracle\VirtualBox"
VBoxManage controlvm ubuntu-s acpipowerbutton
VBoxManage list runningvms
```

**Réponse.** Aucune sortie *(sortie attendue pour `controlvm`)* : le dossier VirtualBox est ajouté au `PATH` de la session, `ubuntu-s` reçoit le signal d'arrêt propre (ou l'erreur est ignorée si elle est déjà éteinte) et la liste des VM en marche est vide.

**Commande** (clonage de `web`, chronométré)

```powershell
Measure-Command { VBoxManage clonevm ubuntu-s --name web --register } | Select-Object TotalSeconds
```

**Réponse**

```text
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%

TotalSeconds
------------
   28,750629
```

**Lecture.** Le clonage de `web` a pris **28,75 secondes**.

**Commandes** (clone `client`, puis réduction à 1 vCPU / 1 Go)

```powershell
VBoxManage clonevm ubuntu-s --name client --register
VBoxManage modifyvm web --cpus 1 --memory 1024
VBoxManage modifyvm client --cpus 1 --memory 1024
```

**Réponse**

```text
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Machine has been successfully cloned as "client"
```

**Lecture.** Les deux `modifyvm` n'affichent rien : la configuration est appliquée (vérifiée plus bas avec `showvminfo`).

**Commandes** (réseau NAT `tp-cloud` et ses trois redirections)

```powershell
VBoxManage natnetwork add --netname tp-cloud --network "10.10.0.0/24" --enable --dhcp off
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "ssh-web:tcp:[127.0.0.1]:2222:[10.10.0.10]:22"
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "ssh-client:tcp:[127.0.0.1]:2223:[10.10.0.20]:22"
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "http-web:tcp:[127.0.0.1]:8080:[10.10.0.10]:80"
VBoxManage natnetwork list
```

**Réponse** (`natnetwork list`)

```text
NAT Networks:

Name:         tp-cloud
Enabled:      Yes
Network:      10.10.0.0/24
Gateway:      10.10.0.1
DHCP Server:  No
IPv6:         No
IPv6 Prefix:  fd17:625c:f037:2::/64
IPv6 Default: No
Port-forwarding (ipv4)
        http-web:tcp:[127.0.0.1]:8080:[10.10.0.10]:80
        ssh-client:tcp:[127.0.0.1]:2223:[10.10.0.20]:22
        ssh-web:tcp:[127.0.0.1]:2222:[10.10.0.10]:22
loopback mappings (ipv4)
        127.0.0.1=2

1 network found
```

**Lecture.** Réseau `10.10.0.0/24` actif, passerelle `10.10.0.1`, **pas de DHCP** (adresses fixes), trois règles de redirection (2222 et 2223 pour SSH, 8080 pour HTTP).

**Commandes** (branchement des cartes réseau et vérification)

```powershell
VBoxManage modifyvm web --nic1 natnetwork --nat-network1 tp-cloud
VBoxManage modifyvm client --nic1 natnetwork --nat-network1 tp-cloud
VBoxManage showvminfo web | Select-String "Memory size|Number of CPUs|NIC 1"
```

**Réponse**

```text
Memory size:                 1024MB
Number of CPUs:              1
NIC 1:                       MAC: 080027BDEEF2, Attachment: NAT Network 'tp-cloud', Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
```

**Lecture.** `web` a bien 1024 Mo, 1 CPU, et sa carte est attachée au `NAT Network 'tp-cloud'` (câble connecté). La MAC `080027BDEEF2` est propre au clone.

### 1.2 Personnaliser chaque clone

**Commandes** (console de `web`)

```bash
sudo hostnamectl set-hostname web
sudo sed -i 's/ubuntu-srv/web/g' /etc/hosts
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A
```

**Réponse**

```text
Initializing machine ID from random generator.
ssh-keygen: generating new host keys: RSA ECDSA ED25519
```

**Lecture.**

- `hostnamectl` renomme la machine en `web` (visible dans le prompt après reconnexion : `etudiant@web`).
- `sed` met à jour `/etc/hosts` pour que l'ancien nom `ubuntu-srv` ne soit plus référencé.
- Un **nouveau `machine-id`** est généré (`Initializing machine ID from random generator`).
- Les **clés d'hôte SSH** RSA, ECDSA et ED25519 sont régénérées.

**Commandes** (IP fixe de `web`)

```bash
sudo tee /etc/netplan/99-tp.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.10.0.10/24]
      routes:
        - to: default
          via: 10.10.0.1
      nameservers:
        addresses: [10.10.0.1, 1.1.1.1]
EOF
sudo chmod 600 /etc/netplan/99-tp.yaml
sudo netplan apply
ip -4 addr show enp0s3
sudo reboot
```

**Réponse**

```text
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    altname enx080027bdeef2
    inet 10.10.0.10/24 brd 10.10.0.255 scope global enp0s3
       valid_lft forever preferred_lft forever
```

**Lecture.** `enp0s3` porte `**10.10.0.10/24`**. `valid_lft forever` confirme une adresse **statique** (pas de bail DHCP). `netplan apply` n'affiche aucune erreur. Après `sudo reboot` *(sortie attendue)*, la VM redémarre et le prompt affiche `etudiant@web:~$`.

**Étape 7 — même procédure sur `client`** : nom `client` (`sed 's/ubuntu-srv/client/g'`), nouveau `machine-id`, nouvelles clés d'hôte, et fichier Netplan avec `addresses: [10.10.0.20/24]`.

**Commandes** (depuis l'hôte : oubli des anciennes empreintes, puis vérification en SSH)

```powershell
ssh-keygen -R "[127.0.0.1]:2222"
ssh-keygen -R "[127.0.0.1]:2223"
ssh -p 2222 etudiant@127.0.0.1 "hostnamectl; lsb_release -a; uname -r"
ssh -p 2223 etudiant@127.0.0.1 "hostname; ip -4 -br addr"
```

**Réponse** (`ssh-keygen -R`)

```text
# Host [127.0.0.1]:2222 found: line 1
# Host [127.0.0.1]:2222 found: line 2
# Host [127.0.0.1]:2222 found: line 3
C:\Users\HP/.ssh/known_hosts updated.
Original contents retained as C:\Users\HP/.ssh/known_hosts.old
Host [127.0.0.1]:2223 not found in C:\\Users\\HP/.ssh/known_hosts
```

**Lecture.** Les 3 anciennes empreintes de `127.0.0.1:2222` (RSA, ECDSA, ED25519) sont supprimées. Sur `2223`, rien à supprimer (jamais utilisé).

**Réponse** (première connexion SSH : empreinte demandée, `yes`, puis mot de passe)

```text
The authenticity of host '[127.0.0.1]:2223 ([127.0.0.1]:2223)' can't be established.
ED25519 key fingerprint is SHA256:vsy1JXbXsV88J2ls9ExuN2DA1l6+ax7uvITXNy4swGM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:2223' (ED25519) to the list of known hosts.
```

**Réponse** (VM `web`, port 2222)

```text
 Static hostname: web
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 44b0126ac4d44f5182ab46924da4dc80
         Boot ID: da96d61de3a64018bf9cc7b91f1d1643
  Virtualization: oracle
Operating System: Ubuntu 26.04.1 LTS
          Kernel: Linux 7.0.0-38-generic
    Architecture: x86-64
 Hardware Vendor: innotek GmbH
  Hardware Model: VirtualBox
Hardware Version: 1.2
Distributor ID: Ubuntu
Description:    Ubuntu 26.04.1 LTS
Release:        26.04
Codename:       resolute
7.0.0-38-generic
```

**Réponse** (VM `client`, port 2223)

```text
client
lo               UNKNOWN        127.0.0.1/8
enp0s3           UP             10.10.0.20/24
```

**Lecture.** Les deux VM répondent en SSH avec leur propre nom (`web`, `client`), leurs IP fixes et des empreintes SSH **différentes**. Le `Machine ID` de `web` est nouveau.

### Réponses aux questions

**Q1. Combien de temps a pris le clonage de `web` ? Comparaison avec l'installation d'Ubuntu au TP précédent, puis avec la mise en service d'un serveur physique.**

Le clonage a pris **28,75 s**. L'installation d'Ubuntu au TP 1 avait pris **23 min 48 s** (1 428 s) : le clonage est donc **environ 50 fois plus rapide**. Un serveur physique demande commande, livraison, montage en baie, câblage puis installation : **plusieurs jours à plusieurs semaines**. Dans le cloud, on part d'une image déjà installée, d'où des instances disponibles en quelques minutes.

**Q2. Pourquoi changer le nom d'hôte, l'identifiant de machine et les clés SSH d'un clone ? Quel risque avec les mêmes clés SSH sur 50 serveurs ?**

Un clone est une copie exacte : sans changement, deux machines se présentent avec le **même nom**, le **même `machine-id`** (utilisé par `systemd`, `journald`, certains identifiants réseau) et les **mêmes clés d'hôte SSH**. Avec les mêmes clés sur 50 serveurs, **une seule clé privée volée permet d'usurper les 50** (attaque de l'homme du milieu), et un client ne peut plus distinguer un serveur d'un autre grâce à l'empreinte. C'est ce que fait **cloud-init** au premier démarrage d'une instance cloud.

**Q3. Version d'Ubuntu, noyau, vCPU et RAM de `web`. Quel service cloud joue le rôle de `ubuntu-srv` ?**

- Ubuntu **26.04.1 LTS** (`resolute`), noyau **Linux 7.0.0-38-generic**.
- **1 vCPU** (`Number of CPUs: 1`, `nproc` = 1).
- **1024 Mo** configurés, **956 Mi** visibles dans la VM (`free -h`).
- `ubuntu-s` / `ubuntu-srv` joue le rôle d'une **image de machine** : **AMI** (Amazon Machine Image) chez AWS, image de la *Compute Gallery* chez Azure, *image* chez Google Cloud.

---

## Partie 2 — Relier les VM en réseau

### 2.1 Connectivité depuis `client`

**Commandes** (depuis l'hôte, puis dans `client`)

```powershell
ssh -p 2223 etudiant@127.0.0.1
```

```bash
ping -c 4 10.10.0.10
ping -c 2 10.10.0.1
ping -c 2 1.1.1.1
ip route
resolvectl status | grep "DNS Servers"
```

**Réponse**

```text
PING 10.10.0.10 (10.10.0.10) 56(84) bytes of data.
64 bytes from 10.10.0.10: icmp_seq=1 ttl=64 time=13.1 ms
64 bytes from 10.10.0.10: icmp_seq=2 ttl=64 time=5.73 ms
64 bytes from 10.10.0.10: icmp_seq=3 ttl=64 time=6.84 ms
64 bytes from 10.10.0.10: icmp_seq=4 ttl=64 time=7.03 ms

--- 10.10.0.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 5.728/8.171/13.093/2.884 ms

PING 10.10.0.1 (10.10.0.1) 56(84) bytes of data.
64 bytes from 10.10.0.1: icmp_seq=1 ttl=64 time=4.71 ms
64 bytes from 10.10.0.1: icmp_seq=2 ttl=64 time=10.1 ms

--- 10.10.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 4.707/7.411/10.115/2.704 ms

PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=64 time=33.1 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=64 time=36.2 ms

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 33.074/34.618/36.162/1.544 ms

default via 10.10.0.1 dev enp0s3 proto static
10.10.0.0/24 dev enp0s3 proto kernel scope link src 10.10.0.20

       DNS Servers: 10.10.0.1 1.1.1.1
```

**Lecture.**

- `client` joint `web` (0 % de perte, 8,17 ms de moyenne), la passerelle (7,41 ms) et Internet (34,6 ms).
- `ttl=64` : aucun routeur traversé entre `client` et `web`.
- La route par défaut passe par `10.10.0.1` (`proto static` : vient de Netplan). Les DNS sont ceux configurés (`10.10.0.1` et `1.1.1.1`).

### 2.2 Nommer `web` dans `/etc/hosts`

**Commande**

```bash
echo "10.10.0.10 web" | sudo tee -a /etc/hosts
ping -c 2 web
exit
```

**Réponse**

```text
10.10.0.10 web
PING web (10.10.0.10) 56(84) bytes of data.
64 bytes from web (10.10.0.10): icmp_seq=1 ttl=64 time=6.12 ms
64 bytes from web (10.10.0.10): icmp_seq=2 ttl=64 time=5.87 ms

--- web ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 5.870/5.995/6.120/0.125 ms
logout
Connection to 127.0.0.1 closed.
```

**Lecture.** *(sortie attendue)* Le nom `web` est résolu vers `10.10.0.10` grâce à la ligne ajoutée dans `/etc/hosts` ; le ping aboutit (0 % de perte, quelques ms), comme pour l'adresse IP.

### 2.3 Ping depuis Windows

**Commande**

```powershell
ping -n 2 10.10.0.10
```

**Réponse** *(sortie attendue)*

```text
Envoi d'une requête 'Ping'  10.10.0.10 avec 32 octets de données :
Délai d'attente de la demande dépassé.
Délai d'attente de la demande dépassé.

Statistiques Ping pour 10.10.0.10:
    Paquets : envoyés = 2, reçus = 0, perdus = 2 (perte 100%),
```

**Lecture.** Le ping depuis Windows **échoue** (100 % de perte) : l'hôte n'a pas de route vers `10.10.0.0/24`, voir Q4.

### Réponses aux questions

**Q1. Les deux VM sont-elles dans le même sous-réseau ?**

Oui. `web` = `10.10.0.10/24` et `client` = `10.10.0.20/24`. Le masque `/24` (`255.255.255.0`) fixe les trois premiers octets comme partie réseau : les deux ont le réseau `**10.10.0.0/24**` (plage `10.10.0.1` à `10.10.0.254`). Elles se joignent donc directement, sans routeur (`ttl=64`).

**Q2. Passerelle par défaut de `client`, rôle, équivalent AWS / Azure ?**

La passerelle est `**10.10.0.1**` (`default via 10.10.0.1`). Elle sert à **sortir du sous-réseau** : tout paquet destiné à une autre adresse (Internet) lui est envoyé ; elle assure aussi le NAT, le DNS et les redirections de port. Équivalent : le **routeur implicite du VPC** (AWS) avec l'**Internet Gateway / NAT Gateway** ; chez Azure, la **route système par défaut** d'un VNet et la **NAT Gateway**.

**Q3. Temps de réponse moyen du ping `client` → `web` ? Pourquoi si faible ?**

**8,17 ms** en moyenne (min 5,73 ms, max 13,09 ms). Les deux VM sont **sur la même machine physique**, reliées par un réseau **virtuel** : aucun câble, aucun routeur intermédiaire (`ttl=64`), seulement le moteur réseau de VirtualBox. Comparé à Internet (`1.1.1.1` : ~34,6 ms), c'est environ 4 fois plus rapide.

**Q4. Le ping depuis Windows vers `10.10.0.10` aboutit-il ? Comment l'hôte joint-il `web` en SSH ?**

Non : le réseau NAT `tp-cloud` est un **sous-réseau privé** que Windows ne route pas, et le NAT interdit les connexions entrantes non déclarées (ce qui correspond à un sous-réseau privé du cloud). L'hôte joint `web` uniquement grâce aux **redirections de port** : `127.0.0.1:2222` → `10.10.0.10:22` (SSH), et `127.0.0.1:8080` → `10.10.0.10:80` (HTTP).

---

## Partie 3 — Publier une page web et la protéger

### 3.1 Installer Nginx

**Commandes** (sur `web`)

```bash
sudo apt update && sudo apt install -y nginx
systemctl status nginx --no-pager
echo "<h1>trinome TRAORE - KOFFI - DJANGORAN : notre premiere VM en reseau</h1>" | sudo tee /var/www/html/index.html
```

**Réponse** (fin de l'installation)

```text
Paramétrage de nginx-common (1.28.3-2ubuntu1.11) ...
Created symlink '/etc/systemd/system/multi-user.target.wants/nginx.service' → '/usr/lib/systemd/system/nginx.service'.
Paramétrage de nginx (1.28.3-2ubuntu1.11) ...
 * Upgrading binary nginx                                      [ OK ]
Running kernel seems to be up-to-date.
No services need to be restarted.
```

**Réponse** (`systemctl status nginx --no-pager`)

```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running)
```

**Réponse** (`tee`)

```text
<h1>trinome TRAORE - KOFFI - DJANGORAN : notre premiere VM en reseau</h1>
```

**Lecture.** Nginx **1.28.3** est installé, **activé au démarrage** (lien symbolique `multi-user.target.wants`) et démarré (`active (running)`). La page d'accueil a été remplacée par celle du trinôme.

### 3.2 Fermer puis ouvrir le port 80 avec `ufw`

**Commandes** (sur `web` : SSH autorisé **avant** l'activation)

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

**Réponse**

```text
Rules updated
Rules updated (v6)
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
```

**Lecture.** Pare-feu actif : tout le trafic entrant est refusé par défaut, seul le port **22/tcp** est ouvert (IPv4 et IPv6). La session SSH n'a pas été coupée car OpenSSH avait été autorisé avant.

**Commande** (test depuis l'hôte, port 80 **fermé**)

```powershell
ssh -p 2223 etudiant@127.0.0.1 "curl -m 5 http://web; echo code retour: `$?"
```

**Réponse**

```text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0      0   0      0   0      0      0      0 --:--:--  00:01 ...  00:04
0
curl: (28) Connection timed out after 5002 milliseconds
code retour: 28
```

**Lecture.** `curl` attend 5 s sans recevoir un seul octet, puis abandonne : **code retour 28** (timeout). Le pare-feu **ignore silencieusement** les paquets vers le port 80.

**Commandes** (ouverture du port HTTP et vérification)

```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw status numbered
sudo ss -tlnp
```

**Réponse**

```text
Rule added
Rule added (v6)
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] OpenSSH                    ALLOW IN    Anywhere
[ 2] Nginx HTTP                 ALLOW IN    Anywhere
[ 3] OpenSSH (v6)               ALLOW IN    Anywhere (v6)
[ 4] Nginx HTTP (v6)            ALLOW IN    Anywhere (v6)

State    Recv-Q   Send-Q     Local Address:Port    Peer Address:Port   Process
LISTEN   0        4096          127.0.0.54:53           0.0.0.0:*      users:(("systemd-resolve",pid=559,fd=19))
LISTEN   0        4096       127.0.0.53%lo:53           0.0.0.0:*      users:(("systemd-resolve",pid=559,fd=17))
LISTEN   0        4096             0.0.0.0:22           0.0.0.0:*      users:(("sshd",pid=1351,fd=3),("systemd",pid=1,fd=100))
LISTEN   0        511              0.0.0.0:80           0.0.0.0:*      users:(("nginx",pid=1021,fd=5),("nginx",pid=1013,fd=5))
LISTEN   0        4096                [::]:22              [::]:*      users:(("sshd",pid=1351,fd=4),("systemd",pid=1,fd=101))
LISTEN   0        511                 [::]:80              [::]:*      users:(("nginx",pid=1021,fd=6),("nginx",pid=1013,fd=6))
```

**Commande** (retest depuis `client`, port 80 **ouvert**)

```powershell
ssh -p 2223 etudiant@127.0.0.1 "curl -s http://web"
```

**Réponse**

```text
<h1>trinome TRAORE - KOFFI - DJANGORAN : notre premiere VM en reseau</h1>
```

**Lecture.** La page du trinôme est servie : le nom `web` résout bien vers `10.10.0.10`, le pare-feu laisse passer HTTP, Nginx répond.

### Réponses aux questions

**Q1. Que renvoie `curl` quand le port 80 est fermé ? Pourquoi le navigateur reste-t-il bloqué ?**

`curl: (28) Connection timed out after 5002 milliseconds`, **code retour 28**. `ufw` applique la politique `deny` qui **jette** les paquets sans répondre (action DROP, pas REJECT). Le client ne reçoit **aucun message d'erreur** : il continue à réémettre et à attendre jusqu'au timeout, d'où le navigateur « bloqué ». Si Nginx n'avait pas tourné, on aurait eu un `Connection refused` immédiat.

**Q2. Port et adresses d'écoute de Nginx (`ss -tlnp`) ?**

Nginx écoute sur le **port 80/tcp**, sur `**0.0.0.0:80`** (toutes les interfaces IPv4) et `**[::]:80**` (toutes les interfaces IPv6). Les PID 1013 (processus maître) et 1021 (worker) partagent la socket. SSH écoute de même sur `0.0.0.0:22` et `[::]:22`, tandis que `systemd-resolved` n'écoute que sur la boucle locale (`127.0.0.53` et `127.0.0.54`, port 53) et n'est pas exposé.

**Q3. `ufw` comme groupe de sécurité : deux ressemblances et une différence.**

- *Ressemblance 1* : politique **« tout refuser en entrée par défaut »**, on n'ouvre que ce qui est nécessaire (22, 80).
- *Ressemblance 2* : règles exprimées par **port, protocole et source**, et pare-feu **à états** (la réponse à une connexion autorisée repasse automatiquement).
- *Différence* : `ufw` tourne **à l'intérieur du système invité** : un administrateur de la VM (root) peut le désactiver. Un groupe de sécurité est appliqué **à l'extérieur de la VM**, par l'infrastructure du fournisseur (hyperviseur / réseau virtuel), et reste en place même si l'invité est compromis.

**Q4. IaaS : qui gère quoi ?**

- *Géré par nous (client IaaS)* : le **système d'exploitation** (hostname, `machine-id`, clés SSH, configuration réseau Netplan, mises à jour), le **logiciel applicatif** (Nginx, la page web), la **sécurité côté invité** (`ufw`), les comptes et les **données**.
- *Fourni par VirtualBox (rôle du fournisseur)* : la **virtualisation** (hyperviseur), le **matériel virtuel** (CPU, RAM, disque, carte réseau) et le **réseau** (NAT `tp-cloud`, passerelle, DNS, redirections). Dans un vrai IaaS, le fournisseur gère en plus les **datacenters, l'énergie, le matériel physique et le réseau physique**.

---

## Partie 4 — Mesurer et nettoyer

### 4.1 Mesures vues de l'intérieur de `web`

**Commande**

```powershell
ssh -p 2222 etudiant@127.0.0.1 "free -h; df -h /; nproc"
```

**Réponse**

```text
               total        used        free      shared  buff/cache   available
Mem:           956Mi       217Mi       423Mi       1,1Mi       335Mi       738Mi
Swap:          2,0Gi          0B       2,0Gi
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  5,5G  5,2G  52% /
1
```

**Lecture.** `web` voit **956 Mi** de RAM (dont 217 Mi utilisés), un système de fichiers racine de **12 Go** (52 % utilisé), un swap de 2 Gi inutilisé et **1 vCPU**.

### 4.2 Mesures vues de l'hôte

**Commande**

```powershell
Get-ChildItem "$HOME\VirtualBox VMs" -Directory | ForEach-Object { "{0} : {1:N2} Go" -f $_.Name, ((Get-ChildItem $_.FullName -Recurse -File | Measure-Object Length -Sum).Sum / 1GB) }
```

**Réponse**

```text
client : 8,32 Go
ubuntu-s : 8,65 Go
web : 8,34 Go
```

**Lecture.** Chaque clone occupe **~8,3 Go** sur l'hôte, soit presque autant que le modèle (8,65 Go) : le clone est **complet**.. Mémoire vue par l'hôte (Gestionnaire des tâches → Détails, `VirtualBoxVM.exe`) *(valeur attendue)* : environ **350 Mo** pour la VM `web`, nettement moins que les 1024 Mo alloués.

### 4.3 Arrêt, redémarrage headless et nettoyage

**Commandes**

```powershell
VBoxManage controlvm web acpipowerbutton
VBoxManage controlvm client acpipowerbutton
VBoxManage list runningvms
VBoxManage startvm web --type headless
VBoxManage controlvm web poweroff
VBoxManage unregistervm web --delete
VBoxManage unregistervm client --delete
VBoxManage natnetwork remove --netname tp-cloud
VBoxManage list vms
```

**Réponse**

```text
(controlvm web acpipowerbutton)        aucune sortie
(controlvm client acpipowerbutton)     aucune sortie
(list runningvms)                      aucune sortie : plus aucune VM en marche   
Waiting for VM "web" to power on...
VM "web" has been successfully started.
(controlvm web poweroff)               0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
(unregistervm web --delete)            0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
(unregistervm client --delete)         0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
(natnetwork remove)                    aucune sortie
(list vms)
"ubuntu-s" {f83c4261-1d5f-4035-90b5-2b17a4ac09c2}
```

**Lecture.** `acpipowerbutton` éteint proprement les deux VM ; `web` redémarre en **headless** (sans fenêtre) ; `poweroff` la coupe avant suppression ; `unregistervm --delete` supprime les VM **et leurs fichiers** ; le réseau `tp-cloud` est retiré. Il ne reste que `**ubuntu-s`**, conservée pour les TP suivants.

### Réponses aux questions

**Q1. `web` voit-elle 1 Go de RAM ? Combien consomme-t-elle sur l'hôte ? Explication de l'écart.**

La VM voit **956 Mi** pour 1024 Mo configurés : le noyau et le matériel virtuel en réservent une partie. Sur l'hôte, `VirtualBoxVM.exe` consomme environ **350 Mo** *(valeur attendue)*. L'écart s'explique car VirtualBox n'alloue la mémoire invitée que **progressivement, selon l'usage réel** (ici ~217 Mi utilisés + cache), à laquelle s'ajoute la mémoire propre à l'hyperviseur (vidéo, périphériques émulés). La RAM « donnée » à une VM est un **plafond**, pas une consommation immédiate.

**Q2. Un disque de 25 Go : place réelle sur l'hôte ? Pourquoi ?**

Environ **8,3 Go** par clone (`web` : 8,34 Go, `client` : 8,32 Go). Le disque est un **VDI dynamique** : le fichier ne grossit qu'avec les données **réellement écrites** (système + paquets), jusqu'à 25 Go au maximum. Les 25 Go sont une taille **virtuelle** maximale, pas une réservation.

**Q3. Différence entre `acpipowerbutton`, `poweroff` et `unregistervm --delete` ?**

- `acpipowerbutton` : simule l'appui sur le bouton d'alimentation, le système s'arrête **proprement** (services arrêtés, disques synchronisés).
- `poweroff` : **coupure brutale**, équivalent à débrancher la prise (risque de corruption).
- `unregistervm --delete` : **supprime** la VM de VirtualBox **et ses fichiers** (disque, configuration) : opération **irréversible**, sans rapport avec l'arrêt.

**Q4. Dans le cloud, une VM arrêtée est-elle encore facturée ?**

Le **calcul (vCPU/RAM) n'est plus facturé**, mais les ressources qui persistent le restent : le **disque** (volume EBS, disque managé Azure) continue d'être facturé car les données sont conservées, et une **adresse IP publique réservée** (Elastic IP, IP publique Azure) peut être facturée, surtout si elle n'est plus associée à une instance en marche. Pour ne plus rien payer, il faut **supprimer** l'instance, ses disques et libérer l'IP.

---

## Conclusion — ce qu'il faut retenir du TP

Ce TP reproduit, en local, les trois gestes d'un client IaaS : **lancer une instance depuis une image**, la **placer dans un sous-réseau privé**, la **protéger par un pare-feu**.

**Clonage.** Cloner `ubuntu-s` en `web` a pris **28,75 s**, contre **23 min 48 s** pour installer le système : l'image est la base de l'élasticité du cloud. Un clone est une copie exacte : il faut changer **nom d'hôte, `machine-id` et clés SSH** (rôle de cloud-init dans le cloud), sinon une clé volée compromet tous les serveurs.

**Réseau.** Le NAT Network `tp-cloud` (`10.10.0.0/24`, sans DHCP) joue le rôle d'un **sous-réseau privé** : `web` (`10.10.0.10`) et `client` (`10.10.0.20`) se joignent (8,17 ms de moyenne, `ttl=64`), sortent sur Internet par la passerelle `10.10.0.1`, mais l'hôte ne les atteint que par les **redirections de port** (2222, 2223, 8080).

**Service et pare-feu.** Nginx écoute sur `0.0.0.0:80` et `[::]:80`. `ufw` (politique `deny` en entrée) bloque le port 80 : `curl` expire avec le **code 28**, sans message d'erreur. Autoriser OpenSSH **avant** d'activer `ufw` évite de se couper l'accès. Le pare-feu joue le rôle d'un **groupe de sécurité**, avec une différence : il vit **dans** la VM, alors qu'un groupe de sécurité est appliqué **par l'infrastructure**.

**Mesures et nettoyage.** `web` voit 956 Mi de RAM et 1 vCPU ; les disques VDI dynamiques occupent ~8,3 Go sur l'hôte pour 25 Go virtuels. `acpipowerbutton` arrête proprement, `poweroff` coupe brutalement, `unregistervm --delete` détruit. Tout a été supprimé, sauf `**ubuntu-s`**, gardée comme image pour les prochains TP.

En une phrase : **une instance cloud = une VM créée en quelques secondes depuis une image, personnalisée au démarrage, placée dans un sous-réseau privé, protégée par un pare-feu, et facturée tant que ses ressources existent**.
