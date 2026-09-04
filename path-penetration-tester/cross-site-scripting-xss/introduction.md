# Introduction au Cross-Site Scripting (XSS)

Le Cross-Site Scripting fait partie des vulnérabilités web les plus répandues. On en trouve dans à peu près toutes les applications modernes, des petits formulaires de contact aux géants comme Google ou Twitter. Le principe est simple : injecter du code JavaScript dans une page web pour qu'il s'exécute dans le navigateur d'un autre utilisateur. Les conséquences vont du vol de session à l'exécution d'actions au nom de la victime.

## Pourquoi s'y intéresser

Le XSS occupe une place récurrente dans le Top 10 OWASP, et pour cause : la probabilité de le trouver est élevée, même si l'impact direct reste souvent modéré (l'exécution se limite au navigateur, pas au serveur).

En matière d'évaluation des risques, on applique généralement cette grille :

| | Impact faible | Impact élevé |
|---|---|---|
| **Probabilité élevée** | Réduire le risque | Éviter le risque |
| **Probabilité faible** | Accepter le risque | Transférer le risque |

Le XSS se place typiquement dans la case "impact faible + probabilité élevée", ce qui donne un risque moyen. Ce n'est pas catastrophique en soi, mais la fréquence de ces vulnérabilités rend leur détection et leur correction indispensables dans tout audit sérieux.

{% hint style="warning" %}
Ne pas sous-estimer le XSS sous prétexte qu'il "ne touche que le client". Un XSS bien exploité permet de voler des sessions admin, d'exfiltrer des données sensibles ou de piéger des utilisateurs avec du phishing crédible.
{% endhint %}

## Comment ça marche

Le scénario classique repose sur un défaut de sanitisation des entrées utilisateur. Quand une application web affiche du contenu fourni par un utilisateur sans le nettoyer, un attaquant peut y glisser du code JavaScript qui sera interprété par le navigateur de quiconque consulte la page.

Concrètement, le flux ressemble à ceci :

1. L'attaquant identifie un champ de saisie (commentaire, champ de recherche, paramètre d'URL)
2. Il y injecte un payload JavaScript, par exemple `<script>alert(document.cookie)</script>`
3. L'application stocke ou reflète cette entrée sans l'assainir
4. Lorsqu'un autre utilisateur charge la page, le navigateur exécute le JavaScript injecté
5. Le code malveillant s'exécute avec les privilèges de la session de la victime

{% hint style="info" %}
Le XSS s'exécute uniquement côté client, dans le moteur JavaScript du navigateur (V8 pour Chrome, SpiderMonkey pour Firefox). Il ne permet pas directement d'exécuter du code système sur le serveur. Par contre, combiné avec une vulnérabilité du navigateur lui-même (overflow, use-after-free), un XSS peut servir de vecteur initial pour sortir de la sandbox et compromettre la machine de la victime.
{% endhint %}

Les actions possibles via un XSS sont nombreuses :

- Vol de cookies de session (session hijacking)
- Exécution d'appels API au nom de la victime (changement de mot de passe, transfert de fonds)
- Injection de formulaires de phishing dans la page légitime
- Redirection vers un site malveillant
- Keylogging dans le navigateur
- Minage de cryptomonnaie via le navigateur de la victime
- Défacement de la page web

Les navigateurs modernes limitent le XSS au même domaine que le site vulnérable, ce qui réduit le périmètre d'action. Mais dans ce périmètre, les possibilités restent larges.

## Les trois types de XSS

Il existe trois catégories principales, chacune avec ses caractéristiques et son niveau de criticité :

| Type | Persistance | Traitement | Criticité | Exemple typique |
|---|---|---|---|---|
| **Stored (Persistant)** | Oui, stocké en base de données | Côté serveur | Critique | Commentaire sur un forum, message dans un chat |
| **Reflected (Non-persistant)** | Non, présent uniquement dans la réponse | Côté serveur | Moyenne | Message d'erreur incluant l'entrée utilisateur, résultat de recherche |
| **DOM-based (Non-persistant)** | Non, traité uniquement côté client | Côté client (JavaScript) | Variable | Paramètre d'URL traité par du JS sans passer par le serveur |

### XSS Stored (Persistant)

Le plus dangereux des trois. Le payload est enregistré dans la base de données du serveur (via un commentaire, un profil, un message). Chaque visiteur qui charge la page affectée exécute le code malveillant, sans interaction supplémentaire. L'attaque persiste même après un rafraîchissement de page et touche potentiellement tous les utilisateurs du site.

{% hint style="danger" %}
Un XSS Stored sur une page à fort trafic peut compromettre des milliers de sessions en quelques heures. C'est aussi le plus difficile à nettoyer, car il faut identifier et purger l'entrée malveillante directement en base de données.
{% endhint %}

### XSS Reflected (Non-persistant)

Le payload n'est pas stocké mais renvoyé par le serveur dans sa réponse HTTP. On le retrouve souvent dans les messages d'erreur ou les pages de résultats de recherche qui intègrent l'entrée de l'utilisateur. Pour exploiter un Reflected XSS, l'attaquant doit amener la victime à cliquer sur un lien spécialement forgé contenant le payload.

L'attaque est temporaire : elle ne persiste pas entre les requêtes. Seul l'utilisateur qui suit le lien malveillant est affecté.

### XSS DOM-based

Le payload ne transite jamais par le serveur. Tout se passe côté client, dans le JavaScript de la page. Le navigateur manipule le DOM (Document Object Model) en utilisant directement des données contrôlées par l'utilisateur (fragments d'URL avec `#`, paramètres traités par du JS côté client). Si le code JavaScript de la page utilise ces données sans les assainir pour modifier le DOM, un attaquant peut injecter du contenu malveillant.

Comme le Reflected XSS, l'attaque est non-persistante et nécessite que la victime visite un lien forgé.

## En pratique

Pour tester rapidement la présence d'un XSS, le payload classique reste :

```html
<script>alert(window.origin)</script>
```

{% hint style="success" %}
On utilise `window.origin` plutôt qu'une valeur statique comme `1` pour identifier précisément quel domaine exécute le code. C'est particulièrement utile quand l'application utilise des iframes cross-domain : l'alerte révèle immédiatement si c'est le formulaire principal ou l'iframe qui est vulnérable.
{% endhint %}

Si la balise `<script>` est filtrée (ce qui est courant dans les contextes DOM), on peut utiliser des alternatives :

```html
<img src="" onerror=alert(window.origin)>
```

```html
<plaintext>
```

```html
<script>print()</script>
```

Depuis un environnement Exegol, la démarche typique consiste à :

1. Identifier tous les points d'entrée utilisateur (champs de formulaire, paramètres GET/POST, en-têtes HTTP)
2. Injecter un payload de test dans chaque point d'entrée
3. Observer si le payload est exécuté (alerte, modification de la page, requête vers un serveur contrôlé)
4. Analyser le code source de la page pour comprendre comment l'entrée est traitée

## Pièges et galères

**Le payload fonctionne mais rien ne s'affiche.** Certains navigateurs modernes bloquent `alert()` dans certains contextes (iframes sandboxées, par exemple). Utiliser `print()` ou `console.log()` comme alternative de détection.

**Le XSS semble présent mais le payload ne persiste pas.** Vérifier s'il s'agit d'un Reflected ou d'un DOM-based plutôt que d'un Stored. Rafraîchir la page : si l'alerte ne revient pas, ce n'est pas persistant.

**L'entrée est affichée mais les balises sont encodées.** L'application applique probablement un encodage HTML (`&lt;` au lieu de `<`). Il faut chercher d'autres points d'injection ou tenter des techniques de contournement.

**Le filtre bloque `<script>` mais pas les événements HTML.** De nombreux filtres se contentent de bloquer la balise `<script>`. Les payloads basés sur des événements (`onerror`, `onload`, `onfocus`) passent souvent au travers.

## Retour terrain

Les XSS les plus marquants de l'histoire du web illustrent bien le potentiel de cette vulnérabilité :

**Le ver Samy (MySpace, 2005).** Un XSS Stored auto-réplicant qui ajoutait le message "Samy is my hero" sur le profil de chaque visiteur infecté, tout en propageant le même code. En 24 heures, plus d'un million de profils étaient touchés. Le payload n'était pas destructif, mais il aurait pu l'être : vol de données personnelles, installation de keyloggers, exploitation de vulnérabilités navigateur.

**TweetDeck (Twitter, 2014).** Un chercheur en sécurité a découvert par accident un XSS dans le tableau de bord TweetDeck. La vulnérabilité a été exploitée pour créer un tweet auto-retweeté, qui a atteint plus de 38 000 retweets en moins de deux minutes. Twitter a dû fermer temporairement TweetDeck pour corriger la faille.

**Google Search (2019).** Un XSS a été découvert dans la bibliothèque XML utilisée par le moteur de recherche Google. Même les applications les plus scrutées et les mieux sécurisées ne sont pas à l'abri.

**Apache Server.** Le serveur web le plus utilisé au monde a connu un XSS activement exploité pour voler les mots de passe d'utilisateurs de certaines entreprises.

{% hint style="info" %}
Ces exemples montrent que le XSS n'est pas une vulnérabilité "de débutant" qu'on ne trouve que sur des sites amateurs. Les applications les plus matures et les plus testées en sont régulièrement affectées.
{% endhint %}

## Mémo express

| Élément | Détail |
|---|---|
| **Définition** | Injection de JavaScript côté client via un défaut de sanitisation |
| **Exécution** | Dans le navigateur de la victime, pas sur le serveur |
| **Portée** | Limitée au domaine du site vulnérable (same-origin policy) |
| **XSS Stored** | Persistant, stocké en BDD, affecte tous les visiteurs |
| **XSS Reflected** | Non-persistant, renvoyé dans la réponse HTTP, nécessite un lien forgé |
| **XSS DOM-based** | Non-persistant, traité côté client, ne passe pas par le serveur |
| **Payload de test** | `<script>alert(window.origin)</script>` |
| **Alternative DOM** | `<img src="" onerror=alert(window.origin)>` |
| **Risque** | Moyen (impact faible + probabilité élevée) |
| **Impacts possibles** | Vol de session, phishing, défacement, keylogging, minage |

***
