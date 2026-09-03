# Archives web

## Pourquoi

Les sites web évoluent en permanence : des pages apparaissent, changent, disparaissent. Le Wayback Machine de l'Internet Archive capture et conserve des instantanés de millions de sites depuis 1996. En reconnaissance web, c'est un outil de reconnaissance passive qui permet de remonter dans le temps pour analyser d'anciennes versions d'un site, sans jamais interagir avec les serveurs de la cible.

## Comment ça marche

### Le Wayback Machine

Le Wayback Machine fonctionne en trois étapes :

1. **Crawling** : des robots parcourent le web en suivant les liens, comme les crawlers des moteurs de recherche, et téléchargent les pages visitées.
2. **Archivage** : chaque page capturée est stockée avec tous ses fichiers associés (HTML, CSS, images, scripts). Chaque capture est horodatée pour créer un instantané fidèle du site à ce moment précis.
3. **Consultation** : en saisissant une URL sur le Wayback Machine, on peut naviguer entre les différentes versions d'un site selon les dates de capture disponibles.

{% hint style="info" %}
La fréquence de capture varie selon la popularité du site, sa fréquence de mise à jour et l'intérêt que lui portent les contributeurs de l'Internet Archive. Un site très visité peut avoir des captures quotidiennes, tandis qu'un site peu connu peut n'avoir que quelques instantanés par an.
{% endhint %}

## En pratique

### Ce que les archives révèlent

#### Analyse historique

Comparer différentes versions d'un site permet d'identifier :

- **Des changements de structure** : répertoires ajoutés ou supprimés, réorganisations.
- **Des technologies abandonnées** : anciennes versions de CMS, frameworks obsolètes encore référencés.
- **Des données supprimées depuis** : pages, fichiers, contenus retirés de la version actuelle mais conservés dans les archives.

#### Accès à des ressources disparues

Certaines pages ou sous-domaines visibles dans le passé peuvent avoir été désindexés ou supprimés. Les archives peuvent révéler :

- Des zones d'administration oubliées.
- Des fichiers de configuration ou de sauvegarde.
- Des documents internes publiés par erreur puis retirés.

#### Collecte OSINT

Les anciennes pages peuvent contenir des informations exploitables : noms d'employés, détails sur les technologies utilisées, communiqués qui n'auraient pas du être publics.

#### Discrétion totale

Le Wayback Machine est une source publique tierce. Consulter les archives ne génère aucun trafic vers le site cible et ne laisse aucune trace dans ses logs. C'est de la reconnaissance passive pure.

### Utilisation

Pour consulter les archives d'un site, il suffit de se rendre sur le Wayback Machine et de saisir l'URL du domaine cible. L'interface affiche un calendrier avec les dates de capture disponibles.

## Pièges et galères

- **Couverture incomplète** : le Wayback Machine ne capture pas tout. Les pages protégées par authentification, les contenus en noindex/noarchive et les sites obscurs peuvent ne pas être archivés.
- **Contenus dynamiques** : les applications web modernes chargées en JavaScript ne sont souvent que partiellement capturées. Les formulaires, les fonctionnalités interactives et les sessions authentifiées ne sont pas reproductibles.
- **Informations obsolètes** : les données trouvées dans les archives reflètent l'état du site à un moment donné. Les configurations, technologies et contacts peuvent avoir changé depuis.

## Retour terrain

Le Wayback Machine est un réflexe à prendre en début de reconnaissance. Quelques minutes passées à explorer les anciennes versions d'un site peuvent révéler des informations que la version actuelle ne montre plus. C'est particulièrement utile pour les cibles qui ont récemment refondu leur site : l'ancienne version peut avoir été moins sécurisée ou plus bavarde.

## Mémo express

| Ressource | URL |
|---|---|
| Wayback Machine | `https://web.archive.org/web/` |
| Sauvegarder une page | `https://web.archive.org/save/` |

***
