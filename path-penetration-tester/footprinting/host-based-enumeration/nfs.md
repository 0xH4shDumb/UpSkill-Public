# NFS

Le Network File System (NFS) est le protocole de partage de fichiers natif des environnements Unix/Linux. Développé par Sun Microsystems, il permet de monter des systèmes de fichiers distants comme s'ils étaient locaux. En pentest, un partage NFS mal configuré peut donner un accès direct à des fichiers sensibles ou servir de tremplin pour une escalade de privilèges.

## Pourquoi

NFS est largement utilisé dans les infrastructures Linux pour centraliser les données, partager des répertoires home ou distribuer des fichiers de configuration. Le problème, c'est que NFS n'a pas de mécanisme d'authentification propre : il délègue le contrôle d'accès aux UID/GID Unix. Si un partage est exporté sans restriction, n'importe qui sur le réseau peut le monter et accéder aux fichiers avec les permissions de l'utilisateur local correspondant.

## Comment ça marche

### Le protocole

NFS repose sur ONC-RPC (Remote Procedure Call) et utilise le port 111 (portmapper/rpcbind) pour l'enregistrement des services, et le port 2049 pour le transfert de données en NFSv4. Les versions antérieures utilisent des ports dynamiques négociés via le portmapper.

### Versions

| Version | Caractéristiques |
|---|---|
| NFSv2 | UDP uniquement, ancien mais encore présent |
| NFSv3 | TCP/UDP, fichiers de taille variable, meilleurs rapports d'erreurs |
| NFSv4 | Authentification Kerberos, ACLs, port unique 2049, fonctionne à travers les pare-feux |
| NFSv4.1 | Multipathing (session trunking), accès parallèle distribué (pNFS) |

### Authentification et sécurité

NFS s'appuie sur les UID/GID pour contrôler l'accès. Le serveur suppose que le client présente un UID légitime, sans le vérifier. Si un attaquant crée localement un utilisateur avec le même UID que le propriétaire des fichiers sur le serveur, il obtient les mêmes droits.

{% hint style="danger" %}
NFS est conçu pour des réseaux de confiance. Sans Kerberos (NFSv4), il n'y a aucune vérification d'identité réelle. Le contrôle d'accès repose entièrement sur l'intégrité du client.
{% endhint %}

### Configuration

La configuration des exports se fait dans `/etc/exports`. Chaque ligne définit un répertoire exporté, les hôtes autorisés et les options d'accès.

```bash
/mnt/nfs  10.10.10.0/24(rw,sync,no_subtree_check)
```

| Option | Effet |
|---|---|
| `rw` | Lecture et écriture |
| `ro` | Lecture seule |
| `sync` | Écriture synchronisée (plus sûr, plus lent) |
| `async` | Écriture asynchrone (plus rapide, risque de perte) |
| `root_squash` | Remplace root par nobody (protection par défaut) |
| `no_root_squash` | Conserve les droits root du client sur le serveur |
| `insecure` | Accepte les connexions depuis des ports > 1024 |

{% hint style="warning" %}
`no_root_squash` est le paramètre le plus dangereux. Il permet à un client root de conserver ses privilèges sur le partage distant. Combiné à un export en `rw`, ça permet de déposer un binaire SUID ou de modifier des fichiers système.
{% endhint %}

## En pratique

### Scanner le service

```bash
# depuis Exegol - détection des services NFS
sudo nmap -p111,2049 -sV -sC <IP_CIBLE>
```

```bash
# depuis Exegol - scripts NSE spécifiques NFS
sudo nmap --script nfs* -p111,2049 <IP_CIBLE>
```

### Lister les exports

```bash
# depuis Exegol - voir les répertoires exportés
showmount -e <IP_CIBLE>
```

### Monter un partage

```bash
# depuis Exegol - monter le partage localement
mkdir target-NFS
sudo mount -t nfs <IP_CIBLE>:/chemin/export ./target-NFS/ -o nolock
```

### Explorer les fichiers

```bash
# - lister avec noms d'utilisateurs
ls -l target-NFS/

# - lister avec UID/GID numériques
ls -n target-NFS/
```

L'affichage numérique (`-n`) est important : il montre les UID/GID réels, ce qui permet de créer un utilisateur local correspondant pour accéder aux fichiers restreints.

### Démonter

```bash
sudo umount ./target-NFS
```

## Pièges et galères

- Si `showmount` retourne une erreur, le portmapper est peut-être filtré (port 111). Tenter un accès direct sur le port 2049 avec NFSv4.
- Les UID/GID affichés par `ls -n` sur un montage NFS correspondent aux utilisateurs du serveur. Si l'UID 1001 est `admin` sur le serveur mais `nobody` sur le client, créer un utilisateur local avec l'UID 1001 donne accès aux fichiers de `admin`.

{% hint style="success" %}
Pour exploiter un export `no_root_squash`, monter le partage en tant que root, copier un shell (`/bin/bash`), lui appliquer le bit SUID (`chmod +s`), puis l'exécuter depuis le serveur. C'est une escalade de privilèges classique via NFS.
{% endhint %}

## Retour terrain

NFS est un service qui passe souvent sous le radar parce que les scanners se concentrent sur TCP et que le portmapper (UDP 111) n'est pas toujours inclus dans les scans par défaut. Pourtant, les partages NFS mal configurés sont fréquents, surtout dans les environnements de développement ou les infrastructures legacy. Les répertoires home exportés en `rw` avec `no_root_squash` sont un chemin d'escalade fiable et reproductible.

## Mémo express

| Commande | Usage |
|---|---|
| `showmount -e <IP>` | Lister les exports NFS |
| `sudo mount -t nfs <IP>:/path ./mount -o nolock` | Monter un partage |
| `ls -n ./mount` | Afficher les UID/GID numériques |
| `sudo nmap --script nfs* -p111,2049 <IP>` | Enumération NSE |
| `sudo umount ./mount` | Démonter le partage |

***
