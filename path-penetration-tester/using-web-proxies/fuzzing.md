# Fuzzing web

## Pourquoi

Le fuzzing web consiste a envoyer automatiquement un grand nombre de payloads a un parametre ou un endpoint pour decouvrir des repertoires, des fichiers, des identifiants valides ou des vulnerabilites. Burp Intruder et ZAP Fuzzer sont les deux fuzzers integres aux proxys web. Ils remplacent ou completent les outils CLI comme ffuf, gobuster ou wfuzz, avec l'avantage de travailler directement sur les requetes deja capturees.

## Comment ca marche

Le principe est toujours le meme :

1. Capturer une requete de reference dans l'historique du proxy
2. Definir les positions de payload (quels parametres seront remplaces)
3. Charger une wordlist
4. Lancer l'attaque et analyser les reponses

La difference entre les deux outils tient a la vitesse (ZAP n'a pas de limitation, Burp Community est throttle a 1 req/s) et aux options de configuration.

## En pratique

### Burp Intruder

**1. Envoyer la requete a Intruder :**

Depuis l'historique du proxy, clic droit sur la requete, puis `Send to Intruder` (ou `Ctrl+I`).

**2. Definir la position du payload :**

Dans l'onglet `Intruder > Positions`, selectionner le texte a remplacer et cliquer sur `Add §`. Burp encadre la position avec des marqueurs `§` :

```http
GET /§DIRECTORY§/ HTTP/1.1
Host: <IP_CIBLE>
```

**Types d'attaque :**

| Type | Usage |
|---|---|
| Sniper | Un seul payload, une position a la fois |
| Battering Ram | Le meme payload dans toutes les positions |
| Pitchfork | Un payload different par position (synchro) |
| Cluster Bomb | Toutes les combinaisons de plusieurs listes |

**3. Configurer la wordlist :**

Dans `Payloads`, choisir le type (generalement `Simple List`), puis `Load` pour charger un fichier. Par exemple `/usr/share/seclists/Discovery/Web-Content/common.txt`.

{% hint style="info" %}
Pour les tres grosses wordlists, utiliser le type `Runtime file` au lieu de `Simple List`. La liste est lue ligne par ligne au lieu d'etre chargee en memoire.
{% endhint %}

**4. Traitement des payloads (optionnel) :**

Dans `Payload Processing`, ajouter des regles :
- `Skip if matches regex` : exclure certains patterns (ex: `^\..*$` pour sauter les lignes commencant par `.`)
- `Add prefix/suffix` : ajouter une extension (ex: `.php`)

**5. Options :**

Dans `Settings`, configurer le `Grep - Match` pour filtrer les resultats :
- Activer le grep, clear la liste par defaut
- Ajouter `200 OK` comme pattern de correspondance
- Decocher `Exclude HTTP Headers` si le pattern est dans les headers

**6. Lancer l'attaque :**

Cliquer sur `Start Attack`. Trier les resultats par code de reponse ou par le flag `200 OK`.

{% hint style="warning" %}
En version Community, Intruder est limite a environ 1 requete par seconde. Pour une wordlist de 5000 entrees, compter plus d'une heure. Pour du fuzzing rapide sans licence, utiliser ZAP Fuzzer ou un outil CLI.
{% endhint %}

### ZAP Fuzzer

**1. Envoyer la requete au Fuzzer :**

Depuis l'historique, clic droit sur la requete, puis `Attack > Fuzz`.

**2. Definir la position :**

Selectionner le texte a remplacer dans la requete et cliquer sur `Add`. ZAP marque la position en vert.

**3. Configurer les payloads :**

Cliquer sur `Add` dans la fenetre de payloads :

| Type | Description |
|---|---|
| `File` | Charger une wordlist depuis un fichier |
| `File Fuzzers` | Wordlists integrees (dirbuster, fuzzdb si installe) |
| `Numberzz` | Sequences numeriques avec increment |

ZAP inclut des wordlists de dirbuster par defaut. Des listes supplementaires (fuzzdb) peuvent etre ajoutees via le Marketplace.

**4. Processeurs (optionnel) :**

Ajouter des transformations sur chaque payload : URL Encode, Base64, MD5 hash, prefixe/suffixe, ou un script personnalise.

**5. Options :**

Configurer le nombre de threads (`Concurrent Scanning Threads per Scan`). 20 threads est un bon compromis entre vitesse et stabilite.

Deux strategies de parcours :
- `Depth First` : toutes les valeurs sur une position avant de passer a la suivante
- `Breadth First` : chaque valeur sur toutes les positions avant de passer a la suivante

**6. Lancer et analyser :**

Cliquer sur `Start Fuzzer`. Trier par code HTTP (`Response`), taille de reponse, ou temps de reponse (utile pour les injections time-based).

## Pieges et galeres

- Burp Intruder en version gratuite est trop lent pour une utilisation en conditions reelles. Le reserver pour des tests cibles avec de petites wordlists.
- Un fuzzer qui genere trop de requetes simultanees peut declencher un WAF ou un rate limiter. Adapter le nombre de threads.
- Trier uniquement par code 200 peut faire manquer des resultats. Certaines pages retournent 301 (redirection) ou 403 (acces interdit mais existant). Regarder aussi la taille de reponse pour reperer les anomalies.
- Ne pas oublier d'activer l'encodage URL dans les payloads si la wordlist contient des caracteres speciaux.

## Retour terrain

Pour du fuzzing de repertoires ou de fichiers en volume, les outils CLI (ffuf, gobuster) restent plus rapides et plus pratiques. Les fuzzers integres aux proxys prennent tout leur sens quand on a besoin de fuzzer un parametre specifique dans une requete complexe (cookie encode, header particulier, body JSON) deja capturee dans le proxy. ZAP Fuzzer avec ses processeurs (encodage, hash) permet de construire des attaques sophistiquees sans script externe.

## Memo express

| Action | Burp Intruder | ZAP Fuzzer |
|---|---|---|
| Envoyer au fuzzer | `Ctrl+I` | Clic droit > `Attack > Fuzz` |
| Position du payload | Marqueurs `§` | Selection + `Add` |
| Charger une wordlist | `Payloads > Load` | `Add > File` ou `File Fuzzers` |
| Vitesse | Throttle (1 req/s en free) | Aucune limite |
| Threads | `Resource Pool` | `Options > Concurrent Threads` |
| Processeurs | `Payload Processing` | URL Encode, Base64, Hash, Script |

***
