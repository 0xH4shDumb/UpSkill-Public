# Types de XSS

Il existe trois grandes familles de Cross-Site Scripting : le Stored XSS (persistant), le Reflected XSS (non persistant côté serveur) et le DOM-based XSS (non persistant côté client). Chacune a ses propres conditions d'exploitation et son propre niveau d'impact. Comprendre ces différences, c'est savoir où chercher et comment adapter ses payloads.

## Pourquoi distinguer les trois types

La classification ne sert pas juste à cocher des cases dans un rapport. Elle détermine directement la stratégie d'exploitation.

| Type | Persistance | Traitement | Impact |
|---|---|---|---|
| **Stored (Persistent)** | Oui, stocké en base | Backend | Touche tous les utilisateurs qui visitent la page |
| **Reflected (Non-Persistent)** | Non | Backend | Nécessite que la victime clique sur un lien piégé |
| **DOM-based** | Non | Client-side uniquement | Nécessite un lien piégé, jamais traité par le serveur |

{% hint style="danger" %}
Le Stored XSS est le plus critique : une seule injection suffit pour compromettre tous les visiteurs de la page affectée. Le payload s'exécute à chaque chargement, sans interaction supplémentaire.
{% endhint %}

---

## Stored XSS (Persistant)

### Comment ça marche

Le Stored XSS se produit quand une entrée utilisateur est enregistrée en base de données, puis réaffichée sans assainissement lors de la consultation de la page. Les cibles classiques : champs de commentaires, profils utilisateurs, forums, tickets de support.

Le cycle est simple :

1. L'attaquant soumet un payload JavaScript dans un champ de saisie
2. Le serveur stocke cette entrée en base de données
3. Quand un autre utilisateur charge la page, le serveur récupère le contenu depuis la base et l'injecte dans le HTML
4. Le navigateur de la victime exécute le JavaScript malveillant

{% hint style="info" %}
La persistance est ce qui rend ce type particulièrement dangereux. L'attaquant n'a pas besoin de piéger individuellement chaque victime. Le payload reste actif tant qu'il n'est pas supprimé de la base.
{% endhint %}

### En pratique

Le premier réflexe pour tester un Stored XSS est d'injecter un payload basique dans un champ de saisie :

```html
<script>alert(window.origin)</script>
```

{% hint style="success" %}
On utilise `window.origin` plutôt qu'une valeur statique comme `1` pour une raison précise : beaucoup d'applications modernes utilisent des IFrames cross-domain pour gérer les entrées utilisateur. Si le formulaire est dans un IFrame, `window.origin` révèle l'URL exacte où le script s'exécute, ce qui permet de confirmer quel formulaire est réellement vulnérable.
{% endhint %}

Si l'alerte apparaît avec l'URL de la page, on peut vérifier l'injection dans le code source (`Ctrl+U`) :

```html
<div><ul class="list-unstyled" id="todo"><ul>
<script>alert(window.origin)</script>
</ul></ul>
```

Pour confirmer la persistance, il suffit de rafraîchir la page. Si l'alerte réapparaît, c'est bien un Stored XSS. Tout utilisateur qui visite cette page déclenchera le payload.

**Payloads alternatifs** : certains navigateurs modernes bloquent `alert()` dans certains contextes. Deux alternatives utiles :

```html
<plaintext>
```

Ce tag arrête le rendu HTML et affiche tout le reste en texte brut. Très visible si l'injection fonctionne.

```html
<script>print()</script>
```

Ouvre la boîte de dialogue d'impression du navigateur. Aucun navigateur ne bloque cette fonction.

### Pièges et galères

- Le payload peut être stocké mais encodé en sortie. L'alerte ne se déclenche pas, mais l'injection existe bien. Vérifier le code source pour voir comment l'entrée est rendue.
- Certaines applications limitent la longueur des champs en front-end. Intercepter la requête avec Burp pour contourner cette limitation.
- Si l'entrée passe par un éditeur WYSIWYG, le contenu peut être transformé (balises supprimées, entités HTML encodées) avant le stockage.

---

## Reflected XSS (Non-Persistant)

### Comment ça marche

Le Reflected XSS se produit quand le serveur renvoie l'entrée utilisateur dans sa réponse HTTP sans la stocker. Les cas typiques : messages d'erreur qui reprennent la saisie, pages de résultats de recherche, messages de confirmation.

Le flux est différent du Stored XSS :

1. L'attaquant construit une URL contenant un payload dans un paramètre
2. La victime clique sur ce lien (souvent via phishing ou ingénierie sociale)
3. Le serveur traite la requête et inclut le paramètre dans la réponse HTML
4. Le navigateur de la victime exécute le JavaScript

{% hint style="warning" %}
Le Reflected XSS disparaît dès qu'on quitte la page ou qu'on la rafraîchit sans le paramètre malveillant. La portée est limitée à la victime qui clique sur le lien piégé.
{% endhint %}

### En pratique

Face à un champ qui renvoie un message d'erreur du type `Task 'test' could not be added.`, on injecte le même payload de base :

```html
<script>alert(window.origin)</script>
```

Si le serveur ne filtre pas, l'alerte apparaît. Le message d'erreur affiche alors des guillemets vides (`Task '' could not be added.`) parce que le contenu entre les balises `<script>` n'est pas rendu visuellement par le navigateur.

Dans le code source, on retrouve le payload :

```html
<div style="padding-left:25px">Task '<script>alert(window.origin)</script>' could not be added.</div>
```

**Identifier le vecteur d'exploitation** : ouvrir les DevTools du navigateur (`Ctrl+Shift+I`), aller dans l'onglet Réseau, et soumettre le payload. Si la requête est un `GET`, les paramètres sont dans l'URL. C'est ce qui permet de construire un lien piégé.

```
http://<IP_CIBLE>/index.php?task=<script>alert(window.origin)</script>
```

Il suffit d'envoyer cette URL à la victime. Quand elle clique, le payload s'exécute dans son navigateur.

{% hint style="info" %}
Si la requête utilise `POST` au lieu de `GET`, l'exploitation est plus complexe. Il faut alors soit héberger un formulaire HTML qui soumet automatiquement la requête POST, soit trouver un moyen de convertir la requête en GET (certaines applications acceptent les deux méthodes).
{% endhint %}

### Pièges et galères

- Un payload qui fonctionne dans la barre d'adresse peut échouer quand il est envoyé par lien, parce que le navigateur encode automatiquement certains caractères dans les URL.
- Certains WAF détectent `<script>` dans les paramètres GET. Tester des variantes comme `<img src=x onerror=alert(1)>` ou encoder le payload.
- Si l'entrée apparaît dans un attribut HTML (par exemple `value="INPUT"`), il faut d'abord fermer l'attribut et la balise avant d'injecter du JavaScript.

---

## DOM-based XSS

### Comment ça marche

Le DOM XSS est entièrement traité côté client. Le serveur ne voit jamais le payload, aucune requête HTTP n'est envoyée. L'entrée utilisateur est manipulée par du JavaScript qui modifie le DOM (Document Object Model) de la page directement dans le navigateur.

Le mécanisme repose sur deux éléments :

- **Source** : l'objet JavaScript qui récupère l'entrée utilisateur (paramètre d'URL, champ de saisie, fragment `#` de l'URL)
- **Sink** : la fonction JavaScript qui écrit cette entrée dans le DOM

Si le Sink n'assainit pas l'entrée, il y a vulnérabilité.

**Sinks dangereux en JavaScript natif** :

| Fonction | Risque |
|---|---|
| `document.write()` | Écrit directement dans le HTML |
| `DOM.innerHTML` | Remplace le contenu HTML d'un élément |
| `DOM.outerHTML` | Remplace l'élément entier et son contenu |

**Sinks dangereux en jQuery** :

| Fonction | Risque |
|---|---|
| `add()` | Ajoute des éléments au DOM |
| `after()` | Insère du contenu après un élément |
| `append()` | Ajoute du contenu à la fin d'un élément |

### En pratique

Un signe révélateur du DOM XSS : quand on soumet une entrée et qu'aucune requête n'apparaît dans l'onglet Réseau des DevTools. L'URL utilise souvent un fragment (`#`) au lieu d'un paramètre classique (`?`).

Pour analyser la vulnérabilité, on examine le code JavaScript de la page. Voici un exemple de Source vulnérable :

```javascript
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);
```

Et le Sink correspondant :

```javascript
document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);
```

L'entrée passe directement de l'URL au DOM sans aucun filtrage. La vulnérabilité est confirmée.

{% hint style="warning" %}
La fonction `innerHTML` bloque l'exécution des balises `<script>` comme mesure de sécurité. Le payload classique `<script>alert(...)</script>` ne fonctionnera pas ici. Il faut utiliser des alternatives basées sur les événements HTML.
{% endhint %}

Le payload adapté pour `innerHTML` :

```html
<img src="" onerror=alert(window.origin)>
```

Ce payload crée un élément image avec une source vide. Comme l'image ne peut pas être chargée, l'événement `onerror` se déclenche et exécute le JavaScript. Pas besoin de balises `<script>`.

Pour exploiter la vulnérabilité, on copie l'URL contenant le payload (via le fragment `#`) et on l'envoie à la victime, exactement comme pour un Reflected XSS.

{% hint style="info" %}
Le DOM XSS est invisible pour les outils de détection côté serveur (WAF, logs). La requête HTTP ne contient aucune trace du payload quand celui-ci est dans le fragment (`#`). Seule une analyse du code JavaScript côté client permet de le détecter.
{% endhint %}

### Pièges et galères

- Le code source brut (`Ctrl+U`) ne montre pas l'injection, car le DOM est modifié après le chargement initial. Utiliser l'inspecteur web (`Ctrl+Shift+C`) pour voir le DOM rendu.
- Certains frameworks JavaScript (React, Angular, Vue) échappent automatiquement les entrées insérées dans le DOM via leur système de templates. Mais l'utilisation directe de `innerHTML` ou de `dangerouslySetInnerHTML` (React) contourne ces protections.
- Les fragments d'URL (`#`) ne sont pas envoyés au serveur. Si l'application utilise des paramètres classiques (`?param=value`) pour le DOM XSS, le serveur voit la requête mais ne traite pas le paramètre de la même manière.

---

## Retour terrain

En audit, la majorité des XSS découverts sont des Reflected XSS dans des fonctionnalités de recherche ou des messages d'erreur. Le Stored XSS est plus rare mais bien plus impactant, surtout dans les applications avec beaucoup d'utilisateurs.

Le DOM XSS est souvent le plus difficile à trouver parce qu'il nécessite de lire du JavaScript, pas juste de tester des payloads. Les Single Page Applications (React, Angular, Vue) sont particulièrement exposées si les développeurs manipulent le DOM directement au lieu d'utiliser le data binding du framework.

Un point important : la sévérité d'un XSS dépend du contexte. Un Stored XSS dans un commentaire public d'un site à fort trafic est critique. Un Reflected XSS dans un paramètre accessible uniquement après authentification est moins impactant parce que le lien piégé ne fonctionnera que si la victime est déjà connectée.

---

## Mémo express

| Élément | Stored | Reflected | DOM |
|---|---|---|---|
| **Persistance** | Oui (en base) | Non | Non |
| **Traitement** | Backend | Backend | Client-side |
| **Vecteur** | Visite de la page | Clic sur un lien | Clic sur un lien |
| **Visibilité serveur** | Oui | Oui | Non (fragment `#`) |
| **Payload basique** | `<script>alert(window.origin)</script>` | `<script>alert(window.origin)</script>` | `<img src="" onerror=alert(window.origin)>` |
| **Confirmation** | Rafraîchir la page | Vérifier le code source | Inspecter le DOM rendu |
| **Cibles typiques** | Commentaires, profils, forums | Recherche, erreurs, confirmations | SPA, paramètres URL, fragments |

{% hint style="success" %}
Règle pratique : toujours tester `<script>alert(window.origin)</script>` en premier. Si ça ne passe pas, essayer `<img src=x onerror=alert(window.origin)>`. Si rien ne fonctionne en aveugle, analyser le code source pour comprendre comment l'entrée est traitée et adapter le payload au contexte d'injection.
{% endhint %}

***
