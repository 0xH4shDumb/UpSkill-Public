# Introduction aux inclusions de fichiers

Les applications web modernes s'appuient sur des moteurs de templates pour charger dynamiquement le contenu de leurs pages. Le principe est simple : un squelette commun (en-tête, barre de navigation, pied de page) est partagé entre toutes les pages, et seul le contenu central change en fonction d'un paramètre utilisateur, par exemple `index.php?page=about`. Si ce paramètre est utilisé tel quel dans une fonction d'inclusion sans validation ni filtrage, un attaquant peut forcer le chargement d'un fichier arbitraire. C'est le fondement des vulnérabilités d'inclusion de fichiers.

## Pourquoi s'y intéresser

L'inclusion de fichiers fait partie des vulnérabilités web les plus répandues et les plus sous-estimées. Son impact varie selon le contexte, mais il peut aller très loin :

- **Divulgation de code source** : lire les fichiers PHP, les configurations, les clés d'API. Un accès au code source permet souvent de découvrir d'autres vulnérabilités (SQLi, hardcoded credentials, etc.).
- **Exposition de données sensibles** : lecture de `/etc/passwd`, de fichiers de configuration contenant des mots de passe de base de données, de clés SSH privées.
- **Exécution de code à distance (RCE)** : sous certaines conditions, l'inclusion permet d'exécuter du code arbitraire sur le serveur, ce qui compromet l'intégralité du système.

{% hint style="danger" %}
Une vulnérabilité d'inclusion de fichiers, même limitée à la lecture de fichiers locaux, peut conduire à une compromission complète du serveur si elle permet de récupérer des credentials ou des clés SSH.
{% endhint %}

## Comment ça marche

### Le mécanisme de base

Le scénario classique repose sur un paramètre GET qui détermine quel fichier charger. Prenons une application multilingue : l'URL `index.php?language=fr` charge le fichier `fr.php`, tandis que `index.php?language=en` charge `en.php`. Si le développeur passe directement la valeur du paramètre à une fonction d'inclusion, rien n'empêche un attaquant de remplacer `fr` par un chemin vers un fichier sensible du système.

On distingue deux grandes catégories :

| Type | Description |
|------|-------------|
| **LFI (Local File Inclusion)** | Inclusion d'un fichier présent sur le serveur lui-même. Le paramètre contrôlé pointe vers un chemin local (`/etc/passwd`, un fichier de configuration, des logs). |
| **RFI (Remote File Inclusion)** | Inclusion d'un fichier hébergé sur un serveur distant. L'attaquant fournit une URL externe (HTTP, FTP, SMB) qui pointe vers un script malveillant. Plus rare car souvent désactivé par défaut dans les configurations serveur. |

{% hint style="info" %}
Toute vulnérabilité RFI est aussi une LFI (si on peut inclure un fichier distant, on peut a fortiori inclure un fichier local). L'inverse n'est pas vrai : une LFI ne permet pas forcément l'inclusion distante.
{% endhint %}

### Exemples de code vulnérable

La vulnérabilité se retrouve dans la plupart des langages et frameworks. Voici les schémas les plus courants.

**PHP**

Le cas le plus fréquent. La fonction `include()` charge et exécute le fichier spécifié :

```php
if (isset($_GET['language'])) {
    include($_GET['language']);
}
```

Le paramètre `language` est injecté directement dans `include()`. Un attaquant peut fournir n'importe quel chemin. D'autres fonctions PHP présentent le même risque : `include_once()`, `require()`, `require_once()` et `file_get_contents()`. La différence principale est que `file_get_contents()` lit le contenu sans l'exécuter, tandis que les fonctions `include`/`require` exécutent le code PHP contenu dans le fichier.

**NodeJS**

Avec le module `fs` natif, la lecture de fichiers dépend du paramètre de requête :

```javascript
if (req.query.language) {
    fs.readFile(path.join(__dirname, req.query.language), function (err, data) {
        res.write(data);
    });
}
```

Avec Express.js et la fonction `render()`, le paramètre peut provenir directement du chemin d'URL :

```javascript
app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
});
```

Ici, `/about/en` charge la version anglaise, mais un attaquant peut manipuler le segment de chemin pour naviguer dans l'arborescence du serveur.

**Java**

Les applications Java (JSP) utilisent des directives d'inclusion similaires :

```java
<c:if test="${not empty param.language}">
    <jsp:include file="<%= request.getParameter('language') %>" />
</c:if>
```

La directive `import` offre des capacités supplémentaires, notamment l'inclusion de ressources distantes :

```java
<c:import url="<%= request.getParameter('language') %>"/>
```

**.NET**

En ASP.NET, plusieurs méthodes permettent le chargement dynamique de fichiers :

```cs
@if (!string.IsNullOrEmpty(HttpContext.Request.Query['language'])) {
    <% Response.WriteFile("<% HttpContext.Request.Query['language'] %>"); %>
}
```

La fonction `@Html.Partial()` rend un fichier comme fragment de template :

```cs
@Html.Partial(HttpContext.Request.Query['language'])
```

Et la directive `include` côté serveur peut à la fois lire et exécuter des fichiers :

```cs
<!--#include file="<% HttpContext.Request.Query['language'] %>"-->
```

### Lecture vs exécution

Un point fondamental à comprendre : certaines fonctions ne font que lire le contenu d'un fichier, d'autres l'exécutent comme du code. De plus, certaines acceptent des URLs distantes, d'autres non.

| Fonction | Lecture | Exécution | URL distante |
|----------|:-------:|:---------:|:------------:|
| **PHP** | | | |
| `include()` / `include_once()` | Oui | Oui | Oui |
| `require()` / `require_once()` | Oui | Oui | Non |
| `file_get_contents()` | Oui | Non | Oui |
| `fopen()` / `file()` | Oui | Non | Non |
| **NodeJS** | | | |
| `fs.readFile()` | Oui | Non | Non |
| `fs.sendFile()` | Oui | Non | Non |
| `res.render()` | Oui | Oui | Non |
| **Java** | | | |
| `include` | Oui | Non | Non |
| `import` | Oui | Oui | Oui |
| **.NET** | | | |
| `@Html.Partial()` | Oui | Non | Non |
| `@Html.RemotePartial()` | Oui | Non | Oui |
| `Response.WriteFile()` | Oui | Non | Non |
| `include` | Oui | Oui | Oui |

{% hint style="warning" %}
Les fonctions qui exécutent le fichier sont les plus dangereuses : elles permettent potentiellement l'exécution de code à distance. Les fonctions en lecture seule restent exploitables pour la divulgation de code source et de données sensibles.
{% endhint %}

## En pratique

Sur Exegol, on peut tester rapidement une LFI en manipulant le paramètre vulnérable :

```bash
# - Test basique de LFI
curl "http://<IP_CIBLE>:<PORT>/index.php?language=/etc/passwd"

# - Avec traversée de répertoire si le paramètre est préfixé par un chemin
curl "http://<IP_CIBLE>:<PORT>/index.php?language=../../../../etc/passwd"

# - Lecture de fichier de configuration PHP via un filtre base64
curl "http://<IP_CIBLE>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config"
```

La première étape est toujours de vérifier si le paramètre est vulnérable en tentant de lire un fichier connu comme `/etc/passwd` (Linux) ou `C:\Windows\win.ini` (Windows). Si le contenu du fichier apparait dans la réponse, la vulnérabilité est confirmée.

{% hint style="success" %}
En audit, commencer par identifier les paramètres qui chargent du contenu dynamique (souvent `page`, `file`, `language`, `template`, `view`, `include`). Tester chacun d'eux pour l'inclusion de fichiers locaux avant de passer aux techniques avancées.
{% endhint %}

## Pièges et galères

- **Le fichier est inclus mais pas affiché** : certaines fonctions exécutent le fichier au lieu de l'afficher. Un fichier PHP inclus via `include()` sera exécuté, pas lu. Pour obtenir le code source, il faut utiliser des filtres PHP (comme `convert.base64-encode`).
- **Confusion entre LFI et RFI** : on peut avoir une LFI exploitable sans que la RFI fonctionne. La configuration `allow_url_include` doit être activée pour la RFI en PHP, ce qui est désactivé par défaut.
- **Erreurs silencieuses** : certaines applications masquent les erreurs PHP en production. L'absence de message d'erreur ne signifie pas que le paramètre n'est pas vulnérable. Il faut observer les changements de taille de réponse ou de comportement.
- **Faux sentiment de sécurité avec les extensions** : ajouter `.php` au paramètre (`include($_GET['page'] . ".php")`) ne protège pas complètement. Des techniques de contournement existent (filtres PHP, null bytes sur d'anciennes versions, troncature de chemin).

## Retour terrain

En pentest, les inclusions de fichiers se trouvent souvent dans des endroits inattendus : un sélecteur de thème, un chargeur de langue, un système de templates personnalisé, ou même un paramètre de téléchargement d'avatar. Les paramètres exposés mais non liés à des formulaires HTML sont souvent les plus vulnérables, car ils reçoivent moins d'attention lors du développement.

La démarche typique en engagement :

1. Identifier tous les paramètres qui influencent le contenu affiché
2. Tester chacun pour la LFI avec des chemins classiques
3. Si un filtre bloque, tenter les contournements (encodage, traversée récursive)
4. Lire le code source pour comprendre les filtres en place et adapter les payloads
5. Escalader vers l'exécution de code si possible (wrappers PHP, log poisoning, upload + LFI)

{% hint style="info" %}
Les fonctions vulnérables ne se limitent pas à celles listées ici. Dans un audit en boite blanche, il faut chercher toute fonction qui lit ou charge un fichier à partir d'une entrée utilisateur, quel que soit le langage.
{% endhint %}

## Mémo express

| Concept | Détail |
|---------|--------|
| **LFI** | Inclusion de fichier local via un paramètre contrôlé |
| **RFI** | Inclusion de fichier distant (requiert `allow_url_include` en PHP) |
| **Fonctions PHP dangereuses** | `include()`, `include_once()`, `require()`, `require_once()` |
| **Lecture seule PHP** | `file_get_contents()`, `fopen()`, `file()` |
| **Test rapide** | `?param=../../../../etc/passwd` |
| **Code source PHP** | `php://filter/read=convert.base64-encode/resource=fichier` |
| **Toute RFI = LFI** | Mais l'inverse n'est pas vrai |
| **Impact maximal** | Lecture de credentials, clés SSH, puis RCE via log poisoning ou wrappers |

***
