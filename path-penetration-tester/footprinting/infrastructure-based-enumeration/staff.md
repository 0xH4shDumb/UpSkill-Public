# Personnel et réseaux sociaux

Les employés d'une entreprise publient involontairement des informations techniques précieuses sur les réseaux sociaux professionnels. Analyser ces données fait partie intégrante de la reconnaissance passive et peut révéler des pans entiers de l'infrastructure.

## Pourquoi

Les offres d'emploi et les profils LinkedIn des employés techniques d'une entreprise dessinent une carte très précise de sa stack technologique. Un poste qui demande de l'expérience en Django, PostgreSQL et Docker dit clairement ce qui tourne en production. Un profil qui mentionne Atlassian, Kafka et Elasticsearch confirme les intégrations internes. Ce sont des informations qu'aucun scan réseau ne peut fournir.

## Comment ça marche

### Offres d'emploi

Les offres d'emploi techniques listent explicitement les technologies utilisées. En croisant plusieurs offres du même département, on peut reconstituer la stack complète : langages, frameworks, bases de données, outils de CI/CD, services cloud.

Un poste qui demande de l'expérience avec Flask, Django, PostgreSQL, Redis et Docker permet de déduire que l'entreprise utilise des applications Python conteneurisées avec du cache Redis et une base PostgreSQL. Ce n'est pas de la spéculation, c'est de la documentation publique.

{% hint style="info" %}
Les plateformes de recherche d'emploi (LinkedIn, Indeed, Glassdoor) sont des sources de renseignement technique sous-estimées. Les offres archivées, accessibles via des caches Google ou la Wayback Machine, peuvent révéler des technologies qui ne sont plus mentionnées dans les offres actuelles mais qui tournent encore en production.
{% endhint %}

### Profils d'employés

Les profils LinkedIn des développeurs et des administrateurs systèmes sont particulièrement révélateurs :

- **Compétences listées** : les technologies maitrisées par l'équipe correspondent presque toujours à ce qui est déployé en interne
- **Projets partagés** : des liens vers des dépôts GitHub, des articles de blog ou des présentations techniques peuvent contenir du code, des configurations ou des architectures internes
- **Historique de carrière** : les postes précédents au sein de l'entreprise indiquent l'évolution de la stack technique

{% hint style="danger" %}
Les développeurs publient parfois du code personnel sur GitHub qui contient des artefacts de leur environnement professionnel : des URLs internes, des tokens d'API, des fichiers de configuration, voire des clés privées. C'est un vecteur d'attaque réel et documenté.
{% endhint %}

### Réseaux sociaux et contributions publiques

Au-delà de LinkedIn :

| Source | Ce qu'on y trouve |
|---|---|
| GitHub / GitLab | Code, configurations, dépendances, tokens accidentels |
| Twitter / X | Partages techniques, conférences, outils utilisés |
| Stack Overflow | Questions techniques qui exposent des détails d'infrastructure |
| Conférences (slides, vidéos) | Architecture interne, choix techniques, retours d'expérience |

## En pratique

1. **Recherche LinkedIn** : filtrer par entreprise, département technique, rôle (sécurité, infrastructure, développement)
2. **Analyse des offres** : collecter les technologies mentionnées dans les offres actuelles et archivées
3. **Profils GitHub** : identifier les développeurs de l'entreprise et analyser leurs dépôts publics
4. **Recoupement** : croiser les informations pour valider les hypothèses sur la stack technique

{% hint style="warning" %}
La collecte d'informations sur les employés doit rester dans le cadre légal et éthique de l'engagement. On analyse des données publiques, on ne contacte pas les employés et on ne se fait pas passer pour un recruteur ou un collègue. L'ingénierie sociale sort du périmètre du footprinting passif.
{% endhint %}

## Retour terrain

En mission, l'analyse du personnel est souvent ce qui fait la différence entre un rapport générique et un rapport actionnable. Savoir que l'entreprise utilise Atlassian permet de chercher des instances Jira/Confluence exposées. Savoir qu'un développeur publie du code React avec une API interne permet de cartographier les endpoints. Savoir que l'équipe sécurité utilise un outil spécifique permet d'adapter sa stratégie d'évasion.

Le point le plus critique reste les dépôts GitHub publics des employés. Un `package.json` avec une URL de registry privée, un `.env.example` avec des noms de variables révélateurs, un fichier de configuration avec un JWT en dur : ces artefacts sont régulièrement présents et rarement nettoyés.

***
