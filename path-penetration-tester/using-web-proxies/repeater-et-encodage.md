# Repeater et encodage

## Pourquoi

Intercepter et modifier une requete a chaque fois qu'on veut tester un parametre est fastidieux. Le Repeater permet de renvoyer une requete a volonte en modifiant ses parametres, sans repasser par le navigateur. Combine aux outils d'encodage/decodage integres, il permet de travailler efficacement sur des payloads encodes ou des cookies chiffres.

## Comment ca marche

### Historique du proxy

Chaque requete qui passe par le proxy est enregistree dans l'historique. C'est le point de depart pour envoyer une requete au Repeater : on la selectionne dans l'historique, on l'envoie dans l'outil de repetition, et on peut ensuite la modifier et la renvoyer autant de fois que necessaire.

{% tabs %}
{% tab title="Burp" %}
L'historique est dans `Proxy > HTTP History`. Chaque requete affiche la methode, l'URL, le code de reponse et la taille. Burp permet de voir la requete originale et la version modifiee si elle a ete alteree.
{% endtab %}
{% tab title="ZAP" %}
L'historique est dans l'onglet `History` en bas de l'interface. Le HUD affiche aussi un historique dans le panneau inferieur du navigateur.
{% endtab %}
{% endtabs %}

### Repeater

{% tabs %}
{% tab title="Burp" %}
Depuis l'historique, selectionner une requete et appuyer sur `Ctrl+R` pour l'envoyer au Repeater. Basculer vers le Repeater avec `Ctrl+Shift+R`. Modifier la requete dans le panneau gauche, cliquer sur `Send`, et lire la reponse dans le panneau droit.

Clic droit sur la requete pour `Change Request Method` (bascule POST/GET sans tout reecrire).
{% endtab %}
{% tab title="ZAP" %}
Clic droit sur la requete dans l'historique, puis `Open/Resend with Request Editor`. La fenetre permet de modifier la requete et de la renvoyer avec `Send`. Un menu deroulant permet de changer la methode HTTP directement.

Dans le HUD, cliquer sur une requete de l'historique ouvre le Request Editor. Choisir `Replay in Console` (reponse dans le HUD) ou `Replay in Browser` (reponse rendue dans le navigateur).
{% endtab %}
{% endtabs %}

## En pratique

### Tester une injection de commande avec le Repeater

Apres avoir intercepte une requete POST vulnerable :

```http
POST /ping HTTP/1.1
Host: <IP_CIBLE>
Content-Type: application/x-www-form-urlencoded

ip=;id;
```

On envoie cette requete au Repeater, puis on modifie le parametre `ip` pour tester differentes commandes sans repasser par le navigateur :

```
ip=;cat /etc/passwd;
ip=;whoami;
ip=;ls -la /;
```

Chaque modification est envoyee instantanement et la reponse est visible dans le meme panneau.

### Encodage URL

Les donnees envoyees dans une requete HTTP doivent etre encodees en URL. Certains caracteres ont une signification speciale et doivent etre echappes :

| Caractere | Signification HTTP | Encodage URL |
|---|---|---|
| Espace | Fin du champ de donnees | `%20` ou `+` |
| `&` | Separateur de parametres | `%26` |
| `#` | Identifiant de fragment | `%23` |
| `=` | Separateur cle/valeur | `%3D` |

{% tabs %}
{% tab title="Burp" %}
Dans le Repeater, selectionner le texte a encoder et utiliser `Ctrl+U` pour l'encoder en URL. Le clic droit offre aussi `Convert Selection > URL > URL-encode key characters`. L'option d'encodage automatique a la saisie est disponible via clic droit.
{% endtab %}
{% tab title="ZAP" %}
ZAP encode automatiquement les donnees de la requete en arriere-plan avant l'envoi.
{% endtab %}
{% endtabs %}

### Decoder/Encoder des donnees

Les applications web encodent souvent des donnees en Base64, hex, ou d'autres formats. Les deux outils integrent un decodeur complet.

{% tabs %}
{% tab title="Burp" %}
Aller dans l'onglet `Decoder`. Coller la chaine, puis `Decode as > Base64` (ou tout autre format). Pour encoder, coller le texte clair et choisir `Encode as > Base64`.

Le `Burp Inspector` (dans Proxy ou Repeater) permet aussi de decoder les valeurs directement dans le contexte de la requete.

Exemple de decodage Base64 :

```
eyJ1c2VybmFtZSI6Imd1ZXN0IiwgImlzX2FkbWluIjpmYWxzZX0=
→ {"username":"guest", "is_admin":false}
```
{% endtab %}
{% tab title="ZAP" %}
Utiliser `Encoder/Decoder/Hash` accessible via `Ctrl+E`. L'onglet `Decode` tente automatiquement le decodage avec plusieurs methodes. Des onglets personnalises peuvent etre crees avec le bouton `Add New Tab`.
{% endtab %}
{% endtabs %}

**Formats supportes :**

| Format | Usage courant |
|---|---|
| URL encoding | Parametres HTTP |
| Base64 | Cookies, tokens, donnees serializees |
| HTML entities | Contenu de pages web |
| Hex (ASCII) | Donnees binaires, obfuscation |
| Hashing (MD5, SHA) | Verification d'integrite, cookies |

{% hint style="warning" %}
Un cookie encode en Base64 qui contient `"is_admin":false` est un vecteur d'escalade de privileges. Le modifier en `"is_admin":true`, le re-encoder et le renvoyer peut suffire si le serveur ne signe pas ses cookies.
{% endhint %}

## Pieges et galeres

- Oublier l'encodage URL dans un payload casse la requete silencieusement. Le serveur recoit des donnees tronquees ou malformees.
- Certaines donnees sont encodees plusieurs fois (double URL encoding, Base64 d'un hex). Il faut decoder couche par couche pour identifier le format original.
- Le Repeater de Burp maintient l'historique des requetes envoyees dans chaque onglet. Utile pour revenir a une version anterieure d'un payload.

## Retour terrain

Le Repeater est l'outil qu'on utilise le plus en pentest web, devant l'interception. Il permet d'iterer rapidement sur un parametre sans quitter le proxy. Combine a l'encodeur, il couvre la majorite des tests manuels : injections, manipulation de cookies, bypass d'authentification. C'est l'equivalent d'un `curl` interactif avec gestion automatique des cookies et headers.

## Memo express

| Action | Burp | ZAP |
|---|---|---|
| Envoyer au Repeater | `Ctrl+R` | Clic droit > `Open/Resend with Request Editor` |
| Ouvrir le Repeater | `Ctrl+Shift+R` | Fenetre dediee |
| URL encoder | `Ctrl+U` (selection) | Automatique |
| Decoder | Onglet `Decoder` | `Ctrl+E` |
| Changer la methode | Clic droit > `Change Request Method` | Menu deroulant `Method` |
| Inspector | Panneau `Inspector` (Repeater/Proxy) | N/A |

***
