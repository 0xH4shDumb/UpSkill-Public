# Prévention des XSS

Se protéger contre les failles XSS ne repose pas sur une seule mesure miracle. C'est un ensemble de pratiques coordonnées entre le frontend, le backend et la configuration serveur qui permet de réduire la surface d'attaque. Chaque couche compense les faiblesses potentielles des autres.

## Pourquoi

Une XSS exploitée avec succès peut mener au vol de session, au phishing transparent, au defacement ou à l'exécution d'actions au nom de la victime. Les conséquences vont de la perte de confiance des utilisateurs à la compromission complète de comptes privilégiés. La prévention doit intervenir à chaque point où une donnée utilisateur entre, transite ou s'affiche dans l'application.

{% hint style="danger" %}
La validation côté client seule ne protège rien. Un attaquant peut contourner n'importe quel contrôle JavaScript en forgeant directement ses requêtes HTTP. La validation frontend améliore l'expérience utilisateur, mais la sécurité repose sur le backend.
{% endhint %}

## Comment ça marche

La prévention XSS s'articule autour de trois axes : valider et assainir les entrées, encoder les sorties, et configurer le serveur pour limiter l'impact d'une injection qui passerait malgré tout.

### Côté frontend

#### Validation des entrées

Le premier réflexe consiste à vérifier que chaque champ de saisie correspond au format attendu avant de l'envoyer au serveur. Par exemple, pour valider un champ email en JavaScript :

```javascript
function validateEmail(email) {
    const re = /^(([^<>()[\]\\.,;:\s@"]+(\.[^<>()[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test(email);
}
```

Cette approche rejette les entrées qui ne correspondent pas au format attendu avant même qu'elles n'atteignent le serveur. Elle s'applique à tous les types de champs : numéros de téléphone, noms d'utilisateur, URLs.

#### Assainissement des entrées

Au-delà de la validation du format, il faut neutraliser tout code potentiellement dangereux dans les entrées utilisateur. La bibliothèque [DOMPurify](https://github.com/cure53/DOMPurify) est la référence pour cette tâche :

```html
<script type="text/javascript" src="dist/purify.min.js"></script>
```

```javascript
let clean = DOMPurify.sanitize(dirty);
```

DOMPurify échappe les caractères spéciaux avec un antislash (`\`), ce qui empêche l'exécution de code JavaScript injecté dans les entrées. C'est particulièrement utile pour prévenir les XSS de type DOM.

#### Éviter l'injection directe dans le HTML

Il ne faut jamais insérer une entrée utilisateur directement dans certains contextes HTML :

| Contexte dangereux | Exemple |
|---|---|
| Balises JavaScript | `<script>USER_INPUT</script>` |
| Blocs de style CSS | `<style>USER_INPUT</style>` |
| Attributs de balise | `<div name="USER_INPUT"></div>` |
| Commentaires HTML | `<!-- USER_INPUT -->` |

Dans chacun de ces cas, un attaquant peut injecter du code qui sera interprété par le navigateur.

#### Fonctions JavaScript dangereuses

Certaines fonctions permettent de modifier le contenu HTML brut de la page. Si elles reçoivent une entrée utilisateur non assainie, elles deviennent des vecteurs d'injection :

**Fonctions JavaScript natives à éviter avec des données non fiables :**

- `DOM.innerHTML`
- `DOM.outerHTML`
- `document.write()`
- `document.writeln()`
- `document.domain`

**Fonctions jQuery à éviter avec des données non fiables :**

- `html()`, `parseHTML()`
- `add()`, `append()`, `prepend()`
- `after()`, `insertAfter()`
- `before()`, `insertBefore()`
- `replaceAll()`, `replaceWith()`

{% hint style="warning" %}
Ces fonctions ne sont pas à bannir de tout projet. Elles sont dangereuses uniquement quand elles reçoivent du texte brut provenant d'une source non contrôlée. Si l'entrée a été correctement assainie en amont, leur utilisation reste légitime.
{% endhint %}

### Côté backend

#### Validation des entrées

Le backend doit reproduire (et renforcer) les contrôles du frontend. En PHP, la fonction `filter_var` permet de valider rapidement différents types de données :

```php
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) {
    // traiter la requete
} else {
    // rejeter l'entree - ne pas l'afficher
}
```

En NodeJS, les mêmes expressions régulières utilisées côté client peuvent servir de validation côté serveur.

#### Assainissement des entrées

C'est côté serveur que l'assainissement est véritablement critique, car les contrôles frontend sont facilement contournables via des requêtes HTTP forgées.

En PHP, la fonction `addslashes` échappe les caractères spéciaux :

```php
$safe_input = addslashes($_GET['email']);
```

{% hint style="danger" %}
Ne jamais afficher directement une entrée utilisateur brute (comme `$_GET['email']` ou `$_POST['comment']`) dans le HTML de la page. C'est le vecteur d'injection le plus classique.
{% endhint %}

En NodeJS, DOMPurify fonctionne aussi côté serveur :

```javascript
import DOMPurify from 'dompurify';
var clean = DOMPurify.sanitize(dirty);
```

#### Encodage HTML en sortie

Même après validation et assainissement, il est recommandé d'encoder les caractères spéciaux en entités HTML avant de les afficher. Cela garantit que le navigateur les rendra comme du texte, pas comme du code.

En PHP :

```php
echo htmlentities($_GET['comment']);
// <script> devient &lt;script&gt;
```

Les fonctions `htmlspecialchars()` et `htmlentities()` convertissent les caractères comme `<`, `>`, `"`, `'` et `&` en leurs équivalents HTML (`&lt;`, `&gt;`, etc.).

En NodeJS, la bibliothèque `html-entities` offre la même fonctionnalité :

```javascript
import encode from 'html-entities';
encode('<');  // resultat : '&lt;'
```

{% hint style="success" %}
L'encodage en sortie est la dernière ligne de défense. Même si une injection passe à travers la validation et l'assainissement, l'encodage empêche son exécution dans le navigateur.
{% endhint %}

### Configuration serveur

En complément du code applicatif, plusieurs configurations serveur permettent de réduire l'impact d'une XSS qui passerait malgré les protections :

| Mesure | Effet |
|---|---|
| HTTPS sur tout le domaine | Empêche l'interception des cookies et des données en transit |
| `X-Content-Type-Options: nosniff` | Empêche le navigateur d'interpréter un type MIME différent de celui déclaré |
| `Content-Security-Policy: script-src 'self'` | Autorise uniquement les scripts hébergés localement, bloque les scripts distants injectés |
| Cookie `HttpOnly` | Empêche JavaScript d'accéder aux cookies de session (bloque `document.cookie`) |
| Cookie `Secure` | Force le transport des cookies uniquement via HTTPS |
| En-têtes XSS prevention | Couche de protection supplémentaire dans les navigateurs compatibles |

{% hint style="info" %}
Le flag `HttpOnly` sur les cookies est l'une des mesures les plus efficaces contre le vol de session par XSS. Si `document.cookie` ne retourne pas le cookie de session, l'attaque de session hijacking échoue même si l'injection XSS fonctionne.
{% endhint %}

#### WAF et protections intégrées

Un Web Application Firewall (WAF) ajoute une couche de détection automatique des payloads XSS dans les requêtes HTTP. Il ne remplace pas le code sécurisé, mais il intercepte les attaques les plus courantes.

Certains frameworks intègrent nativement des protections anti-XSS. ASP.NET, par exemple, encode automatiquement les sorties par défaut. Django échappe les variables dans les templates. React échappe automatiquement le contenu rendu via JSX. Ces protections intégrées ne dispensent pas de vigilance, mais elles réduisent considérablement les risques d'oubli.

## En pratique

Pour sécuriser une application web contre les XSS, la démarche concrète suit un ordre logique :

1. **Valider** toutes les entrées côté frontend ET backend (regex, `filter_var`, bibliothèques de validation)
2. **Assainir** les données avec DOMPurify (frontend et NodeJS) ou `addslashes` (PHP) avant tout traitement
3. **Encoder** les sorties HTML avec `htmlentities()` (PHP) ou `html-entities` (NodeJS)
4. **Configurer** les en-têtes de sécurité : CSP, `X-Content-Type-Options`, flags `HttpOnly`/`Secure` sur les cookies
5. **Déployer** un WAF devant l'application pour intercepter les payloads courants

```bash
# - Exemple d'en-tetes de securite dans Apache
# Dans le fichier .htaccess ou la configuration du vhost
Header set X-Content-Type-Options "nosniff"
Header set Content-Security-Policy "script-src 'self'"
Header set X-XSS-Protection "1; mode=block"
```

```bash
# - Exemple dans Nginx
add_header X-Content-Type-Options "nosniff" always;
add_header Content-Security-Policy "script-src 'self'" always;
add_header X-XSS-Protection "1; mode=block" always;
```

## Pièges et galères

- **Se fier uniquement au frontend** : la validation JavaScript est triviale à contourner. Un simple `curl` ou Burp Suite suffit pour envoyer des données brutes au serveur.
- **Oublier l'encodage en sortie** : valider et assainir les entrées sans encoder les sorties laisse une porte ouverte si un cas limite passe à travers les filtres.
- **CSP trop permissive** : une Content-Security-Policy qui autorise `'unsafe-inline'` ou `'unsafe-eval'` perd l'essentiel de son utilité contre les XSS.
- **Confondre assainissement et encodage** : l'assainissement retire ou neutralise le code dangereux à l'entrée, l'encodage transforme les caractères spéciaux à la sortie. Les deux sont complémentaires, pas interchangeables.
- **Croire qu'un WAF suffit** : les WAF sont contournables avec des techniques d'obfuscation. Ils ajoutent une couche, mais ne remplacent pas le code sécurisé.

## Retour terrain

En pentest, on rencontre régulièrement des applications qui appliquent une ou deux mesures de protection, mais rarement les trois couches simultanément (validation, assainissement, encodage). Les failles XSS persistent souvent dans les champs secondaires auxquels les développeurs pensent moins : en-têtes User-Agent affichés dans des logs administratifs, paramètres de recherche reflétés dans la page, ou données stockées dans des champs "libres" comme les commentaires ou les descriptions de profil.

L'approche la plus robuste consiste à traiter toute donnée externe comme potentiellement hostile, qu'elle vienne d'un formulaire, d'une URL, d'un en-tête HTTP, ou même d'une base de données (qui a pu être alimentée par une source non fiable).

## Mémo express

| Couche | Mesure | Outil/Fonction |
|---|---|---|
| Frontend | Validation de format | Regex, `filter_var` (PHP) |
| Frontend | Assainissement | DOMPurify |
| Frontend | Éviter injection directe | Pas d'entrée dans `<script>`, `<style>`, attributs |
| Backend | Validation | `filter_var()`, regex serveur |
| Backend | Assainissement | `addslashes()` (PHP), DOMPurify (Node) |
| Backend | Encodage sortie | `htmlentities()` (PHP), `html-entities` (Node) |
| Serveur | En-têtes de sécurité | CSP, `X-Content-Type-Options`, `X-XSS-Protection` |
| Serveur | Cookies sécurisés | Flags `HttpOnly` + `Secure` |
| Serveur | Protection réseau | WAF (ModSecurity, Cloudflare, etc.) |
| Framework | Protection intégrée | ASP.NET, Django, React (échappement auto) |

***
