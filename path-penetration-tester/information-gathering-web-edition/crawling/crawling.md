# Crawling web

## Pourquoi

Le crawling (ou spidering) consiste à explorer automatiquement un site web en suivant les liens d'une page à l'autre, comme une araignée qui parcourt sa toile. Contrairement au fuzzing qui devine des chemins possibles, le crawling découvre les pages et ressources réellement liées entre elles. C'est une étape essentielle de la reconnaissance pour cartographier la structure d'un site et collecter des informations exploitables.

## Comment ça marche

### Le principe

Un crawler part d'une URL de départ (seed URL), télécharge la page, extrait tous les liens qu'elle contient, et ajoute ces liens à une file d'attente. Il visite ensuite chaque lien de la file, extrait de nouveaux liens, et ainsi de suite. Selon sa configuration, il peut explorer un site entier ou se limiter à certaines profondeurs.

### Stratégies de parcours

Deux stratégies principales existent :

**Breadth-first (largeur d'abord)** : le crawler explore tous les liens d'une page avant de descendre d'un niveau. Utile pour obtenir une vue d'ensemble de la structure du site.

```
Page de départ
├── Page A
├── Page B
└── Page C
    (puis on explore A, B, C avant d'aller plus loin)
```

**Depth-first (profondeur d'abord)** : le crawler suit un chemin de liens aussi loin que possible avant de revenir en arrière et d'explorer d'autres branches. Utile pour atteindre des contenus enfouis dans la structure du site.

```
Page de départ → Page A → Page D → Page G
(puis retour en arrière vers Page A → Page E, etc.)
```

### Ce que le crawling collecte

| Type de données | Valeur en reconnaissance |
|---|---|
| **Liens internes** | Cartographie de la structure du site, découverte de pages non référencées |
| **Liens externes** | Relations avec d'autres domaines, services tiers utilisés |
| **Commentaires HTML** | Parfois des informations sensibles, des TODO de développeurs, des credentials oubliés |
| **Métadonnées** | Titres, descriptions, auteurs, dates. Contexte sur le contenu et l'organisation |
| **Fichiers sensibles** | Sauvegardes (`.bak`, `.old`), configurations (`web.config`), logs, fichiers avec credentials ou clés API |
| **Emails** | Adresses email exploitables pour le phishing ou l'énumération d'utilisateurs |
| **Fichiers JS** | Scripts pouvant contenir des endpoints API, des tokens, des chemins internes |
| **Formulaires** | Champs de saisie potentiellement vulnérables (injection, XSS) |

{% hint style="warning" %}
L'analyse contextuelle est essentielle. Un lien vers un répertoire `/files/` peut sembler anodin isolément, mais combiné à un commentaire mentionnant un "file server", il peut révéler un directory listing exposant des documents sensibles.
{% endhint %}

## En pratique

### Outils de crawling

| Outil | Description |
|---|---|
| **Burp Suite Spider** | Crawler intégré à Burp Suite, idéal pour cartographier les applications web |
| **OWASP ZAP Spider** | Scanner open-source avec composant de spidering |
| **Scrapy** | Framework Python puissant et modulable pour créer des spiders personnalisés |
| **Apache Nutch** | Crawler open-source Java, adapté aux crawls à grande échelle |

### Crawling avec Scrapy et ReconSpider

Scrapy est un framework Python qui permet de construire des spiders sur mesure. `ReconSpider` est un spider préconstruit orienté reconnaissance web :

```bash
# - Installation de Scrapy
pip3 install scrapy

# - Lancement de ReconSpider
python3 ReconSpider.py http://<DOMAINE_CIBLE>
```

Les résultats sont enregistrés dans un fichier `results.json` structuré par catégorie :

```json
{
  "emails": ["contact@cible.com"],
  "links": ["https://cible.com/page1", "https://cible.com/page2"],
  "external_files": ["https://cible.com/uploads/rapport.pdf"],
  "js_files": ["https://cible.com/assets/app.min.js"],
  "form_fields": [],
  "images": [],
  "videos": [],
  "audio": [],
  "comments": ["<!-- TODO: changer les credentials -->"]
}
```

| Clé JSON | Ce qu'elle contient |
|---|---|
| `emails` | Adresses email trouvées dans les pages |
| `links` | URLs internes et externes découvertes |
| `external_files` | Fichiers téléchargeables (PDF, DOCX, etc.) |
| `js_files` | Scripts JavaScript chargés par les pages |
| `form_fields` | Champs de formulaire identifiés |
| `comments` | Commentaires HTML extraits du code source |

{% hint style="danger" %}
Les commentaires HTML sont une mine d'or souvent négligée. Les développeurs y laissent régulièrement des notes contenant des endpoints internes, des credentials temporaires, des clés API ou des chemins de migration.
{% endhint %}

## Pièges et galères

- **Respect du périmètre** : configurer le crawler pour rester dans le scope autorisé. Un crawler mal configuré peut sortir du domaine cible et scanner des sites tiers.
- **Surcharge serveur** : le crawling génère beaucoup de requêtes. Respecter les directives de `robots.txt` et configurer un délai entre les requêtes pour ne pas surcharger le serveur.
- **Contenu dynamique** : les sites en SPA (Single Page Application) chargent le contenu via JavaScript. Un crawler classique ne verra que le squelette HTML initial. Des outils comme Burp Suite ou des headless browsers (Puppeteer, Playwright) gèrent mieux ce cas.
- **Boucles infinies** : certains sites génèrent des URLs dynamiques qui créent des boucles. Limiter la profondeur de crawling et le nombre de pages visitées.

## Mémo express

| Besoin | Commande |
|---|---|
| Crawler avec ReconSpider | `python3 ReconSpider.py http://<cible>` |
| Lire les résultats | `cat results.json \| jq .` |
| Filtrer les emails | `cat results.json \| jq '.emails'` |
| Filtrer les commentaires | `cat results.json \| jq '.comments'` |

***
