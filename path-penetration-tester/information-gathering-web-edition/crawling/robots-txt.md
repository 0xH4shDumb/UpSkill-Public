# robots.txt

## Pourquoi

Le fichier `robots.txt` est un fichier texte placé à la racine d'un site web (`www.exemple.com/robots.txt`) qui indique aux robots d'indexation quelles parties du site ils peuvent explorer et lesquelles sont interdites. C'est un standard volontaire (Robots Exclusion Standard) : les crawlers légitimes le respectent, les malveillants l'ignorent. Pour un pentester, `robots.txt` est une source d'information précieuse parce qu'il révèle intentionnellement ce que le propriétaire du site cherche à cacher.

## Comment ça marche

### Structure du fichier

Le fichier est composé de blocs de directives, chacun ciblant un user-agent spécifique :

```txt
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://www.exemple.com/sitemap.xml
```

### Directives principales

| Directive | Rôle | Exemple |
|---|---|---|
| `User-agent` | Spécifie le crawler ciblé (`*` pour tous) | `User-agent: Googlebot` |
| `Disallow` | Interdit l'accès à un chemin | `Disallow: /admin/` |
| `Allow` | Autorise explicitement un chemin (même sous un `Disallow` parent) | `Allow: /public/` |
| `Crawl-delay` | Délai (en secondes) entre les requêtes d'un crawler | `Crawl-delay: 10` |
| `Sitemap` | URL du plan de site XML | `Sitemap: https://exemple.com/sitemap.xml` |

{% hint style="warning" %}
`robots.txt` n'est pas un mécanisme de sécurité. Il ne bloque pas l'accès aux chemins listés, il demande poliment aux robots de ne pas les explorer. Rien n'empêche un humain (ou un robot mal intentionné) de visiter ces URLs directement.
{% endhint %}

## En pratique

### Ce que robots.txt révèle en reconnaissance

- **Répertoires cachés** : les chemins en `Disallow` pointent souvent vers des zones sensibles (panels d'administration, répertoires de sauvegarde, environnements de test). C'est littéralement une liste de ce que le propriétaire ne veut pas qu'on trouve.
- **Structure du site** : en analysant les chemins autorisés et interdits, on peut reconstruire une cartographie partielle de l'arborescence du site.
- **Pièges à crawlers** : certains sites incluent des répertoires "honeypot" dans `robots.txt` pour détecter les bots malveillants. L'identifier donne un indice sur la maturité sécurité de la cible.

### Analyse d'un robots.txt

```txt
User-agent: *
Disallow: /admin/
Disallow: /backup/
Disallow: /api/internal/
Allow: /api/public/

Sitemap: https://www.exemple.com/sitemap.xml
```

Ce fichier révèle :

- Un panel d'administration probable à `/admin/`.
- Un répertoire de sauvegardes à `/backup/` (potentiellement exploitable).
- Une API interne à `/api/internal/` (séparée de l'API publique).
- Un sitemap XML qui peut contenir des URLs supplémentaires non visibles sur le site.

### Consulter le robots.txt

```bash
# - Via curl
curl -s http://<DOMAINE_CIBLE>/robots.txt

# - Via un navigateur
# Simplement naviguer vers http://<domaine>/robots.txt
```

## Pièges et galères

- **Informations obsolètes** : un `robots.txt` peut contenir des chemins qui n'existent plus. Toujours vérifier que les répertoires listés sont encore accessibles.
- **Faux sentiment de sécurité** : les administrateurs qui utilisent `robots.txt` comme seule protection de leurs répertoires sensibles se trompent. Ce n'est pas du contrôle d'accès.
- **Piège juridique** : dans certaines juridictions, ignorer les directives de `robots.txt` peut être considéré comme une violation des conditions d'utilisation du site. Rester dans le cadre d'un engagement autorisé.

## Mémo express

| Besoin | Commande |
|---|---|
| Consulter le robots.txt | `curl -s http://<cible>/robots.txt` |
| Extraire les chemins Disallow | `curl -s http://<cible>/robots.txt \| grep Disallow` |
| Récupérer le sitemap | `curl -s http://<cible>/robots.txt \| grep Sitemap` |

***
