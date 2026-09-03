# Recherche via moteurs de recherche

## Pourquoi

Les moteurs de recherche indexent en permanence une portion massive du web. En exploitant des opérateurs de recherche avancés, on peut extraire des informations qui ne sont pas directement visibles sur les sites : pages de connexion cachées, fichiers de configuration exposés, documents sensibles indexés par erreur. Cette technique, souvent appelée Google Dorking ou OSINT par moteur de recherche, est entièrement passive, gratuite et ne nécessite aucune interaction directe avec la cible.

## Comment ça marche

Les moteurs de recherche offrent des opérateurs spéciaux qui permettent de filtrer et d'affiner les résultats avec une grande précision. Ces opérateurs transforment une recherche banale en outil de reconnaissance puissant.

### Opérateurs essentiels

| Opérateur | Rôle | Exemple |
|---|---|---|
| `site:` | Limite les résultats à un domaine spécifique | `site:exemple.com` |
| `inurl:` | Cherche un terme dans l'URL | `inurl:admin` |
| `intitle:` | Cherche un terme dans le titre de la page | `intitle:"index of"` |
| `filetype:` | Filtre par type de fichier | `filetype:pdf` |
| `intext:` | Cherche un terme dans le corps de la page | `intext:"mot de passe"` |
| `cache:` | Affiche la version en cache d'une page | `cache:exemple.com` |
| `link:` | Trouve les pages qui pointent vers une URL | `link:exemple.com` |
| `related:` | Trouve des sites similaires | `related:exemple.com` |

### Opérateurs de combinaison

| Opérateur | Rôle | Exemple |
|---|---|---|
| `AND` | Les deux termes doivent être présents | `site:exemple.com AND inurl:admin` |
| `OR` | Au moins un des termes doit être présent | `"linux" OR "ubuntu"` |
| `NOT` ou `-` | Exclut un terme des résultats | `site:exemple.com -inurl:blog` |
| `" "` | Recherche une expression exacte | `"internal use only"` |
| `*` | Wildcard (joker) | `site:exemple.com filetype:pdf user* manual` |
| `..` | Recherche dans une plage numérique | `"prix" 100..500` |

## En pratique

### Google Dorking pour la reconnaissance

Le Google Dorking combine ces opérateurs pour des recherches ciblées en reconnaissance. Voici les catégories de requêtes les plus utiles :

{% tabs %}
{% tab title="Pages de connexion" %}
```
site:exemple.com inurl:login
site:exemple.com (inurl:login OR inurl:admin)
site:exemple.com intitle:"admin panel"
```
{% endtab %}

{% tab title="Fichiers exposés" %}
```
site:exemple.com filetype:pdf
site:exemple.com (filetype:xls OR filetype:docx)
site:exemple.com filetype:sql
```
{% endtab %}

{% tab title="Configurations" %}
```
site:exemple.com inurl:config.php
site:exemple.com (ext:conf OR ext:cnf)
site:exemple.com filetype:env
```
{% endtab %}

{% tab title="Sauvegardes" %}
```
site:exemple.com inurl:backup
site:exemple.com (filetype:bak OR filetype:old)
site:exemple.com intitle:"index of" backup
```
{% endtab %}
{% endtabs %}

### Google Hacking Database

La Google Hacking Database (GHDB), maintenue sur Exploit-DB, est une collection de dorks pré-construits classés par catégorie. C'est une ressource de référence pour découvrir de nouvelles techniques de recherche.

### Au-delà de Google

D'autres moteurs de recherche offrent des capacités complémentaires :

| Moteur | Spécialité |
|---|---|
| **Shodan** | Indexe les services et appareils connectés à Internet (ports, bannières, certificats) |
| **Censys** | Scan de l'espace IPv4, certificats, services exposés |
| **DuckDuckGo** | Recherche web avec plus de respect de la vie privée |
| **Bing** | Opérateurs similaires à Google, parfois des résultats différents |

## Pièges et galères

- **Résultats obsolètes** : les pages indexées par Google peuvent avoir été modifiées ou supprimées depuis l'indexation. La version en cache (`cache:`) permet de vérifier.
- **Rate limiting** : Google peut bloquer temporairement les requêtes si on enchaîne trop de dorks en peu de temps. Espacer les recherches ou utiliser différents moteurs.
- **Informations partielles** : les moteurs de recherche n'indexent pas tout. Les pages protégées par authentification, les contenus en noindex, et les réseaux internes échappent à l'indexation.
- **Cadre légal** : la recherche d'informations via Google est légale tant qu'on ne tente pas d'accéder à des systèmes sans autorisation. Les informations trouvées sont publiques, mais leur exploitation sans mandat peut poser problème.

## Mémo express

| Besoin | Requête |
|---|---|
| Pages d'un domaine | `site:exemple.com` |
| Pages de login | `site:exemple.com inurl:login` |
| PDFs exposés | `site:exemple.com filetype:pdf` |
| Fichiers de config | `site:exemple.com (ext:conf OR ext:ini OR ext:env)` |
| Directory listing | `site:exemple.com intitle:"index of"` |

***
