# Enumeration du systeme

L'enumeration est le socle de toute elevation de privileges. Sans une collecte d'informations rigoureuse, on passe a cote de vecteurs d'attaque parfois triviaux. Cette page couvre les techniques d'enumeration manuelle : environnement systeme, services internes et recherche de credentials.

## Pourquoi

Un systeme Linux expose des dizaines de vecteurs potentiels d'escalade. L'enumeration permet de cartographier la surface d'attaque : version du noyau, services en cours, taches planifiees, fichiers de configuration accessibles, credentials oublies dans des scripts. Chaque information collectee peut reveler une faille exploitable.

## Comment ca marche

### Enumeration de l'environnement

L'objectif est de comprendre le systeme sur lequel on se trouve : distribution, version du noyau, architecture, utilisateurs, defenses actives.

| Information | Pourquoi c'est utile |
|---|---|
| **Version du noyau** | Determine si des kernel exploits sont applicables |
| **Distribution et version** | Oriente la recherche d'exploits specifiques |
| **Architecture** | 32 bits vs 64 bits, important pour la compilation d'exploits |
| **Utilisateurs et groupes** | Identifie les groupes privilegies (docker, lxd, sudo) |
| **Defenses** | SELinux, AppArmor, exec-shield peuvent bloquer l'exploitation |
| **Systemes de fichiers montes** | Partages NFS, montages avec `nosuid`/`noexec` |

### Services et mecanismes internes

Les services qui tournent sur la machine revelent des opportunites. Un processus root avec un fichier de configuration modifiable, un cron job qui execute un script world-writable, une version de sudo vulnérable.

| Element | Ou chercher |
|---|---|
| **Interfaces reseau** | `ip a`, `/etc/hosts`, routes |
| **Connexions actives** | `ss -tulnp`, `netstat -antp` |
| **Historique de connexions** | `lastlog`, `w`, `who` |
| **Bash history** | `~/.bash_history`, `~/.zsh_history` |
| **Cron jobs** | `/etc/crontab`, `/etc/cron.d/`, `crontab -l` |
| **Paquets installes** | `dpkg -l`, `rpm -qa` |
| **Version de sudo** | `sudo -V` (certaines versions sont vulnerables) |
| **Fichiers de config** | `/etc/*.conf`, applications web |

### Recherche de credentials

Les credentials sont souvent stockees en clair dans des endroits previsibles. Les repertoires web, les fichiers de configuration, les emails locaux et les cles SSH sont les cibles prioritaires.

| Emplacement | Ce qu'on cherche |
|---|---|
| `/var/www/html/` | `wp-config.php`, `.env`, fichiers de configuration applicatifs |
| `/var/spool/mail/` | Emails contenant des mots de passe ou des informations sensibles |
| `~/.ssh/` | Cles privees SSH (`id_rsa`, `id_ed25519`) |
| `/tmp/`, `/opt/`, `/home/*/` | Scripts contenant des credentials en dur |
| Historiques shell | Commandes passees avec des mots de passe en argument |

## En pratique

### Enumeration de l'environnement

```bash
# - Informations systeme
uname -a
cat /etc/os-release
lsb_release -a 2>/dev/null

# - Architecture
arch

# - Utilisateurs avec un shell valide
grep -v "nologin\|false" /etc/passwd

# - Groupes de l'utilisateur courant
id

# - Tous les utilisateurs et leurs groupes
cat /etc/passwd | cut -d: -f1,3,4 | sort -t: -k3 -n

# - Dernieres connexions
lastlog | grep -v "Never"
w
who

# - Defenses actives
cat /etc/apparmor.d/* 2>/dev/null
getenforce 2>/dev/null
aa-status 2>/dev/null

# - Systemes de fichiers montes
df -h
cat /etc/fstab | grep -v "^#"
mount | grep -E "nosuid|noexec"
```

### Enumeration des services et mecanismes internes

```bash
# - Interfaces reseau et routes
ip a
ip route
cat /etc/hosts

# - Ports en ecoute et connexions actives
ss -tulnp
netstat -antp 2>/dev/null

# - Processus en cours d'execution
ps aux
ps aux | grep root

# - Historique bash (credentials dans les commandes)
cat ~/.bash_history 2>/dev/null
cat /home/*/.bash_history 2>/dev/null

# - Taches cron
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ 2>/dev/null
for user in $(cut -d: -f1 /etc/passwd); do crontab -l -u $user 2>/dev/null; done

# - Paquets installes (Debian/Ubuntu)
dpkg -l 2>/dev/null | head -30

# - Version de sudo
sudo -V | head -n1

# - Fichiers de configuration modifiables
find /etc -writable -type f 2>/dev/null
```

### Recherche de credentials

```bash
# - Fichiers de configuration web
find /var/www -name "*.conf" -o -name "*.config" -o -name "*.php" -o -name ".env" 2>/dev/null
cat /var/www/html/wp-config.php 2>/dev/null | grep -i "db_\|pass\|user"

# - Emails locaux
ls -la /var/spool/mail/ 2>/dev/null
cat /var/mail/* 2>/dev/null

# - Cles SSH accessibles
find / -name "id_rsa" -o -name "id_ed25519" -o -name "authorized_keys" 2>/dev/null
ls -la /home/*/.ssh/ 2>/dev/null

# - Recherche de mots-cles dans les fichiers accessibles
grep -rni "passw\|secret\|token\|api_key" /var/www/ /opt/ /home/ 2>/dev/null | head -30

# - Fichiers recemment modifies (potentiellement interessants)
find / -mmin -10 -type f 2>/dev/null | grep -v "/proc\|/sys"
```

{% hint style="success" %}
Penser a chercher aussi dans les repertoires non standards : `/opt/`, `/srv/`, `/backup/`, `/mnt/`. Les administrateurs y placent souvent des scripts de maintenance contenant des credentials.
{% endhint %}

### Surveillance des processus avec pspy

Quand on ne peut pas lire les crontabs des autres utilisateurs, `pspy` permet de voir les processus qui se lancent en temps reel.

```bash
# - Telecharger et executer pspy
./pspy64 -pf -i 1000

# Surveiller les lignes avec UID=0 (processus root)
# Reperer les scripts executes periodiquement
```

{% hint style="info" %}
`pspy` fonctionne en scrutant `/proc` sans necessite de privileges root. C'est l'outil de reference pour decouvrir les cron jobs caches et les processus ephemeres.
{% endhint %}

## Pieges et galeres

- **Bash history efface** : certains administrateurs configurent `HISTSIZE=0` ou redirigent l'historique vers `/dev/null`. Ne pas compter uniquement sur l'historique
- **Processus ephemeres** : les cron jobs qui s'executent et se terminent rapidement n'apparaissent pas dans un `ps aux` classique. Utiliser `pspy` pour les detecter
- **Permissions sur /proc** : des configurations `hidepid=2` sur `/proc` empechent de voir les processus des autres utilisateurs. `pspy` peut etre affecte
- **Fichiers pièges** : attention aux fichiers `.bash_history` contenant des commandes piégées (honeypots). Verifier le contexte avant d'exploiter des credentials trouvees
- **grep recursif trop large** : un `grep -r` sur tout le systeme peut prendre un temps considerable et generer du bruit. Cibler les repertoires pertinents

## Memo express

| Commande | Usage |
|---|---|
| `uname -a` | Version du noyau et architecture |
| `cat /etc/os-release` | Distribution et version |
| `id` | Utilisateur courant et groupes |
| `lastlog \| grep -v Never` | Dernieres connexions |
| `ss -tulnp` | Ports en ecoute |
| `ps aux \| grep root` | Processus root |
| `cat ~/.bash_history` | Historique des commandes |
| `cat /etc/crontab` | Taches cron systeme |
| `find /var/www -name ".env"` | Fichiers de config web |
| `find / -name "id_rsa"` | Cles SSH privees |
| `./pspy64 -pf -i 1000` | Surveillance des processus |
| `sudo -V \| head -n1` | Version de sudo |

***
