# Introduction a l'elevation de privileges Linux

L'elevation de privileges est une etape incontournable de tout pentest sur un systeme Linux. Apres avoir obtenu un acces initial, souvent avec un compte a faibles privileges, l'objectif est de progresser jusqu'a `root` pour prendre le controle total de la machine. Cette page pose les bases de la methodologie et des vecteurs d'attaque que l'on rencontre en pratique.

## Pourquoi

Lors d'un engagement, l'acces initial donne rarement un shell root. On atterrit generalement sur un compte de service, un utilisateur standard ou un shell web limite. Sans elevation de privileges, on ne peut pas acceder aux fichiers sensibles, pivoter vers d'autres machines du reseau ou atteindre les objectifs du pentest. Comprendre les mecanismes d'escalade permet aussi de formuler des recommandations de durcissement pertinentes dans le rapport.

## Comment ca marche

### Les vecteurs d'attaque principaux

L'elevation de privileges Linux repose sur l'exploitation de mauvaises configurations, de permissions trop permissives ou de vulnerabilites logicielles. On peut classer les vecteurs en grandes familles.

| Famille | Exemples |
|---|---|
| **Permissions** | Binaires SUID/SGID, capabilities mal attribuees |
| **Sudo** | Regles sudoers trop permissives, NOPASSWD |
| **Groupes privilegies** | Appartenance a `docker`, `lxd`, `disk`, `adm` |
| **Environnement** | Manipulation de PATH, abus de wildcards, shells restreints |
| **Services** | Cron jobs modifiables, services vulnerables, logrotate |
| **Conteneurs** | Evasion Docker, escalade Kubernetes, LXC/LXD |
| **Mecanismes internes** | Kernel exploits, bibliotheques partagees, hijacking Python |
| **CVE recentes** | Sudo (CVE-2021-3156), Polkit (CVE-2021-4034), Dirty Pipe |

### Methodologie d'enumeration

Avant de tenter quoi que ce soit, il faut enumerer le systeme de maniere methodique. L'enumeration couvre plusieurs axes.

| Axe | Ce qu'on cherche |
|---|---|
| **Systeme** | Version du noyau, distribution, architecture |
| **Utilisateurs** | Comptes actifs, groupes, historique de connexions |
| **Reseau** | Interfaces, routes, ports en ecoute, connexions actives |
| **Processus** | Services qui tournent en root, taches planifiees |
| **Fichiers** | Permissions SUID/SGID, fichiers world-writable, configurations sensibles |
| **Logiciels** | Versions de sudo, screen, polkit, paquets installes |
| **Defenses** | SELinux, AppArmor, iptables, exec-shield |

{% hint style="info" %}
L'enumeration est la phase la plus importante. Un oubli a ce stade peut faire rater un vecteur d'escalade trivial. Prendre le temps de tout passer en revue systematiquement avant de se lancer dans l'exploitation.
{% endhint %}

### Outils d'enumeration automatisee

Plusieurs outils permettent d'automatiser cette phase et de gagner du temps.

| Outil | Description |
|---|---|
| **LinPEAS** | Script Bash/Python qui enumere des centaines de vecteurs potentiels |
| **LinEnum** | Script Bash classique d'enumeration |
| **linux-exploit-suggester** | Suggere des kernel exploits en fonction de la version du noyau |
| **pspy** | Surveille les processus sans privileges root (scrute `/proc`) |

{% hint style="warning" %}
Les outils automatises generent beaucoup de bruit. Sur un engagement red team, preferer les commandes manuelles pour limiter la detection. Les scripts d'enumeration restent precieux pour ne rien oublier en CTF ou en lab.
{% endhint %}

## En pratique

### Premiers reflexes apres un acces initial

```bash
# - Qui suis-je et quels sont mes groupes ?
id
whoami

# - Quel est le systeme ?
uname -a
cat /etc/os-release

# - Version du noyau
uname -r

# - Sudo : quelles commandes puis-je executer ?
sudo -l

# - Binaires SUID sur le systeme
find / -perm -4000 -type f 2>/dev/null

# - Taches cron
cat /etc/crontab
ls -la /etc/cron.d/
crontab -l

# - Processus en cours (recherche de services root)
ps aux | grep root

# - Connexions reseau actives
ss -tulnp
```

### Enumeration rapide avec LinPEAS

```bash
# - Telecharger et executer LinPEAS depuis Exegol
curl -sL https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | bash

# - Ou le transferer et l'executer sur la cible
./linpeas.sh -a 2>&1 | tee linpeas_output.txt
```

LinPEAS colore les resultats par severite : le rouge et le jaune meritent une attention immediate.

## Pieges et galeres

- **Enumeration incomplete** : l'erreur la plus frequente est de se precipiter sur le premier vecteur trouve. Toujours terminer l'enumeration avant de commencer l'exploitation
- **Kernel exploits en production** : les exploits noyau peuvent provoquer un kernel panic et planter le systeme. A eviter sur des environnements de production sauf en dernier recours et avec l'accord explicite du client
- **Faux positifs des outils** : LinPEAS et les autres scripts remontent des centaines de lignes. Beaucoup ne sont pas exploitables. Il faut valider manuellement chaque piste
- **Defenses actives** : SELinux et AppArmor peuvent bloquer des exploitations meme si la misconfiguration existe. Verifier leur statut avec `getenforce` et `aa-status`

## Memo express

| Commande | Usage |
|---|---|
| `id` | Utilisateur courant et groupes |
| `uname -a` | Informations systeme et noyau |
| `sudo -l` | Privileges sudo disponibles |
| `find / -perm -4000 2>/dev/null` | Binaires SUID |
| `cat /etc/crontab` | Taches cron systeme |
| `ps aux \| grep root` | Processus root |
| `ss -tulnp` | Ports en ecoute |
| `getcap -r / 2>/dev/null` | Capabilities sur les binaires |
| `cat /etc/os-release` | Distribution et version |
| `./linpeas.sh` | Enumeration automatisee |

***
