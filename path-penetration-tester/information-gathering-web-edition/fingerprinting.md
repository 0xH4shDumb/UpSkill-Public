# Fingerprinting

## Pourquoi

Le fingerprinting consiste à identifier les technologies qui font tourner un site ou une application web : serveur HTTP, langage backend, CMS, frameworks, WAF. Connaître précisément la stack technique d'une cible permet de chercher des vulnérabilités connues pour ces versions spécifiques, de repérer des mauvaises configurations, et de prioriser les vecteurs d'attaque.

## Comment ça marche

### Techniques de fingerprinting

| Technique | Ce qu'elle révèle |
|---|---|
| **Banner Grabbing** | Les bannières d'identification des services (nom et version du serveur HTTP) |
| **Analyse des headers HTTP** | Le header `Server` indique le serveur web, `X-Powered-By` le langage ou framework |
| **Requêtes spécifiques** | Certaines réponses d'erreur ou chemins par défaut trahissent un CMS ou framework |
| **Analyse du code source** | Le HTML, les scripts JS, les commentaires et les chemins de fichiers révèlent les technologies |

### Ce qu'on cherche

- **Serveur web et version** : Apache 2.4.41, Nginx 1.26.1, IIS 10.0. Chaque version a ses CVE connues.
- **CMS** : WordPress, Joomla, Drupal. Les CMS ont des chemins caractéristiques (`/wp-admin/`, `/administrator/`, etc.).
- **Frameworks** : Laravel, Django, Spring. Souvent trahis par les cookies de session ou les headers spécifiques.
- **WAF** : Wordfence, Cloudflare, ModSecurity. La présence d'un WAF change la stratégie d'attaque.

## En pratique

### Banner Grabbing avec curl

La méthode la plus simple pour identifier un serveur web :

```bash
# - Récupérer les headers HTTP
curl -I http://<IP_CIBLE>
```

Les headers intéressants dans la réponse :

- `Server: Apache/2.4.41 (Ubuntu)` : serveur web et OS.
- `X-Powered-By: PHP/7.4` : langage backend.
- `X-Redirect-By: WordPress` : CMS utilisé.
- `Set-Cookie: PHPSESSID=...` : confirme l'utilisation de PHP.

{% hint style="info" %}
Certains serveurs masquent ou modifient leurs headers pour compliquer le fingerprinting. Un `Server: nginx` sans version est courant sur les configurations durcies.
{% endhint %}

### Détection de WAF avec wafw00f

`wafw00f` envoie des requêtes spécifiques et analyse les réponses pour identifier le Web Application Firewall en place :

```bash
# - Installation
pip3 install wafw00f

# - Détection de WAF
wafw00f <DOMAINE_CIBLE>
```

Le résultat indique si un WAF est détecté et lequel. Cette information est cruciale : un WAF actif peut bloquer les scans agressifs et nécessite d'adapter l'approche.

### Scan avec Nikto

Nikto est un scanner web qui combine fingerprinting et détection de vulnérabilités basiques :

```bash
nikto -h <DOMAINE_CIBLE> -Tuning b
```

Nikto identifie :

- Le serveur web et sa version.
- Les CMS et leurs fichiers caractéristiques.
- Les headers de sécurité manquants (HSTS, X-Content-Type-Options, etc.).
- Les fichiers exposés par inadvertance (`license.txt`, `readme.html`, etc.).

### WhatWeb : fingerprinting large spectre

`WhatWeb` reconnaît un grand nombre de technologies web en analysant les headers, le HTML, les cookies et les scripts :

```bash
whatweb <DOMAINE_CIBLE>
```

### Wappalyzer

Extension navigateur (disponible aussi en ligne) qui identifie automatiquement les technologies d'un site lors de la navigation. Pratique pour une reconnaissance rapide sans ouvrir le terminal.

### Tableau récapitulatif des outils

| Outil | Type | Points forts |
|---|---|---|
| `curl -I` | CLI | Rapide, disponible partout, headers bruts |
| `wafw00f` | CLI | Spécialisé détection WAF |
| `Nikto` | CLI | Fingerprinting + vulnérabilités basiques |
| `WhatWeb` | CLI | Large spectre de technologies reconnues |
| `Wappalyzer` | Extension navigateur | Identification visuelle en temps réel |
| `BuiltWith` | Service en ligne | Rapports détaillés sur l'infrastructure |
| `Netcraft` | Service en ligne | Technologie, hébergeur et historique de sécurité |

## Pièges et galères

- **Headers supprimés ou falsifiés** : les serveurs bien configurés masquent leurs bannières. Ne pas se fier uniquement aux headers, recouper avec d'autres techniques.
- **CDN et reverse proxy** : Cloudflare, Akamai et autres CDN masquent le serveur d'origine. Le fingerprinting révèle le CDN, pas forcément le serveur backend.
- **Scan agressif vs WAF** : lancer Nikto avec toutes les options contre un site protégé par un WAF peut déclencher un blocage IP. Commencer doucement et adapter.

## Mémo express

| Besoin | Commande |
|---|---|
| Headers HTTP | `curl -I http://<cible>` |
| Détection WAF | `wafw00f <cible>` |
| Scan Nikto | `nikto -h <cible>` |
| Fingerprinting large | `whatweb <cible>` |

***
