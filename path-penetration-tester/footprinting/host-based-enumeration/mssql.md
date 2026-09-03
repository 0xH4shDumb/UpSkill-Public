# MSSQL

Microsoft SQL Server (MSSQL) est le SGBDR propriétaire de Microsoft, largement déployé dans les environnements d'entreprise Windows. Etroitement intégré à Active Directory et au framework .NET, il représente une cible de choix en pentest interne : un accès à l'instance MSSQL permet souvent de pivoter vers le système d'exploitation sous-jacent.

## Pourquoi

MSSQL n'est pas un simple service de base de données. Avec `xp_cmdshell`, il peut exécuter des commandes système. Avec les linked servers, il peut se connecter à d'autres instances SQL. Avec l'authentification Windows intégrée, un compte de domaine compromis peut donner accès à la base sans mot de passe SQL supplémentaire. L'impact d'un accès MSSQL dépasse largement le périmètre de la base de données elle-même.

## Comment ça marche

### Architecture

MSSQL écoute par défaut sur le port TCP 1433. Chaque installation peut contenir plusieurs instances, identifiées par un nom (par défaut `MSSQLSERVER`). Le SQL Server Browser Service (UDP 1434) permet aux clients de découvrir les instances disponibles.

### Modes d'authentification

| Mode | Description |
|---|---|
| Windows Authentication | Utilise les comptes AD ou SAM local. Transparent pour l'utilisateur. |
| SQL Authentication | Login et mot de passe gérés par SQL Server. Le compte `sa` est le superadmin. |
| Mixed Mode | Les deux modes acceptés. Configuration la plus courante. |

{% hint style="warning" %}
En mode Mixed, le compte `sa` est actif par défaut. Un mot de passe faible sur `sa` donne un contrôle total sur l'instance, y compris la possibilité d'exécuter des commandes système.
{% endhint %}

### Bases système

| Base | Rôle |
|---|---|
| `master` | Configuration de l'instance, logins, linked servers |
| `model` | Template pour les nouvelles bases |
| `msdb` | Jobs du SQL Server Agent, alertes, historique |
| `tempdb` | Objets temporaires, recréée à chaque redémarrage |
| `resource` | Objets système en lecture seule (invisible par défaut) |

### Clients

- **SSMS** (SQL Server Management Studio) : client graphique officiel
- **mssqlclient.py** (Impacket) : client en ligne de commande, très utilisé en pentest
- **sqlcmd** : client CLI natif Microsoft
- **PowerShell** : module `SqlServer`

## En pratique

### Scan du service

```bash
# depuis Exegol - détection et énumération MSSQL
sudo nmap -p1433 -sV --script ms-sql-info,ms-sql-ntlm-info,ms-sql-empty-password <IP_CIBLE>
```

Le script `ms-sql-ntlm-info` est particulièrement utile : il extrait le nom NetBIOS de la machine, le nom de domaine, et la version de l'OS via le challenge NTLM, sans authentification.

### Détection via Metasploit

```bash
use auxiliary/scanner/mssql/mssql_ping
set RHOSTS <IP_CIBLE>
run
```

Retourne le nom d'instance, la version, le port et la présence de named pipes.

### Connexion avec Impacket

```bash
# depuis Exegol - connexion avec authentification Windows
mssqlclient.py user@<IP_CIBLE> -windows-auth
```

```bash
# depuis Exegol - connexion avec authentification SQL
mssqlclient.py sa@<IP_CIBLE>
```

### Enumération T-SQL

```sql
-- lister les bases de données
SELECT name FROM sys.databases;

-- lister les logins
SELECT name, type_desc FROM sys.server_principals;

-- vérifier les privilèges du compte courant
SELECT IS_SRVROLEMEMBER('sysadmin');

-- lister les tables d'une base
USE nom_base;
SELECT * FROM sys.tables;
```

### Exécution de commandes

```sql
-- activer xp_cmdshell (nécessite sysadmin)
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- exécuter une commande système
EXEC xp_cmdshell 'whoami';
```

{% hint style="danger" %}
`xp_cmdshell` avec un compte `sysadmin` donne une exécution de commandes au niveau du compte de service SQL Server, qui tourne souvent avec des privilèges élevés (NT SERVICE\MSSQLSERVER, voire SYSTEM sur les installations legacy).
{% endhint %}

## Pièges et galères

- La connexion n'est pas chiffrée par défaut. Un attaquant positionné sur le réseau peut intercepter les credentials SQL en clair. Les certificats auto-signés utilisés pour le chiffrement sont falsifiables pour du man-in-the-middle.
- Les named pipes (`\\server\pipe\sql\query`) sont un vecteur d'attaque supplémentaire : ils peuvent être utilisés pour des relais NTLM.
- Le SQL Server Browser (UDP 1434) est souvent oublié dans les règles de pare-feu et permet de découvrir des instances cachées sur des ports non standard.

## Retour terrain

MSSQL est le service qui transforme un accès applicatif en accès système. Un compte `sa` avec un mot de passe faible, c'est un `xp_cmdshell` et potentiellement un shell SYSTEM. Même sans `sysadmin`, les linked servers permettent parfois de pivoter vers d'autres instances avec des privilèges plus élevés (`EXECUTE AS LOGIN`). Le script `mssqlclient.py` d'Impacket est l'outil de référence pour l'exploitation, avec ses commandes intégrées (`enum_db`, `enum_logins`, `enable_xp_cmdshell`).

## Mémo express

| Commande | Usage |
|---|---|
| `mssqlclient.py user@<IP> -windows-auth` | Connexion Windows Auth |
| `mssqlclient.py sa@<IP>` | Connexion SQL Auth |
| `SELECT name FROM sys.databases;` | Lister les bases |
| `EXEC xp_cmdshell 'whoami';` | Exécution de commande |
| `sudo nmap -p1433 --script ms-sql-info,ms-sql-ntlm-info <IP>` | Scan NSE |

***
