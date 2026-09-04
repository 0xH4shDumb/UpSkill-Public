# Configuration du proxy

## Pourquoi

Avant de pouvoir intercepter le trafic HTTP, le navigateur doit etre configure pour router ses requetes a travers le proxy. Cette etape inclut la configuration du proxy lui-meme, l'installation du certificat CA pour le HTTPS, et eventuellement l'utilisation d'extensions comme FoxyProxy pour basculer rapidement entre modes.

## Comment ca marche

Le proxy ecoute sur un port local (par defaut `127.0.0.1:8080`). Le navigateur est configure pour envoyer toutes ses requetes a cette adresse au lieu de contacter directement les serveurs. Le proxy recoit la requete, permet de l'inspecter ou de la modifier, puis la transmet au serveur reel.

Pour le HTTPS, le proxy genere un certificat a la volee pour chaque domaine visite. Le navigateur doit faire confiance au certificat CA du proxy, sinon il refusera les connexions ou affichera des erreurs de certificat a chaque requete.

## En pratique

### Methode rapide : navigateur pre-configure

Les deux outils integrent un navigateur pre-configure avec proxy et certificats deja en place.

{% tabs %}
{% tab title="Burp" %}
Dans `Proxy > Intercept`, cliquer sur `Open Browser`. Le navigateur Chromium integre route automatiquement tout le trafic a travers Burp.
{% endtab %}
{% tab title="ZAP" %}
Cliquer sur l'icone Firefox dans la barre d'outils. Le navigateur s'ouvre avec le proxy et les certificats pre-configures.
{% endtab %}
{% endtabs %}

### Methode manuelle : Firefox + FoxyProxy

Pour utiliser un navigateur reel, installer l'extension [FoxyProxy](https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/) dans Firefox.

**Configuration :**

1. Cliquer sur l'icone FoxyProxy, puis `Options`
2. Cliquer sur `Add` dans le panneau gauche
3. Renseigner : IP `127.0.0.1`, Port `8080`, Nom `Burp` ou `ZAP`
4. Sauvegarder, puis selectionner le profil dans le menu FoxyProxy

{% hint style="info" %}
Pour changer le port d'ecoute du proxy : dans Burp, `Proxy > Proxy settings > Proxy listeners`. Dans ZAP, `Tools > Options > Network > Local Servers/Proxies`. Le port configure dans Firefox doit correspondre.
{% endhint %}

### Installation du certificat CA

Sans le certificat CA, le trafic HTTPS sera bloque ou demandera une confirmation manuelle a chaque requete.

{% tabs %}
{% tab title="Burp" %}
1. Activer le proxy Burp dans FoxyProxy
2. Naviguer vers `http://burp`
3. Cliquer sur `CA Certificate` pour telecharger le certificat
{% endtab %}
{% tab title="ZAP" %}
1. Aller dans `Tools > Options > Network > Server Certificates`
2. Cliquer sur `Save` pour exporter le certificat
{% endtab %}
{% endtabs %}

**Import dans Firefox :**

1. Aller dans `about:preferences#privacy`
2. Descendre et cliquer sur `View Certificates`
3. Onglet `Authorities`, puis `Import`
4. Selectionner le certificat telecharge
5. Cocher `Trust this CA to identify websites` et `Trust this CA to identify email users`

## Pieges et galeres

- Oublier le certificat CA est la source numero un de problemes. Les requetes HTTPS echouent silencieusement ou generent des erreurs SSL en boucle.
- FoxyProxy doit etre desactive quand le proxy n'est pas lance, sinon le navigateur n'a plus d'acces internet.
- Certaines applications utilisent du certificate pinning, ce qui empeche le proxy d'intercepter leur trafic HTTPS meme avec le certificat CA installe.

## Retour terrain

En engagement, la configuration du proxy est la premiere etape. Elle prend deux minutes mais conditionne tout le reste. Un certificat CA mal installe peut faire perdre une heure de debug. Utiliser le navigateur pre-configure pour les tests rapides, et Firefox avec FoxyProxy pour les tests approfondis ou l'utilisation d'extensions specifiques.

## Memo express

| Etape | Action |
|---|---|
| Proxy rapide | Navigateur pre-configure (Burp/ZAP) |
| Proxy Firefox | FoxyProxy → `127.0.0.1:8080` |
| Certificat Burp | `http://burp` → `CA Certificate` |
| Certificat ZAP | `Tools > Options > Network > Server Certificates > Save` |
| Import Firefox | `about:preferences#privacy` → `View Certificates` → `Authorities` → `Import` |

***
