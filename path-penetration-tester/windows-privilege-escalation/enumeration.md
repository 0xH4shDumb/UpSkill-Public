# Enumeration du systeme

Avant de chercher un vecteur d'escalade, il faut comprendre le systeme sur lequel on se trouve. L'enumeration initiale couvre l'OS, le reseau, les processus, les services, et les mecanismes de communication internes. C'est cette phase qui oriente toute la suite.

## Pourquoi

Un systeme Windows expose des dizaines de points de donnees exploitables : services en ecoute sur le loopback, processus avec des tokens privilegies, taches planifiees, variables d'environnement revelant des chemins non securises. Sans enumeration methodique, on passe a cote de vecteurs qui seraient visibles en quelques commandes.

## Comment ca marche

### Informations systeme et reseau

L'enumeration commence par les fondamentaux : version de l'OS, architecture, niveau de patch, interfaces reseau, table de routage, connexions actives.

| Donnee | Commande | Interet |
|---|---|---|
| Version OS / Build | `systeminfo` | Identifier les CVE applicables |
| Correctifs installes | `wmic qfe list brief` | Reperer les patchs manquants |
| Variables d'environnement | `set` | Chemins, proxys, variables custom |
| Interfaces reseau | `ipconfig /all` | Interfaces duales, DNS interne |
| Table de routage | `route print` | Reseaux accessibles, passerelles |
| Table ARP | `arp -a` | Hotes recemment contactes |
| Connexions actives | `netstat -ano` | Services en ecoute, connexions etablies |

### Utilisateurs et groupes

| Donnee | Commande |
|---|---|
| Utilisateur courant | `whoami` |
| Privileges du token | `whoami /priv` |
| Groupes | `whoami /groups` |
| Utilisateurs locaux | `net user` |
| Detail d'un utilisateur | `net user <username>` |
| Administrateurs locaux | `net localgroup administrators` |
| Informations domaine | `systeminfo \| findstr Domain` |

### Processus et services

Les processus en cours d'execution revelent les applications installees, les services vulnerables et les tokens exploitables. Un processus tournant en SYSTEM avec un port local accessible est une cible prioritaire.

| Commande | Usage |
|---|---|
| `tasklist /svc` | Liste des processus avec les services associes |
| `Get-Process` | Liste detaillee des processus (PowerShell) |
| `Get-Service` | Etat de tous les services |
| `sc qc <service>` | Configuration detaillee d'un service (compte, chemin, type de demarrage) |
| `wmic service list brief` | Vue synthetique des services |

### Connexions reseau et services locaux

Les services qui ecoutent sur `127.0.0.1` ou `::1` sans etre accessibles de l'exterieur sont souvent moins securises. Le raisonnement des developpeurs est que "ce n'est pas accessible depuis le reseau", ce qui conduit a des interfaces d'administration sans authentification.

| Service | Port typique | Exploitation |
|---|---|---|
| FileZilla Admin | 14147 | Extraction de mots de passe FTP, creation de partages |
| Splunk Universal Forwarder | 8089 | Deploiement d'applications (code execution as SYSTEM) |
| Erlang Port (RabbitMQ) | 25672 | Cookie par defaut (`rabbit`), acces au cluster |
| Bases de donnees locales | 3306, 5432 | Requetes avec des credentials par defaut |

{% hint style="warning" %}
Ne pas ignorer les ports en ecoute uniquement sur localhost. C'est souvent la que se trouvent les vecteurs d'escalade les plus directs, surtout sur des serveurs d'applications.
{% endhint %}

### Tokens d'acces et Named Pipes

Chaque processus Windows possede un token d'acces qui decrit le contexte de securite : identite de l'utilisateur, privileges, groupes. Comprendre les tokens est essentiel pour les attaques de type impersonation (SeImpersonate, Potato).

Les Named Pipes sont un mecanisme de communication inter-processus (IPC). Un pipe est un fichier en memoire qui disparait apres lecture. Les outils de C2 comme Cobalt Strike les utilisent pour la communication entre le beacon et les processus injectes.

| Commande | Usage |
|---|---|
| `pipelist.exe /accepteula` (Sysinternals) | Lister les named pipes |
| `gci \\.\pipe\` (PowerShell) | Enumerer les pipes |
| `accesschk.exe /accepteula \\.\pipe\<nom> -v` | Verifier les permissions sur un pipe |

## En pratique

### Enumeration initiale complete

```cmd
# - Informations systeme
systeminfo

# - Utilisateur courant, privileges et groupes
whoami /all

# - Utilisateurs et groupes locaux
net user
net localgroup administrators

# - Niveau de patch
wmic qfe list brief

# - Services en cours
tasklist /svc

# - Connexions reseau actives
netstat -ano

# - Variables d'environnement
set
```

### Recherche de services sur le loopback

```cmd
# - Identifier les ports en ecoute sur localhost uniquement
netstat -ano | findstr "127.0.0.1"
netstat -ano | findstr "LISTENING"

# - Mapper le PID au processus
tasklist /fi "PID eq <PID>"

# - Verifier le service associe
sc qc <nom_du_service>
```

### Enumeration reseau et domaine

```powershell
# - Interfaces et DNS
ipconfig /all

# - Table ARP (hotes recemment contactes)
arp -a

# - Partages reseau accessibles
net share
net view

# - Informations domaine
systeminfo | findstr /B /C:"Domain"
```

### Enumeration des Named Pipes

```powershell
# - Lister les pipes avec PowerShell
((Get-ChildItem \\.\pipe\).name) | Sort-Object

# - Verifier les permissions avec accesschk
accesschk.exe /accepteula \\.\pipe\lsass -v
accesschk.exe /accepteula -w \\.\pipe\* -v
```

{% hint style="info" %}
Un named pipe avec des permissions en ecriture pour les utilisateurs non privilegies est un vecteur potentiel. On peut l'utiliser pour injecter des commandes dans le contexte du processus proprietaire du pipe.
{% endhint %}

## Pieges et galeres

- **Netstat sans droits admin** : `netstat -ano` fonctionne en tant qu'utilisateur standard, mais certaines informations (noms de processus) ne sont pas affichees. Utiliser `tasklist` pour mapper les PID
- **Double interface reseau** : un serveur avec une interface interne et une externe peut donner acces a des segments reseau supplementaires. Ne pas oublier de verifier `route print`
- **Services caches** : certains services n'apparaissent pas dans `net start` ou `Get-Service` car ils sont marques comme masques dans le registre. Verifier directement dans `HKLM\SYSTEM\CurrentControlSet\Services`
- **Splunk Forwarder** : la configuration par defaut n'a pas d'authentification et permet le deploiement d'applications. Verifier le port 8089 systematiquement
- **Enumeration PowerShell bloquee** : si le Constrained Language Mode est actif, basculer sur `cmd.exe` et les outils Sysinternals

## Memo express

| Commande | Usage |
|---|---|
| `systeminfo` | Version OS, patches, domaine |
| `whoami /all` | Utilisateur, privileges, groupes |
| `net localgroup administrators` | Membres du groupe admin local |
| `tasklist /svc` | Processus avec services associes |
| `netstat -ano` | Connexions actives et ports en ecoute |
| `sc qc <service>` | Configuration d'un service |
| `ipconfig /all` | Interfaces reseau et DNS |
| `wmic qfe list brief` | Correctifs installes |
| `set` | Variables d'environnement |
| `route print` | Table de routage |

***
