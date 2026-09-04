# Introduction aux injections SQL

## Pourquoi

La plupart des applications web modernes stockent leurs donnees dans une base de donnees relationnelle (MySQL, PostgreSQL, MSSQL, etc.). Quand l'application construit ses requetes SQL a partir d'entrees utilisateur sans les securiser, un attaquant peut injecter du code SQL arbitraire. C'est l'injection SQL (SQLi), l'une des vulnerabilites les plus critiques et les plus repandues dans le web.

L'impact d'une SQLi va de la fuite de donnees sensibles (identifiants, donnees personnelles, informations bancaires) au contournement d'authentification, en passant par la lecture/ecriture de fichiers sur le serveur et, dans les cas les plus graves, l'execution de commandes systeme.

## Comment ca marche

### Architecture web classique

Une application web fonctionne typiquement en trois couches :

| Couche | Role | Exemple |
|---|---|---|
| Client (Tier I) | Interface utilisateur | Navigateur, formulaire de login |
| Application (Tier II) | Logique metier, construction des requetes | PHP, Python, Java |
| Base de donnees (Tier III) | Stockage et restitution des donnees | MySQL, PostgreSQL, MSSQL |

Quand un utilisateur soumet un formulaire, l'application construit une requete SQL et l'envoie a la base de donnees. Si l'entree utilisateur est inseree directement dans la requete sans sanitisation, l'injection est possible.

### Bases de donnees relationnelles vs NoSQL

| Type | Structure | Langage | Exemples |
|---|---|---|---|
| Relationnelle (RDBMS) | Tables, lignes, colonnes, cles primaires, schemas | SQL | MySQL, PostgreSQL, MSSQL, Oracle |
| NoSQL | Documents, cles-valeurs, graphes | Varie (JSON, etc.) | MongoDB, Redis, Cassandra |

Les injections SQL concernent les bases relationnelles. Les bases NoSQL ont leurs propres vulnerabilites d'injection (NoSQL injection), qui ne sont pas couvertes dans ce module.

{% hint style="info" %}
Un schema definit les relations entre les tables d'une base relationnelle. Par exemple, une table `users` liee a une table `posts` via la colonne `user_id`.
{% endhint %}

### Principe de l'injection

L'injection se produit quand l'application interprete une entree utilisateur comme du code SQL au lieu d'une simple chaine de caracteres. L'attaquant echappe les limites de l'entree prevue (generalement avec `'` ou `"`) et injecte du SQL qui sera execute par le SGBD.

Exemple en PHP vulnerable :

```php
$searchInput = $_POST['findUser'];
$query = "SELECT * FROM logins WHERE username LIKE '%$searchInput'";
$result = $conn->query($query);
```

Si l'utilisateur entre `admin`, la requete devient `...LIKE '%admin'`. Mais s'il entre `' OR '1'='1`, la requete devient `...LIKE '%' OR '1'='1'` et retourne tous les enregistrements.

## Types d'injections SQL

| Type | Sous-type | Principe |
|---|---|---|
| **In-band** | Union-based | Le resultat de la requete injectee est affiche directement dans la page |
| **In-band** | Error-based | Les messages d'erreur SQL revelent des informations |
| **Blind** | Boolean-based | La page repond differemment selon que la condition injectee est vraie ou fausse |
| **Blind** | Time-based | La reponse est retardee (via `SLEEP()`) si la condition est vraie |
| **Out-of-band** | DNS/HTTP | Le resultat est exfiltre vers un serveur externe |

Ce module se concentre sur les injections **Union-based**, les plus directes a comprendre et exploiter.

## Pieges et galeres

- L'injection SQL ne se limite pas aux formulaires de login. Tout parametre utilisateur qui finit dans une requete SQL est potentiellement vulnerable : champs de recherche, filtres, parametres d'URL, headers HTTP, cookies.
- Une page qui affiche une erreur SQL quand on injecte un `'` est un signe fort de vulnerabilite. Mais l'absence d'erreur ne signifie pas l'absence de vulnerabilite (blind SQLi).
- Les injections contre MSSQL et PostgreSQL supportent les requetes empilees (`;`), ce qui n'est pas le cas de MySQL. Chaque SGBD a ses specificites.
- Les injections NoSQL existent aussi (MongoDB, etc.) mais utilisent des techniques completement differentes.

## Retour terrain

En engagement, l'injection SQL est souvent l'une des premieres vulnerabilites testees sur les applications web. La decouverte se fait en injectant des caracteres speciaux (`'`, `"`, `#`, `;`, `)`) dans tous les points d'entree et en observant les reponses. Un outil comme Burp Suite facilite ce travail en interceptant et modifiant les requetes.

L'impact d'une SQLi confirmee est presque toujours critique : acces a la base complete, contournement d'authentification, et potentiellement execution de code. C'est pourquoi elle figure systematiquement dans le top 10 OWASP.

## Memo express

| Concept | Detail |
|---|---|
| SQLi | Injection de code SQL via des entrees utilisateur non sanitisees |
| RDBMS | MySQL, PostgreSQL, MSSQL, Oracle |
| Caracteres de test | `'`, `"`, `#`, `;`, `)` |
| In-band | Union-based (resultat visible), Error-based (erreur visible) |
| Blind | Boolean-based (reponse differente), Time-based (delai) |
| Out-of-band | Exfiltration via DNS/HTTP |
| Impact | Fuite de donnees, bypass auth, lecture/ecriture fichiers, RCE |

***
