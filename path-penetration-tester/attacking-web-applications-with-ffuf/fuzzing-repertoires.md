# Fuzzing de repertoires et fichiers

## Pourquoi

La premiere etape de la reconnaissance web est d'identifier les repertoires et fichiers accessibles sur le serveur. Les pages d'administration, les backups, les fichiers de configuration mal proteges : tout ca peut donner un point d'entree ou de l'information sensible. Le fuzzing de repertoires est le moyen le plus rapide de les decouvrir.

## Comment ca marche

ffuf remplace le marqueur `FUZZ` dans l'URL par chaque entree de la wordlist. Le serveur repond avec un code HTTP : 200 (existe), 301 (redirection), 403 (interdit mais existe), ou 404 (n'existe pas). ffuf filtre automatiquement les 404 par defaut.

## En pratique

### Fuzzing de repertoires

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://<IP_CIBLE>/FUZZ
```

Un repertoire existant retourne generalement un code 301 (redirection vers `/repertoire/`).

### Identification des extensions

Avant de fuzzer les fichiers, identifier les extensions utilisees par le serveur. Le fichier `index` existe sur quasiment tous les sites :

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://<IP_CIBLE>/blog/indexFUZZ
```

Un `.php` en 200 confirme que le serveur utilise PHP. Adapter l'extension pour les scans suivants.

### Fuzzing de fichiers

Une fois l'extension identifiee, fuzzer les noms de fichiers dans les repertoires decouverts :

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://<IP_CIBLE>/blog/FUZZ.php
```

### Scan recursif

ffuf peut scanner automatiquement les sous-repertoires decouverts avec l'option `-recursion` :

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://<IP_CIBLE>/FUZZ \
     -recursion -recursion-depth 1 \
     -e .php \
     -v
```

| Option | Effet |
|---|---|
| `-recursion` | Active le scan recursif |
| `-recursion-depth 1` | Limite la profondeur (sous-repertoires directs uniquement) |
| `-e .php` | Ajoute l'extension `.php` a chaque entree |
| `-v` | Affiche les URLs completes (indispensable en mode recursif) |

{% hint style="warning" %}
Le scan recursif multiplie le nombre de requetes par le nombre de repertoires trouves. Avec `-recursion-depth 2` et 10 repertoires au premier niveau, le scan peut devenir tres long. Commencer par `-recursion-depth 1` et affiner ensuite.
{% endhint %}

## Pieges et galeres

- Un repertoire qui retourne une page vide (200, taille 0) existe mais n'a pas de fichier `index`. Ca ne veut pas dire qu'il est vide : il peut contenir d'autres fichiers accessibles.
- Le code 403 est un signal fort : la ressource existe mais l'acces est bloque. A explorer avec d'autres techniques (bypass, credentials).
- Les wordlists generiques couvrent environ 90% des cas. Les pages avec des noms personnalises ou aleatoires ne seront pas decouvertes par fuzzing.
- La wordlist `directory-list-2.3-small` contient ~87 000 entrees. La version `medium` en contient ~220 000. Utiliser la petite pour un premier passage rapide.

## Retour terrain

Le scan recursif de ffuf est un gain de temps considerable par rapport au scan manuel repertoire par repertoire. En engagement, on commence par un scan non recursif pour identifier les repertoires principaux, puis on lance un scan recursif cible sur les repertoires les plus interessants (admin, api, backup, uploads). Documenter chaque repertoire et fichier decouvert dans les notes de reconnaissance.

## Memo express

| Objectif | Commande |
|---|---|
| Repertoires | `ffuf -w list:FUZZ -u http://<IP>/FUZZ` |
| Extensions | `ffuf -w web-extensions.txt:FUZZ -u http://<IP>/indexFUZZ` |
| Fichiers PHP | `ffuf -w list:FUZZ -u http://<IP>/dir/FUZZ.php` |
| Recursif | `ffuf -w list:FUZZ -u http://<IP>/FUZZ -recursion -recursion-depth 1 -e .php -v` |

***
