# MySQL

MySQL est l'un des systèmes de gestion de bases de données relationnelles les plus déployés au monde. Open-source, maintenu par Oracle, il est au coeur de la plupart des stacks web (LAMP, LEMP). En pentest, un service MySQL exposé est une cible prioritaire : il contient les données applicatives, les comptes utilisateurs, et parfois les credentials d'autres services.

## Pourquoi

MySQL est rarement exposé sur Internet de manière intentionnelle, mais en interne il est omniprésent. Les applications web, les CMS (WordPress, Joomla), les systèmes de ticketing, les outils de monitoring stockent tous leurs données dans MySQL. Un accès à la base, même en lecture seule, permet d'extraire des hash de mots de passe, des données personnelles, des configurations applicatives.

## Comment ça marche

### Architecture

MySQL fonctionne en client-serveur. Le serveur écoute par défaut sur le port TCP 3306. Les clients envoient des requêtes SQL, le serveur les exécute et retourne les résultats. L'authentification se fait par login/mot de passe, avec des privilèges granulaires par base, table, et opération.

### Bases système

| Base | Rôle |
|---|---|
| `mysql` | Comptes utilisateurs, privilèges, configuration |
| `information_schema` | Métadonnées sur toutes les bases (tables, colonnes, types) |
| `performance_schema` | Statistiques d'exécution et de performance |
| `sys` | Vues simplifiées sur performance_schema |

### Configuration

Le fichier principal est `/etc/mysql/mysql.conf.d/mysqld.cnf` :

```ini
[mysqld]
user = mysql
port = 3306
datadir = /var/lib/mysql
bind-address = 0.0.0.0
```

### Paramètres dangereux

| Paramètre | Risque |
|---|---|
| `bind-address = 0.0.0.0` | Le serveur écoute sur toutes les interfaces |
| `skip-grant-tables` | Désactive complètement l'authentification |
| `secure_file_priv = ""` | Permet la lecture/écriture de fichiers arbitraires |
| Compte root sans mot de passe | Accès total sans authentification |

{% hint style="danger" %}
`secure_file_priv` vide permet d'utiliser `LOAD_FILE()` pour lire des fichiers du système et `INTO OUTFILE` pour écrire. C'est un vecteur d'exécution de code si le serveur web et MySQL tournent sur la même machine.
{% endhint %}

## En pratique

### Scan du service

```bash
# depuis Exegol - détection et scripts NSE MySQL
sudo nmap -sV -sC -p3306 --script mysql* <IP_CIBLE>
```

Les scripts NSE détectent la version, testent les mots de passe vides (`mysql-empty-password`), et listent les bases accessibles.

### Connexion

```bash
# depuis Exegol - connexion avec credentials
mysql -u root -p -h <IP_CIBLE>
```

### Enumération de base

```sql
-- lister les bases de données
SHOW DATABASES;

-- sélectionner une base
USE nom_base;

-- lister les tables
SHOW TABLES;

-- voir la structure d'une table
SHOW COLUMNS FROM nom_table;

-- extraire les données
SELECT * FROM nom_table;
```

### Extraction des comptes

```sql
-- utilisateurs et hash de mots de passe
SELECT user, authentication_string FROM mysql.user;
```

{% hint style="info" %}
Les hash MySQL sont crackables avec hashcat (`-m 300` pour les anciens hash MySQL, `-m 7401` pour MySQL 5.7+/SHA256). Un hash extrait de `mysql.user` combiné à une wordlist ciblée donne souvent des résultats.
{% endhint %}

### MariaDB

MariaDB est un fork open-source de MySQL, 100% compatible au niveau des commandes et de la connectivité. Les outils d'énumération fonctionnent de manière identique sur les deux.

## Pièges et galères

- Les scripts NSE MySQL génèrent parfois des faux positifs (bases listées alors qu'elles ne sont pas accessibles). Toujours vérifier manuellement avec une connexion directe.
- Un accès en lecture sur `information_schema` permet de cartographier l'intégralité de la structure de la base sans avoir les droits sur les données elles-mêmes. C'est suffisant pour planifier l'extraction une fois qu'on obtient des droits supérieurs.

## Retour terrain

MySQL est le service qui donne souvent le deuxième foothold, après SMB ou FTP. Les credentials trouvés dans des fichiers de configuration (`wp-config.php`, `.env`, `config.php`) permettent de se connecter à la base et d'extraire les données utilisateurs. La réutilisation de mots de passe entre MySQL et les comptes système est courante : un mot de passe root MySQL qui fonctionne aussi en SSH, ça arrive plus souvent qu'on ne le pense.

## Mémo express

| Commande | Usage |
|---|---|
| `mysql -u root -p -h <IP>` | Connexion au serveur |
| `SHOW DATABASES;` | Lister les bases |
| `SELECT user, authentication_string FROM mysql.user;` | Extraire les comptes |
| `sudo nmap -sV -sC -p3306 --script mysql* <IP>` | Scan NSE MySQL |

***
