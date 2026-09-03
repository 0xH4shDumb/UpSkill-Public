# SMB

Le protocole SMB (Server Message Block) est le pilier du partage de fichiers et de ressources dans les environnements Windows. Il permet d'accéder à des fichiers, des imprimantes et des services réseau depuis n'importe quel poste du réseau. En pentest, c'est l'un des premiers services à énumérer parce qu'il expose régulièrement des informations critiques.

## Pourquoi

SMB est omniprésent dans les réseaux d'entreprise, et sa configuration par défaut est souvent trop permissive. Les partages accessibles anonymement, les sessions nulles, les permissions mal définies sont des classiques. En l'énumérant correctement, on peut récupérer des listes d'utilisateurs, des fichiers de configuration, des sauvegardes, voire des credentials en clair.

## Comment ça marche

### Le protocole

SMB fonctionne en client-serveur sur TCP (ports 139 et 445). Le client envoie des requêtes pour accéder aux ressources partagées, et le serveur répond en fonction des permissions configurées (ACLs). L'authentification peut passer par NTLM ou Kerberos en environnement Active Directory.

### Versions du protocole

| Version | Systèmes | Points clés |
|---|---|---|
| CIFS | Windows NT 4.0 | Interface NetBIOS, ancien et vulnérable |
| SMB 1.0 | Windows 2000 | Connexion directe via TCP |
| SMB 2.0 | Vista / Server 2008 | Caching, signature améliorée |
| SMB 2.1 | Windows 7 / 2008 R2 | Verrouillages optimisés |
| SMB 3.0 | Windows 8 / 2012 | Multi-connexion, chiffrement de bout en bout |
| SMB 3.1.1 | Windows 10 / 2016 | Intégrité pré-authentification, AES-128 |

{% hint style="warning" %}
SMB 1.0 est encore actif sur certains réseaux pour des raisons de compatibilité. C'est une surface d'attaque majeure (EternalBlue, WannaCry). Sa présence mérite toujours une mention dans le rapport.
{% endhint %}

### Samba

Sur Linux, **Samba** implémente le protocole SMB et permet l'interopérabilité avec les environnements Windows. Depuis la version 4, Samba peut même servir de contrôleur de domaine Active Directory.

La configuration se fait via `/etc/samba/smb.conf` :

```ini
[global]
  workgroup = WORKGROUP
  server string = Samba Server
  map to guest = Bad User

[partage]
  path = /mnt/share
  browseable = yes
  read only = no
  guest ok = yes
```

### Paramètres sensibles

| Paramètre | Risque |
|---|---|
| `browseable = yes` | Permet de lister le contenu du partage |
| `guest ok = yes` | Autorise les connexions anonymes |
| `read only = no` | Permet l'écriture sur le partage |
| `create mask = 0777` | Droits maximaux sur les fichiers créés |
| `enable privileges = yes` | Honore les SID spéciaux |

## En pratique

### Lister les partages disponibles

```bash
# depuis Exegol - listing anonyme des partages
smbclient -N -L //<IP_CIBLE>
```

L'option `-N` tente une connexion sans mot de passe (session nulle). Si des partages apparaissent, c'est que le serveur accepte les connexions anonymes.

### Se connecter à un partage

```bash
# depuis Exegol - accès à un partage spécifique
smbclient //<IP_CIBLE>/partage
```

Une fois connecté :

```bash
smb: \> ls
smb: \> get fichier.txt
smb: \> !cat fichier.txt
```

### Enumération avec rpcclient

rpcclient permet d'interroger les services MS-RPC et d'extraire des informations sur les utilisateurs, les groupes et les partages.

```bash
# depuis Exegol - connexion nulle
rpcclient -U "" <IP_CIBLE>
```

| Commande | Usage |
|---|---|
| `srvinfo` | Informations sur le serveur |
| `enumdomusers` | Lister les utilisateurs du domaine |
| `queryuser <RID>` | Détails d'un utilisateur |
| `netshareenumall` | Lister tous les partages |

### Outils complémentaires

{% tabs %}
{% tab title="smbmap" %}
```bash
# depuis Exegol - cartographie des partages et permissions
smbmap -H <IP_CIBLE>
```

SMBMap affiche les partages avec leurs permissions (READ, WRITE) et permet de naviguer dans l'arborescence sans se connecter manuellement.
{% endtab %}

{% tab title="crackmapexec" %}
```bash
# depuis Exegol - enumération rapide des partages
crackmapexec smb <IP_CIBLE> --shares -u '' -p ''
```

CrackMapExec centralise l'énumération SMB, les tests de credentials et l'exécution de commandes dans un seul outil.
{% endtab %}

{% tab title="enum4linux-ng" %}
```bash
# depuis Exegol - énumération complète automatisée
enum4linux-ng -A <IP_CIBLE>
```

enum4linux-ng automatise la collecte d'informations : partages, utilisateurs, groupes, politique de mots de passe, informations de domaine.
{% endtab %}

{% tab title="nmap" %}
```bash
# depuis Exegol - scripts NSE SMB
sudo nmap -sV -sC -p139,445 <IP_CIBLE>
```

Les scripts NSE détectent automatiquement la version SMB, les partages accessibles et les vulnérabilités connues.
{% endtab %}
{% endtabs %}

## Pièges et galères

{% hint style="danger" %}
Un partage accessible en écriture peut servir à déposer un fichier malveillant (webshell, SCF, LNK) pour capturer des hash NTLMv2. C'est un vecteur d'attaque classique en interne qu'il faut documenter dans le rapport, même si l'exploitation n'est pas demandée.
{% endhint %}

- Une session nulle qui échoue ne signifie pas qu'il n'y a rien. Tester avec des credentials faibles (`guest:`, `anonymous:`) ou des comptes trouvés par ailleurs.
- `smbstatus` sur le serveur permet de voir les connexions actives. Utile si on a un accès limité au serveur et qu'on veut savoir qui d'autre est connecté.

## Retour terrain

SMB est souvent le service qui donne le premier foothold en interne. Les partages `NETLOGON`, `SYSVOL` et les partages départementaux contiennent régulièrement des scripts de logon avec des mots de passe en clair, des fichiers de configuration XML (GPP), ou des sauvegardes de bases de données. L'énumération des utilisateurs via rpcclient permet aussi de construire une wordlist ciblée pour du password spraying.

## Mémo express

| Commande | Usage |
|---|---|
| `smbclient -N -L //<IP>` | Lister les partages (session nulle) |
| `smbclient //<IP>/share` | Se connecter à un partage |
| `rpcclient -U "" <IP>` | Enumération RPC anonyme |
| `smbmap -H <IP>` | Cartographie des permissions |
| `enum4linux-ng -A <IP>` | Enumération automatisée complète |
| `crackmapexec smb <IP> --shares -u '' -p ''` | Enumération rapide |
| `sudo nmap -sV -sC -p139,445 <IP>` | Scan NSE SMB |

***
