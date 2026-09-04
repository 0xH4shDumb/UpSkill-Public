# Scanners de vulnerabilites

## Pourquoi

Les scanners integres aux proxys web automatisent la detection de vulnerabilites en combinant crawling (cartographie de l'application) et tests actifs/passifs. Ils identifient des problemes comme les injections de commandes, les XSS, les headers de securite manquants, et produisent des rapports exploitables. C'est un complement indispensable aux tests manuels.

## Comment ca marche

Le scan se deroule en plusieurs phases :

1. **Crawling/Spider** : navigation automatique sur toutes les pages accessibles pour construire la cartographie du site
2. **Scan passif** : analyse du code source et des reponses deja recues pour detecter des problemes potentiels (headers manquants, DOM XSS, cookies non securises)
3. **Scan actif** : envoi de payloads de test (injections, XSS, traversals) sur chaque parametre identifie pour confirmer les vulnerabilites

### Differences entre Burp Scanner et ZAP Scanner

| Aspect | Burp Scanner | ZAP Scanner |
|---|---|---|
| Disponibilite | Pro uniquement | Gratuit |
| Crawler | Burp Crawler | ZAP Spider + Ajax Spider |
| Scan passif | Inclus | Automatique pendant le Spider |
| Scan actif | Inclus | Inclus |
| Rapports | HTML/XML personnalisable | HTML/XML/Markdown |

## En pratique

### Definir le scope

Avant de scanner, definir le perimetre pour eviter de tester des domaines hors scope.

{% tabs %}
{% tab title="Burp" %}
Dans `Target > Site map`, clic droit sur la cible, puis `Add to scope`. Burp propose de restreindre ses fonctionnalites aux elements in-scope.

Pour exclure des endpoints (comme une page de deconnexion), clic droit > `Remove from scope`. Visualiser et ajuster le scope dans `Target > Scope`.
{% endtab %}
{% tab title="ZAP" %}
Lors du premier Spider, ZAP propose d'ajouter automatiquement le site au scope. Le scope est egalement configurable manuellement via les proprietes du site.
{% endtab %}
{% endtabs %}

### Crawling

{% tabs %}
{% tab title="Burp" %}
Dans `Dashboard`, cliquer sur `New Scan`, selectionner `Crawl`, configurer la strategie (utiliser `Crawl strategy - fastest` pour les tests rapides). Le crawler suit les liens et remplit le `Target > Site map`.

L'onglet `Application login` permet de fournir des identifiants pour crawler les zones authentifiees.
{% endtab %}
{% tab title="ZAP" %}
**Spider classique** : clic droit sur la cible dans l'historique > `Attack > Spider`, ou bouton Spider dans le HUD. Suit les liens HTML classiques.

**Ajax Spider** : utilise un navigateur headless pour executer le JavaScript et decouvrir les liens charges dynamiquement (AJAX). Plus lent mais plus complet. A lancer apres le Spider classique.
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Le crawler ne fait pas de fuzzing de repertoires. Il suit uniquement les liens presents dans le code. Pour decouvrir des pages non referencees, utiliser Intruder/Fuzzer ou un outil dedié comme ffuf.
{% endhint %}

### Scan passif

{% tabs %}
{% tab title="Burp" %}
Clic droit sur la cible dans le Site map > `Do passive scan`. Les resultats apparaissent dans `Dashboard > Issue activity`. Le scan passif identifie des problemes comme :
- Cookies sans flag HttpOnly ou Secure
- Headers de securite manquants (X-Frame-Options, CSP)
- Divulgation d'informations dans les reponses
{% endtab %}
{% tab title="ZAP" %}
Le scan passif s'execute automatiquement sur chaque reponse recue (pendant le Spider ou la navigation manuelle). Les alertes sont visibles dans l'onglet `Alerts` de l'interface principale.
{% endtab %}
{% endtabs %}

### Scan actif

{% tabs %}
{% tab title="Burp" %}
Dans `Dashboard > New Scan`, selectionner `Crawl and Audit`. Configurer l'audit avec une preset (`Audit checks - critical issues only` pour cibler les failles graves). Le scanner teste activement chaque parametre avec des payloads d'injection.

Suivre la progression dans `Dashboard > Tasks`. Les requetes envoyees sont visibles dans le `Logger`.
{% endtab %}
{% tab title="ZAP" %}
Clic droit sur la cible > `Attack > Active Scan`, ou bouton Active Scan dans le HUD. ZAP lance automatiquement un Spider si le site n'a pas encore ete cartographie.

Les alertes sont classees par severite (High, Medium, Low, Informational). Cliquer sur une alerte pour voir le detail, la requete envoyee et la preuve d'exploitation.
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Un scan actif envoie des payloads d'attaque reels. Ne jamais lancer un scan actif sur un systeme sans autorisation ecrite. En lab, c'est libre. En engagement, verifier que le scope contractuel autorise le scan automatise.
{% endhint %}

### Rapports

{% tabs %}
{% tab title="Burp" %}
Dans `Target > Site map`, clic droit sur la cible > `Issue > Report issues for this host`. Choisir le format (HTML/XML) et les niveaux de severite a inclure. Le rapport contient les details de chaque vulnerabilite, les preuves et les recommandations de remediation.
{% endtab %}
{% tab title="ZAP" %}
`Report > Generate HTML Report` (ou XML/Markdown). Le rapport liste toutes les alertes identifiees avec leur severite, leur description et les requetes de preuve.
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Les rapports generes par les outils ne remplacent jamais un rapport de pentest redige. Ils servent de donnees complementaires en annexe ou de base de travail pour la remediation.
{% endhint %}

## Pieges et galeres

- Les scans actifs generent beaucoup de trafic et peuvent etre longs (plusieurs heures sur une grosse application). Limiter le scope aux endpoints pertinents.
- Un scan actif peut provoquer des effets de bord : creation de comptes, envoi d'emails, modification de donnees. Identifier et exclure les endpoints sensibles avant de lancer le scan.
- Les faux positifs sont frequents, surtout en severite Low/Medium. Toujours verifier manuellement les resultats avant de les inclure dans un rapport.
- Le Spider ne detecte pas les pages protegees par authentification sauf si des identifiants sont fournis.

## Retour terrain

Le scanner du proxy est un filet de securite : il attrape les vulnerabilites evidentes qu'on pourrait manquer en test manuel. En engagement, on commence par un scan passif et un crawl pendant qu'on explore manuellement l'application. Le scan actif vient en complement, cible sur les endpoints les plus prometteurs. Ne jamais se fier uniquement aux resultats automatises : les vulnerabilites logiques et les chaines d'exploitation complexes echappent systematiquement aux scanners.

## Memo express

| Phase | Burp (Pro) | ZAP (gratuit) |
|---|---|---|
| Crawl | `Dashboard > New Scan > Crawl` | `Attack > Spider` + Ajax Spider |
| Scan passif | `Do passive scan` (automatique) | Automatique sur chaque reponse |
| Scan actif | `New Scan > Crawl and Audit` | `Attack > Active Scan` |
| Rapport | `Issue > Report issues for this host` | `Report > Generate HTML Report` |
| Scope | `Target > Scope` | Ajout automatique au premier Spider |

***
