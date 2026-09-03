# Introduction à la reconnaissance web

La reconnaissance web constitue le socle de toute évaluation de sécurité sérieuse. Avant de chercher des vulnérabilités ou de tenter quoi que ce soit, il faut comprendre ce qu'on a en face. Cette phase, souvent sous-estimée, conditionne la qualité de tout ce qui suit.

## Pourquoi

Un pentest sans reconnaissance, c'est comme opérer à l'aveugle. La reconnaissance web permet de :

- **Cartographier les actifs exposés** : pages, sous-domaines, adresses IP, technologies utilisées. On veut une vue complète de la surface d'attaque.
- **Dénicher l'information cachée** : fichiers de sauvegarde oubliés, configurations exposées, documentation interne accessible. Ces éléments fournissent souvent des points d'entrée inattendus.
- **Évaluer la surface d'attaque** : quelles technologies sont en place, comment elles sont configurées, où se trouvent les faiblesses potentielles.
- **Collecter du renseignement exploitable** : contacts clés, adresses email, habitudes, tout ce qui peut servir pour la suite (y compris l'ingénierie sociale).

Côté offensif, ces informations permettent de calibrer les attaques. Côté défensif, elles permettent de colmater les brèches avant qu'un attaquant ne les exploite.

## Reconnaissance active vs passive

Deux approches fondamentales coexistent, chacune avec ses avantages et ses limites.

### Reconnaissance active

L'approche active implique une **interaction directe avec la cible**. On envoie des requêtes, on scanne, on sollicite les services. C'est plus complet mais aussi plus détectable.

| Technique | Description | Outils | Risque de détection |
|---|---|---|---|
| Scan de ports | Identifier les ports ouverts et services actifs | Nmap, Masscan | Élevé |
| Scan de vulnérabilités | Rechercher des failles connues (logiciels obsolètes, mauvaises configurations) | Nessus, OpenVAS, Nikto | Élevé |
| Cartographie réseau | Tracer la topologie réseau de la cible | Traceroute, Nmap | Moyen à élevé |
| Banner Grabbing | Récupérer les bannières d'identification des services | Netcat, curl | Faible |
| Fingerprinting OS | Déterminer le système d'exploitation de la cible | Nmap (`-O`), Xprobe2 | Faible |
| Énumération de services | Identifier les versions précises des services exposés | Nmap (`-sV`) | Faible |
| Web Spidering | Explorer automatiquement la structure d'un site web | Burp Suite Spider, OWASP ZAP, Scrapy | Faible à moyen |

### Reconnaissance passive

L'approche passive repose sur l'**analyse d'informations publiquement accessibles**, sans jamais toucher directement la cible. Plus discrète, mais potentiellement moins exhaustive.

| Technique | Description | Outils | Risque de détection |
|---|---|---|---|
| Requêtes moteurs de recherche | Exploiter Google, Shodan pour trouver des informations indexées | Google, DuckDuckGo, Shodan | Très faible |
| WHOIS | Obtenir les détails d'enregistrement d'un domaine | `whois`, services en ligne | Très faible |
| DNS | Analyser les enregistrements DNS pour cartographier l'infrastructure | `dig`, `nslookup`, `host`, dnsenum | Très faible |
| Archives web | Consulter les versions historiques d'un site | Wayback Machine | Très faible |
| OSINT réseaux sociaux | Collecter des informations sur LinkedIn, Twitter, GitHub | LinkedIn, Twitter, outils OSINT | Très faible |
| Dépôts de code | Chercher des credentials ou du code vulnérable dans les repos publics | GitHub, GitLab | Très faible |

{% hint style="info" %}
En pratique, les deux approches se complètent. On commence généralement par la reconnaissance passive pour minimiser le bruit, puis on affine avec des techniques actives une fois le périmètre mieux défini.
{% endhint %}

## Ce que couvre ce module

Ce module explore les techniques essentielles de reconnaissance web, en partant des fondamentaux (WHOIS, DNS) pour aller vers des méthodes plus avancées (crawling, fingerprinting, automatisation). Chaque technique est abordée avec son contexte d'utilisation et ses outils pratiques, dans un environnement Exegol.

***
