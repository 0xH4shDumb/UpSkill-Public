# Introduction aux injections de commandes

L'injection de commandes OS fait partie des vulnérabilités les plus critiques qu'on peut rencontrer en audit web. Elle permet d'exécuter des commandes système directement sur le serveur hébergeant l'application, avec les privilèges du processus web. En une seule requête bien construite, on passe de la lecture d'une page web à l'exécution de code arbitraire sur la machine cible. Ce type de faille figure régulièrement dans le top 3 du OWASP Top 10, et ce n'est pas un hasard : elle est à la fois courante et dévastatrice.

## Pourquoi

Les applications web modernes ont parfois besoin d'interagir avec le système d'exploitation sous-jacent. Installer un plugin, convertir un fichier, vérifier la disponibilité d'un hôte : autant de cas où un développeur peut être tenté d'appeler une commande système depuis le code applicatif. Si l'entrée utilisateur atterrit dans cette commande sans être correctement validée, un attaquant peut détourner l'appel pour exécuter ses propres commandes. Le résultat : lecture de fichiers sensibles, accès aux identifiants de base de données, pivotement vers le réseau interne, voire prise de contrôle complète du serveur.

## Comment ça marche

### Les injections en général

Le principe d'une injection est toujours le même : une entrée utilisateur est interprétée comme faisant partie d'une requête ou d'un code exécuté, au lieu d'être traitée comme une simple donnée. Selon le contexte d'exécution, on parle de types d'injection différents :

| Type d'injection | Description |
|---|---|
| OS Command Injection | L'entrée utilisateur est intégrée dans une commande système |
| Code Injection | L'entrée est passée à une fonction qui évalue du code (eval, exec) |
| SQL Injection | L'entrée est insérée dans une requête SQL |
| XSS / HTML Injection | L'entrée est affichée telle quelle dans une page web |
| LDAP Injection | L'entrée est utilisée dans une requête LDAP |
| NoSQL Injection | L'entrée est insérée dans une requête NoSQL (MongoDB, etc.) |

D'autres variantes existent (XPath, IMAP, ORM, Header Injection), mais le mécanisme fondamental reste identique : l'absence de séparation entre données et instructions.

### Fonctions vulnérables par langage

Chaque langage web dispose de fonctions permettant d'exécuter des commandes système. Quand une entrée utilisateur y transite sans sanitization, la porte est ouverte.

{% tabs %}
{% tab title="PHP" %}
Les fonctions à surveiller en PHP :

- `exec()` : exécute une commande, retourne la dernière ligne de sortie
- `system()` : exécute une commande et affiche la sortie directement
- `shell_exec()` : exécute via le shell, retourne la sortie complète
- `passthru()` : exécute et transmet la sortie brute (utile pour les données binaires)
- `popen()` : ouvre un processus et retourne un pointeur de fichier

```php
// - Exemple vulnérable : le paramètre filename est concaténé sans validation
<?php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}
?>
```

Un attaquant peut fournir `test; whoami` comme valeur de `filename`, ce qui transforme la commande en `touch /tmp/test; whoami.pdf`. Le point-virgule sépare les deux commandes, et `whoami` s'exécute sur le serveur.
{% endtab %}

{% tab title="NodeJS" %}
En NodeJS, les fonctions du module `child_process` sont les vecteurs principaux :

- `child_process.exec()` : exécute via le shell (vulnérable aux injections)
- `child_process.spawn()` : lance un processus (moins vulnérable si utilisé correctement)
- `child_process.execSync()` : version synchrone de exec

```javascript
// - Exemple vulnérable : l'interpolation de chaîne injecte directement l'entrée
app.get("/createfile", function(req, res) {
    child_process.exec(`touch /tmp/${req.query.filename}.txt`);
});
```

Le même principe s'applique : si `filename` contient un opérateur d'injection suivi d'une commande, celle-ci sera exécutée.
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
L'injection de commandes ne se limite pas aux applications web. Tout binaire ou client lourd qui passe une entrée utilisateur non validée à une fonction d'exécution système est potentiellement vulnérable au même type d'attaque.
{% endhint %}

### Opérateurs d'injection

Pour injecter une commande supplémentaire, on utilise un opérateur qui permet de chaîner ou de séparer les commandes dans le shell. La syntaxe est la même peu importe le langage de l'application web ou le framework utilisé.

| Opérateur | Caractère | URL-encodé | Comportement |
|---|---|---|---|
| Point-virgule | `;` | `%3b` | Exécute les deux commandes séquentiellement |
| Nouvelle ligne | `\n` | `%0a` | Exécute les deux commandes séquentiellement |
| Background | `&` | `%26` | Exécute les deux (la 2e souvent affichée en premier) |
| Pipe | `\|` | `%7c` | Exécute les deux (seule la sortie de la 2e est affichée) |
| AND | `&&` | `%26%26` | Exécute la 2e seulement si la 1ère réussit |
| OR | `\|\|` | `%7c%7c` | Exécute la 2e seulement si la 1ère échoue |
| Sub-shell (backtick) | `` ` ` `` | `%60%60` | Exécute les deux (Linux uniquement) |
| Sub-shell | `$()` | `%24%28%29` | Exécute les deux (Linux uniquement) |

{% hint style="info" %}
Le point-virgule ne fonctionne pas avec CMD sous Windows, mais il est reconnu par PowerShell. Tous les autres opérateurs fonctionnent sur les deux systèmes, à l'exception des sub-shells qui sont spécifiques à Linux/bash.
{% endhint %}

Le payload se construit toujours de la même façon : on fournit l'entrée attendue (par exemple une adresse IP), on ajoute un opérateur d'injection, puis on écrit la commande à exécuter.

```bash
# - Exemples de payloads d'injection
127.0.0.1; whoami           # - Point-virgule : exécute les deux
127.0.0.1 && cat /etc/passwd # - AND : exécute la 2e si ping réussit
|| id                        # - OR : exécute la 2e car la 1ère échoue (pas d'IP)
127.0.0.1 | id              # - Pipe : seule la sortie de id est affichée
```

### Opérateurs d'injection par type de vulnérabilité

Chaque famille de vulnérabilité par injection utilise ses propres opérateurs de rupture. Cette table sert de référence rapide pour identifier les caractères à tester selon le contexte :

| Type d'injection | Opérateurs courants |
|---|---|
| SQL Injection | `'` `,` `;` `--` `/* */` |
| Command Injection | `;` `&&` `\|` `&` |
| LDAP Injection | `*` `(` `)` `&` `\|` |
| XPath Injection | `'` `or` `and` `not` `substring` |
| Code Injection | `'` `;` `--` `/* */` `$()` `${}` |
| Directory Traversal | `../` `..\` `%00` |
| Header Injection | `\n` `\r\n` `\t` `%0d` `%0a` |

## En pratique

### Détecter une injection de commandes

Depuis un conteneur Exegol, la démarche de détection est méthodique :

```bash
# - Étape 1 : identifier un paramètre qui pourrait interagir avec le système
# - Chercher les fonctionnalités de type ping, DNS lookup, traceroute, conversion de fichiers

# - Étape 2 : tester un opérateur simple
curl -s -X POST "http://<IP_CIBLE>:<PORT>/check.php" \
     -d "ip=127.0.0.1;id" \
     --proxy http://127.0.0.1:8080

# - Étape 3 : si la validation front-end bloque, intercepter avec Burp
# - Envoyer une requête valide, l'intercepter, modifier le paramètre dans Repeater

# - Étape 4 : tester les différents opérateurs si le premier est filtré
curl -s -X POST "http://<IP_CIBLE>:<PORT>/check.php" -d "ip=127.0.0.1%0aid"
curl -s -X POST "http://<IP_CIBLE>:<PORT>/check.php" -d "ip=127.0.0.1%26%26id"
curl -s -X POST "http://<IP_CIBLE>:<PORT>/check.php" -d "ip=127.0.0.1%7cid"
```

{% hint style="warning" %}
La validation côté client (JavaScript) n'est jamais une mesure de sécurité. Elle se contourne en envoyant la requête directement au serveur via un proxy comme Burp, sans passer par le navigateur. Toute validation doit être dupliquée côté serveur.
{% endhint %}

### Choix de l'opérateur selon le contexte

Le choix de l'opérateur dépend de ce qu'on cherche à obtenir :

- **Point-virgule (`;`)** : le plus direct, exécute inconditionnellement. Premier choix en test initial.
- **AND (`&&`)** : utile quand on veut que la commande légitime s'exécute aussi (discrétion, logs normaux).
- **OR (`||`)** : utile quand on ne peut pas fournir d'entrée valide, ou quand on veut supprimer la sortie de la commande légitime.
- **Pipe (`|`)** : n'affiche que la sortie de notre commande. Pratique pour une lecture propre du résultat.
- **Nouvelle ligne (`%0a`)** : souvent oublié dans les blacklists. Bon candidat quand les opérateurs classiques sont filtrés.

## Pièges et galères

- **Validation front-end** : si le navigateur refuse l'entrée (format IP uniquement, par exemple), ce n'est pas un indicateur de sécurité back-end. Toujours tester en envoyant la requête directement.
- **Sortie non visible** : certaines injections fonctionnent mais la sortie n'est pas renvoyée dans la réponse HTTP. On parle d'injection aveugle (blind). Dans ce cas, utiliser des techniques basées sur le temps (`sleep 5`) ou l'exfiltration DNS/HTTP.
- **URL-encoding** : les caractères spéciaux doivent être URL-encodés dans les requêtes HTTP. Un `&` non encodé sera interprété comme séparateur de paramètres, pas comme opérateur d'injection. Penser à encoder avec Burp (Ctrl+U) ou manuellement.
- **Confusion entre les opérateurs** : `&&` exécute la 2e commande seulement si la 1ère réussit. Si on fournit une IP invalide avec `&&`, notre commande ne s'exécutera pas. Utiliser `||` ou `;` dans ce cas.

## Retour terrain

En audit, les injections de commandes se trouvent principalement dans les fonctionnalités qui interagissent avec le système : ping/traceroute, conversion de fichiers (PDF, images), recherche de fichiers, gestion de certificats, ou tout formulaire qui déclenche un traitement côté serveur. Les applications développées en interne sont plus souvent vulnérables que celles basées sur des frameworks matures, car ces derniers découragent l'appel direct à des commandes système.

La première étape est toujours d'identifier les points d'entrée susceptibles de finir dans un appel système. Ensuite, on teste les opérateurs un par un en commençant par le plus simple (`;`). Si des filtres sont en place, on adapte la stratégie (les techniques de contournement sont couvertes dans les pages suivantes).

L'impact d'une injection de commandes confirmée est presque toujours critique. Depuis l'exécution de code, on accède aux secrets applicatifs, aux identifiants de base de données, et on peut pivoter vers d'autres systèmes du réseau. C'est une vulnérabilité qui justifie un arrêt immédiat pour correction dans un rapport de pentest.

## Mémo express

| Élément | Détail |
|---|---|
| Risque | Critique (OWASP Top 3) |
| Fonctions PHP vulnérables | `exec`, `system`, `shell_exec`, `passthru`, `popen` |
| Fonctions NodeJS vulnérables | `child_process.exec`, `child_process.spawn` |
| Opérateur le plus fiable | `;` (Linux), `&&` (cross-platform) |
| Opérateur souvent non filtré | `\n` (`%0a`) |
| Premier test | `127.0.0.1; id` ou `127.0.0.1%0aid` |
| Bypass front-end | Envoyer la requête directement via Burp/cURL |
| Encoder les payloads | Ctrl+U dans Burp, ou `%xx` manuellement |

***
