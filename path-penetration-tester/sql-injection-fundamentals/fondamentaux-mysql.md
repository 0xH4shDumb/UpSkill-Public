# Fondamentaux MySQL

## Pourquoi

Avant d'exploiter une injection SQL, il faut comprendre comment fonctionne SQL. Ce chapitre couvre les bases de MySQL/MariaDB : connexion, creation de bases et de tables, requetes CRUD, filtres et operateurs. Ces connaissances sont indispensables pour construire des payloads d'injection efficaces.

## En pratique

### Connexion au serveur MySQL

```bash
# Connexion locale (mot de passe demande interactivement)
mysql -u root -p

# Connexion distante
mysql -u root -h <IP_CIBLE> -P 3306 -p
```

{% hint style="info" %}
Le port par defaut de MySQL est 3306. L'option `-P` (majuscule) specifie le port, `-p` (minuscule) le mot de passe.
{% endhint %}

### Gestion des bases de donnees

```sql
-- Lister les bases
SHOW DATABASES;

-- Creer une base
CREATE DATABASE users;

-- Utiliser une base
USE users;
```

### Creation de tables

```sql
CREATE TABLE logins (
    id INT NOT NULL AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL,
    date_of_joining DATETIME DEFAULT NOW(),
    PRIMARY KEY (id)
);
```

Proprietes courantes :

| Propriete | Effet |
|---|---|
| `AUTO_INCREMENT` | L'id s'incremente automatiquement |
| `NOT NULL` | Champ obligatoire |
| `UNIQUE` | Valeur unique dans la table |
| `DEFAULT` | Valeur par defaut si non specifiee |
| `PRIMARY KEY` | Identifiant unique de chaque enregistrement |

```sql
-- Voir les tables d'une base
SHOW TABLES;

-- Voir la structure d'une table
DESCRIBE logins;
```

### Requetes CRUD

#### INSERT (creation)

```sql
-- Inserer un enregistrement complet
INSERT INTO logins VALUES(1, 'admin', 'p@ssw0rd', '2024-01-01');

-- Inserer en specifiant les colonnes (les champs avec DEFAULT sont auto-remplis)
INSERT INTO logins(username, password) VALUES('admin', 'p@ssw0rd');

-- Inserer plusieurs enregistrements
INSERT INTO logins(username, password) VALUES ('john', 'john123!'), ('tom', 'tom123!');
```

#### SELECT (lecture)

```sql
-- Tout selectionner
SELECT * FROM logins;

-- Colonnes specifiques
SELECT username, password FROM logins;
```

#### UPDATE (mise a jour)

```sql
UPDATE logins SET password = 'new_password' WHERE id > 1;
```

{% hint style="warning" %}
Toujours utiliser `WHERE` avec `UPDATE`. Sans clause de filtre, tous les enregistrements sont modifies.
{% endhint %}

#### DROP (suppression de table)

```sql
DROP TABLE logins;
```

#### ALTER (modification de structure)

```sql
-- Ajouter une colonne
ALTER TABLE logins ADD email VARCHAR(100);

-- Renommer une colonne
ALTER TABLE logins RENAME COLUMN email TO mail;

-- Changer le type d'une colonne
ALTER TABLE logins MODIFY mail TEXT;

-- Supprimer une colonne
ALTER TABLE logins DROP mail;
```

### Filtrage et tri des resultats

#### WHERE (filtre conditionnel)

```sql
SELECT * FROM logins WHERE id > 1;
SELECT * FROM logins WHERE username = 'admin';
```

{% hint style="info" %}
Les chaines et les dates doivent etre entourees de guillemets simples (`'`) ou doubles (`"`). Les nombres s'utilisent directement.
{% endhint %}

#### LIKE (recherche par pattern)

```sql
-- Tous les usernames commencant par 'admin'
SELECT * FROM logins WHERE username LIKE 'admin%';

-- Usernames de exactement 3 caracteres
SELECT * FROM logins WHERE username LIKE '___';
```

- `%` remplace zero ou plusieurs caracteres
- `_` remplace exactement un caractere

#### ORDER BY (tri)

```sql
-- Tri ascendant (defaut)
SELECT * FROM logins ORDER BY password;

-- Tri descendant
SELECT * FROM logins ORDER BY password DESC;

-- Tri sur plusieurs colonnes
SELECT * FROM logins ORDER BY password DESC, id ASC;
```

#### LIMIT (nombre de resultats)

```sql
-- 2 premiers resultats
SELECT * FROM logins LIMIT 2;

-- 2 resultats a partir du 2e (offset 1)
SELECT * FROM logins LIMIT 1, 2;
```

### Operateurs logiques

| Operateur | Symbole | Effet |
|---|---|---|
| `AND` | `&&` | Vrai si les deux conditions sont vraies |
| `OR` | `\|\|` | Vrai si au moins une condition est vraie |
| `NOT` | `!` | Inverse la valeur booleenne |

```sql
-- Combiner des conditions
SELECT * FROM logins WHERE username != 'john' AND id > 1;

-- Precedence : AND est evalue avant OR
SELECT * FROM logins WHERE username = 'admin' OR id > 1 AND password = 'test';
-- Equivalent a : username = 'admin' OR (id > 1 AND password = 'test')
```

Ordre de precedence (du plus prioritaire au moins prioritaire) :

1. Arithmetique (`/`, `*`, `%`, `+`, `-`)
2. Comparaison (`=`, `>`, `<`, `>=`, `<=`, `!=`, `LIKE`)
3. `NOT` (`!`)
4. `AND` (`&&`)
5. `OR` (`||`)

## Pieges et galeres

- Les mots-cles SQL ne sont pas sensibles a la casse (`SELECT` = `select`), mais les noms de bases et de tables le sont sur les systemes Linux.
- `LIKE` est insensible a la casse par defaut dans MySQL. Pour une recherche sensible a la casse, utiliser `LIKE BINARY`.
- `LIMIT 1, 2` signifie "sauter 1, prendre 2", pas "prendre de 1 a 2". L'offset commence a 0.
- La precedence des operateurs est cruciale pour les injections SQL. `AND` etant evalue avant `OR`, la requete `WHERE a OR b AND c` est interpretee comme `WHERE a OR (b AND c)`.

## Retour terrain

Ces commandes SQL sont la base de toute injection. Comprendre comment `SELECT`, `WHERE`, `ORDER BY`, `UNION` et les operateurs logiques fonctionnent permet de construire des payloads precis. En engagement, on ne lance pas sqlmap en aveugle : on comprend d'abord la structure de la requete pour injecter intelligemment.

## Memo express

| Commande | Effet |
|---|---|
| `SHOW DATABASES;` | Lister les bases |
| `USE db;` | Changer de base |
| `SHOW TABLES;` | Lister les tables |
| `DESCRIBE table;` | Structure d'une table |
| `SELECT * FROM t WHERE c = 'v';` | Lecture filtree |
| `INSERT INTO t(c) VALUES('v');` | Insertion |
| `UPDATE t SET c = 'v' WHERE id = 1;` | Mise a jour |
| `ORDER BY col DESC;` | Tri |
| `LIMIT offset, count;` | Pagination |
| `LIKE 'pattern%';` | Recherche par motif |

***
