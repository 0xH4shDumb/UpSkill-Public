# Comprendre la sortie de SQLMap

SQLMap produit une quantité importante de messages pendant ses scans. Savoir les lire, c'est la différence entre un opérateur qui comprend ce qui se passe et quelqu'un qui lance des commandes à l'aveugle. Cette page décortique chaque message clé et couvre aussi les techniques de débogage quand les choses ne se passent pas comme prévu.

## Pourquoi

Quand SQLMap tourne, il affiche des dizaines de lignes d'information. Chaque message a une signification précise : détection d'un paramètre injectable, identification du SGBD, confirmation d'une vulnérabilité. Si on ne comprend pas ces messages, on passe à côté d'indices importants, on répond mal aux prompts interactifs, et on perd du temps à relancer des scans inutiles.

Le débogage est tout aussi critique. Un scan qui échoue silencieusement ou qui renvoie des faux négatifs peut venir d'un problème réseau, d'un WAF, ou simplement d'une mauvaise configuration de la requête. Les outils de diagnostic intégrés à SQLMap permettent de tracer exactement ce qui transite entre l'outil et la cible.

## Comment ça marche

### Messages de log essentiels

SQLMap suit une séquence logique pendant l'analyse d'un paramètre. Voici les messages clés dans l'ordre où ils apparaissent généralement.

#### 1. Stabilité du contenu de l'URL

```
[INFO] testing if the target URL content is stable
```

SQLMap compare plusieurs réponses identiques pour vérifier que la page ne change pas de manière aléatoire entre deux requêtes. Si le contenu est instable (contenu dynamique, compteurs, timestamps), l'outil aura du mal à distinguer une réponse normale d'une réponse modifiée par une injection.

{% hint style="warning" %}
Si ce test échoue, SQLMap propose d'utiliser `--string` ou `--regex` pour définir manuellement ce qui distingue une réponse « vraie » d'une réponse « fausse ». C'est souvent nécessaire sur des pages avec du contenu dynamique (publicités, horodatages).
{% endhint %}

#### 2. Détection de paramètre dynamique

```
[INFO] GET parameter 'id' appears to be dynamic
```

Le paramètre testé influence bien le contenu de la réponse. Si un paramètre est marqué comme non dynamique, il y a peu de chances qu'il soit injectable, car modifier sa valeur ne change rien côté serveur.

#### 3. Heuristique d'injectabilité

```
[INFO] heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')
```

SQLMap envoie des caractères spéciaux (guillemets, parenthèses) et observe si le serveur renvoie une erreur SGBD. Ce n'est qu'une heuristique : un résultat positif ne garantit pas l'injection, mais il oriente la suite des tests vers le SGBD détecté.

#### 4. Heuristique XSS

```
[INFO] heuristic (XSS) test shows that GET parameter 'id' might be vulnerable to cross-site scripting (XSS) attacks
```

En parallèle des tests SQLi, SQLMap vérifie si le paramètre reflète des caractères HTML sans encodage. Ce n'est pas le rôle principal de l'outil, mais c'est une information bonus utile pour le rapport de pentest.

#### 5. Détection du SGBD back-end

```
[INFO] it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n]
```

Quand SQLMap identifie le SGBD, il propose de limiter les payloads à ce moteur. Répondre « Y » accélère considérablement le scan en éliminant les tests pour PostgreSQL, Oracle, MSSQL, etc.

{% hint style="success" %}
En mode `--batch`, SQLMap répond automatiquement par le choix par défaut (généralement « Y »). C'est le comportement souhaité dans la plupart des cas.
{% endhint %}

#### 6. Prompt level/risk

```
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n]
```

SQLMap propose d'étendre les tests au-delà des valeurs de level et risk configurées. Accepter augmente la couverture mais rallonge le scan.

#### 7. Valeurs réflectives

```
[INFO] reflective value(s) found and filtering out
```

Le paramètre injecté est reflété dans la réponse HTTP. SQLMap filtre automatiquement ces reflets pour éviter les faux positifs dans la détection des résultats. C'est un comportement normal, pas une erreur.

#### 8. Confirmation d'injectabilité avec --string

```
[INFO] target URL appears to have ... injectable parameter 'id'
```

Si SQLMap a du mal à distinguer les réponses True/False (typique des injections boolean-blind), on peut utiliser `--string="contenu_specifique"` pour indiquer un texte qui apparait uniquement dans les réponses « vraies ». Cela améliore la fiabilité de la détection.

#### 9. Modèle statistique pour le time-based

```
[INFO] GET parameter 'id' appears to be 'AND/OR time-based blind' injectable
```

Les injections time-based reposent sur des délais mesurés dans les réponses. SQLMap utilise un modèle statistique pour déterminer si les variations de temps sont significatives ou dues au bruit réseau. Sur des connexions instables, ce type d'injection peut produire des faux positifs.

{% hint style="info" %}
Pour les injections time-based, SQLMap effectue des calculs statistiques sur les temps de réponse. Si la connexion au serveur est lente ou instable, le modèle peut se tromper. Augmenter le `--time-sec` (délai d'attente, défaut 5 secondes) améliore la fiabilité au prix de la durée du scan.
{% endhint %}

#### 10. Extension de la plage UNION

```
[INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
```

Quand SQLMap trouve une injection potentielle par une autre technique (boolean, time-based), il élargit automatiquement la plage de colonnes testées pour l'injection UNION. Le raisonnement : si le paramètre est injectable, autant tester UNION plus en profondeur pour obtenir une technique plus rapide d'extraction de données.

#### 11. Technique ORDER BY

```
[INFO] 'ORDER BY' technique appears to be usable
```

Pour déterminer le nombre de colonnes dans la requête UNION, SQLMap utilise d'abord `ORDER BY` (recherche binaire, plus rapide) avant de tomber sur l'approche par `UNION SELECT NULL,NULL,...`. Ce message confirme que la méthode rapide fonctionne.

#### 12. Vulnérabilité confirmée

```
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N]
```

Le paramètre est confirmé injectable. SQLMap demande s'il faut continuer à tester d'autres paramètres. En général, un seul point d'injection suffit pour l'exploitation.

#### Résumé des points d'injection

À la fin de la phase de détection, SQLMap affiche un bloc récapitulatif de tous les types d'injection trouvés :

```
sqlmap identified the following injection point(s) with a total of 46 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 5634=5634

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 3197 FROM (SELECT(SLEEP(5)))enSv)

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=1 UNION ALL SELECT NULL,CONCAT(0x71706a7671,...),NULL-- -
---
```

Ce résumé indique chaque type d'injection, le titre du payload qui a fonctionné et le payload exact utilisé.

#### Localisation des données

```
[INFO] fetched data logged to text files under '/home/user/.local/share/sqlmap/output/www.example.com'
```

Toutes les données extraites sont sauvegardées localement. Ce répertoire contient les dumps CSV, les logs de session et les informations de connexion réutilisables pour les scans suivants.

### Outils de débogage

Quand un scan échoue ou produit des résultats inattendus, SQLMap offre plusieurs mécanismes pour comprendre ce qui se passe.

#### Affichage des erreurs SGBD

L'option `--parse-errors` force SQLMap à afficher les messages d'erreur renvoyés par le SGBD :

```bash
sqlmap -u "http://<IP_CIBLE>/vuln.php?id=1" --parse-errors --batch
```

```
[WARNING] parsed DBMS error message: 'SQLSTATE[42000]: Syntax error or
access violation: 1064 You have an error in your SQL syntax; check the
manual that corresponds to your MySQL server version for the right
syntax to use near '))"',),)((' at line 1'
```

Ces erreurs aident à comprendre comment la requête SQL est construite côté serveur et pourquoi certains payloads échouent.

#### Enregistrement du trafic

L'option `-t` sauvegarde l'intégralité du trafic HTTP dans un fichier :

```bash
sqlmap -u "http://<IP_CIBLE>/vuln.php?id=1" --batch -t /tmp/traffic.txt
```

Le fichier contient chaque requête envoyée et chaque réponse reçue, avec les en-têtes complets. C'est la méthode la plus exhaustive pour analyser un comportement inattendu.

#### Niveaux de verbosité

L'option `-v` contrôle le niveau de détail affiché dans la console :

| Niveau | Contenu affiché |
|--------|----------------|
| `-v 0` | Résultats uniquement |
| `-v 1` | Informations de base (défaut) |
| `-v 2` | Messages de debug |
| `-v 3` | Payloads injectés |
| `-v 4` | Requêtes HTTP complètes |
| `-v 5` | En-têtes de réponse HTTP |
| `-v 6` | Contenu complet des réponses HTTP |

```bash
# - Afficher tout le trafic HTTP en temps réel
sqlmap -u "http://<IP_CIBLE>/vuln.php?id=1" -v 6 --batch
```

{% hint style="info" %}
Le niveau `-v 3` est souvent le meilleur compromis en pratique : on voit exactement quels payloads sont envoyés sans être noyé par le contenu HTML des réponses.
{% endhint %}

#### Routage via un proxy

L'option `--proxy` redirige tout le trafic de SQLMap à travers un proxy d'interception (Burp Suite, ZAP) :

```bash
sqlmap -u "http://<IP_CIBLE>/vuln.php?id=1" --proxy="http://127.0.0.1:8080" --batch
```

Chaque requête apparait dans l'onglet HTTP History du proxy. On peut alors les inspecter, les modifier et les rejouer manuellement. C'est particulièrement utile pour :

- Vérifier que les payloads sont correctement encodés
- Comparer les réponses entre requêtes normales et injectées
- Identifier un WAF ou un filtre applicatif qui bloque certains payloads

## En pratique

### Scénario type : interpréter un scan complet

Depuis Exegol, on lance un scan basique et on observe la sortie :

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --batch
```

La séquence attendue :

1. Test de stabilité du contenu (quelques requêtes identiques)
2. Vérification que le paramètre est dynamique
3. Heuristique rapide (caractères spéciaux)
4. Si le SGBD est détecté, proposition de limiter les tests
5. Tests des différentes techniques (boolean, time-based, UNION, error-based, stacked)
6. Résumé des injections trouvées

### Scénario type : déboguer un scan qui échoue

```bash
# - Étape 1 : activer les erreurs SGBD et la verbosité
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --parse-errors -v 3 --batch

# - Étape 2 : si rien de visible, capturer le trafic
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" -t /tmp/sqlmap-traffic.txt --batch

# - Étape 3 : router via Burp pour inspection manuelle
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --proxy="http://127.0.0.1:8080" --batch
```

## Pièges et galères

**Contenu instable pris pour un faux négatif.** Si la page contient des éléments dynamiques (timestamp, compteur de visites, token anti-CSRF dans le body), SQLMap peut conclure que le paramètre n'est pas injectable alors qu'il l'est. Solution : utiliser `--string="texte_fixe"` pour ancrer la comparaison sur un élément stable de la page.

**Time-based sur connexion lente.** Les injections temporelles sont sensibles à la latence réseau. Sur un VPN avec du jitter, SQLMap peut détecter des faux positifs (délais réseau interprétés comme des `SLEEP()`) ou des faux négatifs (délais d'injection noyés dans la latence). Augmenter `--time-sec=10` et relancer.

**Proxy qui casse les réponses.** Si Burp est configuré pour modifier les réponses (remplacement automatique, décompression), les résultats de SQLMap peuvent être faussés. Vérifier que le proxy est en mode passif, sans règles de match/replace actives.

**Verbosité excessive.** Lancer `-v 6` sur un scan avec des centaines de payloads produit un volume de sortie ingérable. Préférer `-v 3` pour le diagnostic ciblé, ou `-t fichier.txt` pour une analyse post-mortem.

## Retour terrain

En pentest, la lecture de la sortie SQLMap fait gagner un temps considérable. Quand on voit que l'heuristique détecte MySQL mais que les tests boolean-blind échouent tous, c'est souvent le signe d'un WAF ou d'un filtre applicatif. Plutôt que d'augmenter le level/risk (ce qui multiplie les requêtes), on passe d'abord par le proxy pour comprendre ce qui est bloqué.

L'enregistrement du trafic (`-t`) est aussi précieux pour la documentation. Lors de la rédaction du rapport, on peut retrouver exactement quelles requêtes ont confirmé la vulnérabilité, avec les réponses complètes du serveur.

Un réflexe utile : toujours lancer le premier scan avec `--parse-errors`. Même quand tout fonctionne, les messages d'erreur SGBD donnent des informations sur la version exacte du moteur et la structure de la requête SQL sous-jacente.

## Mémo express

| Besoin | Option |
|--------|--------|
| Voir les erreurs SGBD | `--parse-errors` |
| Sauvegarder tout le trafic | `-t /chemin/fichier.txt` |
| Payloads envoyés dans la console | `-v 3` |
| Trafic HTTP complet en console | `-v 6` |
| Router via Burp/ZAP | `--proxy="http://127.0.0.1:8080"` |
| Ancrer la détection boolean-blind | `--string="texte_a_chercher"` |
| Allonger le délai time-based | `--time-sec=10` |
| Mode silencieux (résultats seuls) | `-v 0` |

***
