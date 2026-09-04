# Interception et modification

## Pourquoi

L'interception est la fonctionnalite centrale d'un proxy web. Elle permet d'arreter une requete ou une reponse en transit, de la lire, de la modifier, puis de la laisser poursuivre son chemin. C'est par ce mecanisme qu'on teste les injections, les contournements d'authentification, les manipulations de parametres, et la plupart des vulnerabilites web.

## Comment ca marche

### Interception des requetes

Quand l'interception est active, chaque requete envoyee par le navigateur est mise en pause dans le proxy. On peut alors lire le contenu (methode, headers, body), modifier n'importe quel element, puis cliquer sur `Forward` (Burp) ou `Step`/`Continue` (ZAP) pour l'envoyer au serveur.

{% tabs %}
{% tab title="Burp" %}
Aller dans `Proxy > Intercept`. L'interception est active par defaut. Le bouton `Intercept is on/off` permet de basculer. Chaque requete interceptee est affichee et modifiable avant envoi.
{% endtab %}
{% tab title="ZAP" %}
L'interception est desactivee par defaut (bouton vert dans la barre d'outils). Cliquer dessus ou utiliser `Ctrl+B` pour l'activer. La requete interceptee apparait dans le panneau superieur droit.

ZAP propose aussi le HUD (Heads Up Display), une interface dans le navigateur pre-configure qui permet de controler l'interception directement depuis la page web.
{% endtab %}
{% endtabs %}

### Interception des reponses

On peut aussi intercepter les reponses du serveur avant qu'elles n'atteignent le navigateur. C'est utile pour modifier le rendu d'une page, activer des champs desactives, ou reveler des elements masques.

{% tabs %}
{% tab title="Burp" %}
Activer dans `Proxy > Proxy settings > Response interception rules` en cochant `Intercept Response`. Apres avoir forward la requete, la reponse sera elle aussi interceptee.
{% endtab %}
{% tab title="ZAP" %}
Quand une requete est interceptee, cliquer sur `Step` envoie la requete et intercepte automatiquement la reponse. Le HUD dispose aussi d'un bouton `Show/Enable` (icone ampoule) qui active les champs desactives et revele les champs masques sans avoir a intercepter la reponse.
{% endtab %}
{% endtabs %}

## En pratique

### Manipulation d'une requete

Supposons une page avec un formulaire qui n'accepte que des chiffres dans un champ `ip` (validation cote client). En interceptant la requete POST, on peut remplacer la valeur du parametre par n'importe quel payload :

```http
POST /ping HTTP/1.1
Host: <IP_CIBLE>
Content-Type: application/x-www-form-urlencoded

ip=;id;
```

Si le serveur ne valide pas l'entree cote backend, la commande est executee. C'est un exemple classique d'injection de commande rendu possible par l'interception.

### Modification d'une reponse

Pour contourner une validation cote client sans modifier chaque requete manuellement, on peut modifier la reponse HTML :

```html
<!-- Avant -->
<input type="number" id="ip" name="ip" min="1" max="255" maxlength="3">

<!-- Apres modification -->
<input type="text" id="ip" name="ip" maxlength="100">
```

Le champ accepte maintenant n'importe quelle saisie directement dans le navigateur.

### Modification automatique (Match & Replace)

Pour ne pas repeter l'interception a chaque requete, les deux outils permettent de definir des regles de remplacement automatique.

{% tabs %}
{% tab title="Burp" %}
Aller dans `Proxy > Proxy settings > HTTP match and replace rules`, cliquer sur `Add` :

- **Type** : `Request header` ou `Response body`
- **Match** : le pattern a remplacer (regex possible)
- **Replace** : la valeur de remplacement

Exemple : remplacer automatiquement le User-Agent :
- Match : `^User-Agent.*$` (regex)
- Replace : `User-Agent: CustomAgent 1.0`

Exemple : modifier la reponse pour changer un champ :
- Type : `Response body`
- Match : `type="number"`
- Replace : `type="text"`
{% endtab %}
{% tab title="ZAP" %}
Utiliser le `Replacer` (accessible via `Ctrl+R` ou le menu Options) :

- **Match Type** : `Request Header` ou `Response Body`
- **Match String** : la chaine a chercher
- **Replacement String** : la valeur de remplacement

ZAP permet aussi de definir des `Initiators` pour restreindre les regles a certains types de requetes.
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Les regles de remplacement automatique sont particulierement utiles pour desactiver des validations cote client de facon persistante, ou pour modifier systematiquement un header comme le User-Agent.
{% endhint %}

## Pieges et galeres

- L'interception bloque toutes les requetes du navigateur, y compris les requetes de fond (CSS, JS, images). On peut se retrouver a forward des dizaines de requetes avant d'atteindre celle qui nous interesse.
- Modifier une reponse sans desactiver l'interception ensuite bloque la navigation. Penser a desactiver l'interception apres la modification.
- Les regles de Match & Replace en regex mal ecrites peuvent casser les requetes silencieusement. Toujours tester avec une requete simple d'abord.

## Retour terrain

L'interception est l'outil de base pour tout test web. La capacite a modifier une requete en vol est ce qui differencie un test de penetration d'un simple scan automatise. Les regles de remplacement automatique sont un gain de temps enorme quand on travaille sur une application avec beaucoup de validation cote client. En engagement, on les configure des le debut pour eviter de perdre du temps a contourner les memes restrictions manuellement.

## Memo express

| Action | Burp | ZAP |
|---|---|---|
| Activer l'interception | `Proxy > Intercept is on` | Bouton vert (barre) ou `Ctrl+B` |
| Intercepter les reponses | `Proxy settings > Response interception rules` | `Step` apres interception de la requete |
| Forward/Continuer | `Forward` | `Step` ou `Continue` |
| Match & Replace | `Proxy settings > HTTP match and replace rules` | `Replacer` (`Ctrl+R`) |
| Activer champs masques | `Proxy settings > Response modification rules` | HUD > bouton ampoule |

***
