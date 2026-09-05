# Prévention des injections de commandes

## Pourquoi

Toutes les techniques d'exploitation et de contournement vues dans ce module partagent un point commun : elles exploitent le passage d'une entrée utilisateur dans une fonction d'exécution système. Comprendre les mécanismes de prévention permet à la fois de formuler des recommandations précises dans un rapport de pentest et d'identifier les failles dans les protections en place. La défense en profondeur reste la seule approche efficace, car chaque couche prise isolément peut être contournée.

## Comment ça marche

### Supprimer les fonctions d'exécution système

La mesure la plus radicale et la plus efficace consiste à ne jamais utiliser de fonctions qui exécutent des commandes OS avec des données provenant de l'utilisateur. La plupart des langages offrent des alternatives natives qui accomplissent la même tâche sans passer par un shell.

{% tabs %}
{% tab title="PHP" %}
```php
// - Mauvaise approche : exécution système directe
system("ping -c 1 " . $_GET['ip']);

// - Bonne approche : fonction native sans shell
$sock = @fsockopen($_GET['ip'], 80, $errno, $errstr, 2);
if ($sock) {
    echo "Hôte joignable";
    fclose($sock);
} else {
    echo "Hôte injoignable";
}
```
{% endtab %}

{% tab title="NodeJS" %}
```javascript
// - Mauvaise approche : exécution système directe
const { exec } = require('child_process');
exec(`ping -c 1 ${req.query.ip}`);

// - Bonne approche : module réseau natif
const net = require('net');
const socket = new net.Socket();
socket.setTimeout(2000);
socket.connect(80, req.query.ip, () => {
    res.send('Hôte joignable');
    socket.destroy();
});
socket.on('error', () => {
    res.send('Hôte injoignable');
});
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
Les fonctions à surveiller en PHP : `system()`, `exec()`, `shell_exec()`, `passthru()`, `popen()`, `proc_open()`. En NodeJS : `child_process.exec()`, `child_process.spawn()` avec l'option `shell: true`. Leur simple présence dans le code source doit déclencher une revue approfondie.
{% endhint %}

Quand une commande système est réellement indispensable (pas d'alternative native), l'entrée utilisateur ne doit jamais y être injectée directement. On passe par une whitelist d'arguments autorisés, ou on utilise un tableau d'arguments sans interpolation dans un shell.

### Validation de l'entrée utilisateur

La validation consiste à vérifier que l'entrée correspond au format attendu avant tout traitement. Elle doit être appliquée côté front-end (ergonomie) ET côté back-end (sécurité). La validation front-end seule n'est jamais suffisante, car toute requête HTTP peut être modifiée via un proxy comme Burp.

{% tabs %}
{% tab title="PHP" %}
```php
// - Validation d'une adresse IP avec filtre natif
if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
    // - Entrée valide, traitement autorisé
} else {
    die("Format invalide");
}

// - Validation avec regex pour un format personnalisé
if (preg_match('/^[a-zA-Z0-9\.\-]+$/', $_GET['hostname'])) {
    // - Entrée valide
} else {
    die("Caractères non autorisés");
}
```
{% endtab %}

{% tab title="JavaScript" %}
```javascript
// - Validation d'une adresse IP avec regex
const ipRegex = /^(25[0-5]|2[0-4]\d|[01]?\d\d?)\.(25[0-5]|2[0-4]\d|[01]?\d\d?)\.(25[0-5]|2[0-4]\d|[01]?\d\d?)\.(25[0-5]|2[0-4]\d|[01]?\d\d?)$/;

if (ipRegex.test(ip)) {
    // - Entrée valide
} else {
    return res.status(400).send("Format invalide");
}

// - Avec la bibliothèque is-ip (NodeJS)
const isIp = require('is-ip');
if (isIp(req.query.ip)) {
    // - Entrée valide
}
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Chaque langage dispose de fonctions de validation intégrées pour les formats courants (IPs, emails, URLs). Toujours privilégier ces fonctions natives plutôt que des regex écrites à la main, qui sont plus sujettes aux erreurs et aux oublis de cas limites.
{% endhint %}

### Sanitisation de l'entrée

La sanitisation intervient après la validation. Même si le format est correct, on supprime tout caractère spécial qui n'est pas strictement nécessaire. L'objectif est de neutraliser les tentatives d'injection que la validation aurait pu laisser passer (regex mal écrite, cas limite, double encodage).

{% tabs %}
{% tab title="PHP" %}
```php
// - Ne conserver que les alphanumériques et le point
$ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);

// - Dernier recours : échappement des caractères spéciaux
// - Moins fiable que la suppression, contournable par obfuscation
$ip = escapeshellcmd($_GET['ip']);
```
{% endtab %}

{% tab title="JavaScript" %}
```javascript
// - Suppression des caractères non autorisés
let ip = req.query.ip.replace(/[^A-Za-z0-9.]/g, '');

// - Avec DOMPurify (NodeJS)
const DOMPurify = require('dompurify');
let sanitized = DOMPurify.sanitize(input);
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
La blacklist de caractères et l'échappement (`escapeshellcmd`, `escape()`) sont des mesures fragiles. Comme démontré tout au long de ce module, les techniques d'obfuscation (variables d'environnement, encodage base64, inversion de commandes) permettent de contourner la plupart des filtres basés sur des listes noires. La suppression stricte par whitelist de caractères autorisés reste l'approche la plus robuste.
{% endhint %}

### Configuration du serveur

Le durcissement du serveur constitue la dernière couche de défense. Il ne remplace pas la correction du code, mais limite les conséquences en cas de compromission.

| Mesure | Configuration | Effet |
|---|---|---|
| WAF applicatif | ModSecurity (Apache), WAF Nginx | Détection et blocage de patterns d'injection |
| WAF externe | Cloudflare, Fortinet, Imperva | Protection réseau en amont du serveur |
| Moindre privilège | Utilisateur `www-data` | Limite les actions possibles après compromission |
| Fonctions désactivées | `disable_functions` dans php.ini | Empêche l'exécution de commandes même en cas d'injection |
| Restriction de répertoire | `open_basedir` dans php.ini | Cantonne l'accès fichier au répertoire web |
| Encodage strict | Rejet du double encodage | Bloque les tentatives de contournement par encodage |

```ini
; - Exemple de durcissement dans php.ini
disable_functions = system,exec,passthru,shell_exec,popen,proc_open
open_basedir = /var/www/html
allow_url_include = Off
allow_url_fopen = Off
```

{% hint style="success" %}
L'isolation par conteneur Docker ajoute une couche supplémentaire. Même si l'attaquant obtient une exécution de commandes, il reste confiné dans le conteneur avec un accès limité au système de fichiers et au réseau.
{% endhint %}

## En pratique

### Checklist de sécurisation

Depuis un audit de code ou une revue de configuration :

```bash
# - Rechercher les fonctions d'exécution système dans le code PHP
grep -rn "system\|exec\|shell_exec\|passthru\|popen\|proc_open" /var/www/html/

# - Rechercher les fonctions dangereuses en NodeJS
grep -rn "child_process\|exec\|spawn" /var/www/app/

# - Vérifier la configuration PHP
php -i | grep -E "disable_functions|open_basedir|allow_url"

# - Vérifier que le serveur web tourne avec des privilèges réduits
ps aux | grep -E "apache|nginx|httpd" | head -5
```

### Ordre de priorité des corrections

1. Remplacer les fonctions d'exécution système par des alternatives natives
2. Si impossible, valider strictement l'entrée (whitelist de format)
3. Sanitiser en supprimant tout caractère non nécessaire (pas de blacklist)
4. Durcir la configuration serveur (disable_functions, open_basedir)
5. Déployer un WAF en complément (jamais comme seule défense)

## Pièges et galères

- **Confiance dans la validation front-end** : toute validation JavaScript peut être contournée en modifiant la requête HTTP via Burp ou cURL. La validation back-end est la seule qui compte en matière de sécurité.
- **Blacklist de caractères** : approche condamnée. Les techniques d'obfuscation (variables d'environnement, encodage, inversion) permettent de construire n'importe quel caractère sans l'écrire directement.
- **`escapeshellcmd` comme solution** : cette fonction échappe les caractères spéciaux mais ne protège pas contre toutes les formes d'injection. Elle peut être contournée par des techniques de substitution de commandes.
- **WAF comme seule protection** : un WAF bloque les patterns connus mais peut être contourné par des payloads obfusqués. Il doit compléter une correction du code, pas la remplacer.
- **Oubli des fonctions indirectes** : `preg_replace` avec le modificateur `/e` (deprecated), `assert()`, `eval()`, `create_function()` sont aussi des vecteurs d'injection de code.

## Retour terrain

En audit, la recommandation la plus fréquente reste la suppression des fonctions d'exécution système au profit d'alternatives natives. C'est la correction qui réduit le plus efficacement la surface d'attaque. Quand le client ne peut pas modifier le code (application tierce, legacy), le durcissement serveur (disable_functions + open_basedir) combiné à un WAF offre une protection raisonnable en attendant une correction de fond.

Un point souvent négligé dans les rapports : le test de la configuration après déploiement. Il ne suffit pas de recommander `disable_functions`, il faut vérifier que la directive est bien appliquée au bon php.ini (celui d'Apache, pas celui du CLI) et que le service a été redémarré. Le nombre de configurations correctes sur le papier mais jamais appliquées en production est surprenant.

## Mémo express

| Couche | Mesure | Fiabilité |
|---|---|---|
| Code | Fonctions natives (pas de shell) | La plus efficace |
| Code | Validation whitelist (format strict) | Très fiable |
| Code | Sanitisation (suppression de caractères) | Fiable |
| Code | Échappement (escapeshellcmd) | Fragile, contournable |
| Code | Blacklist de caractères | Insuffisant |
| Serveur | disable_functions | Bloque l'exécution même si injecté |
| Serveur | open_basedir | Limite l'accès fichier |
| Réseau | WAF (ModSecurity, Cloudflare) | Complément, pas solution |
| Infra | Conteneur Docker | Isolation post-compromission |

***
