# Nmap Scripting Engine (NSE)

## Pourquoi

Un scan de ports, c'est bien. Savoir que le port 80 est ouvert, c'est un début. Mais ce qui t'intéresse vraiment, c'est ce qui tourne derrière : quelle version, quelles failles, quelles infos le service laisse fuiter. Le Nmap Scripting Engine transforme Nmap d'un simple scanner en véritable couteau suisse — capable d'énumérer, de détecter des vulnérabilités, voire d'exploiter certaines failles, le tout en une seule passe.

Pense au NSE comme une boîte à plugins : chaque script est un module spécialisé que tu branches sur ton scan pour aller chercher une info précise.

## Comment ça marche

Le NSE repose sur des scripts écrits en **Lua**, un langage léger et rapide. Ces scripts sont stockés dans `/usr/share/nmap/scripts/` et classés par catégories selon leur finalité.

| Catégorie   | Rôle |
|-------------|------|
| `auth`      | Teste des identifiants sur les services (login par défaut, anonymous, etc.) |
| `broadcast` | Envoie des requêtes broadcast pour découvrir des services sur le réseau |
| `brute`     | Lance des attaques par force brute contre les services détectés |
| `default`   | Scripts lancés automatiquement avec l'option `-sC` |
| `discovery` | Récupère des informations complémentaires sur les services |
| `dos`       | Teste la résistance aux dénis de service — à manier avec précaution |
| `exploit`   | Tente d'exploiter des vulnérabilités connues |
| `external`  | Fait appel à des services tiers pour enrichir les résultats |
| `fuzzer`    | Envoie des données aléatoires pour tester la robustesse |
| `intrusive` | Scripts potentiellement perturbateurs pour la cible |
| `malware`   | Vérifie si la machine est infectée par un malware connu |
| `safe`      | Scripts non destructifs, utilisables sans risque |
| `version`   | Complète la détection de version (`-sV`) avec des sondes supplémentaires |
| `vuln`      | Cherche des vulnérabilités connues (CVE) sur les services détectés |

Tu peux lister tous les scripts disponibles avec `ls /usr/share/nmap/scripts/` ou chercher un script précis avec `--script-help <nom>`.

## En pratique

### Lancer les scripts par défaut

L'option `-sC` active la catégorie `default` — un bon point de départ pour tout scan d'énumération :

```bash
# Depuis Exegol
sudo nmap <IP_CIBLE> -sC
```

### Cibler une catégorie précise

Pour chercher des vulnérabilités connues sur un service web :

```bash
# Depuis Exegol
sudo nmap <IP_CIBLE> -p 80 -sV --script vuln
```

Le script `vulners` va croiser la version détectée avec sa base de CVE et te sortir les failles référencées. Combiné avec `http-enum`, tu obtiens aussi l'arborescence des fichiers exposés (pages d'admin, `robots.txt`, etc.).

### Appeler des scripts spécifiques

Tu peux nommer directement les scripts qui t'intéressent, séparés par des virgules :

```bash
# Depuis Exegol
sudo nmap <IP_CIBLE> -p 25 --script banner,smtp-commands
```

Le script `banner` récupère la bannière brute du service (souvent plus bavarde que ce qu'affiche `-sV`). Le script `smtp-commands` liste les commandes SMTP acceptées — utile pour savoir si `VRFY` est activé et permet l'énumération d'utilisateurs.

### Le mode agressif `-A`

Cette option regroupe quatre fonctionnalités en une seule commande :

- Détection de version (`-sV`)
- Détection d'OS (`-O`)
- Traceroute (`--traceroute`)
- Scripts par défaut (`-sC`)

```bash
# Depuis Exegol
sudo nmap <IP_CIBLE> -p 80 -A
```

C'est pratique pour un premier survol, mais c'est bruyant — à éviter quand la discrétion compte.

## Pièges & galères

- **Les scripts `intrusive` et `exploit` peuvent casser des choses.** Un script de brute force mal calibré peut verrouiller des comptes, un script `dos` peut faire tomber un service. En pentest, vérifie toujours le périmètre autorisé avant de lancer ces catégories.
- **`-sC` ne veut pas dire "scan complet".** La catégorie `default` est un compromis entre utilité et sécurité. Elle n'inclut ni les scripts de brute force, ni les exploits. Ne te repose pas dessus pour une énumération exhaustive.
- **Les scripts sont datés.** La base de scripts livrée avec Nmap n'est pas toujours à jour. Pense à `sudo nmap --script-updatedb` après avoir ajouté des scripts manuellement.
- **Le temps de scan explose.** Lancer `--script vuln` sur tous les ports d'un /24, c'est partir pour des heures. Cible d'abord les ports intéressants, puis affine avec les scripts.

## Retour terrain

En pratique, le combo le plus rentable pour un premier contact, c'est `-sV --script=default,vuln` sur les ports ouverts déjà identifiés. Tu obtiens les versions ET les CVE en une passe. Pour le web, ajoute `http-enum` et `http-title` — ça t'évite de curl chaque service à la main.

Le banner grabbing via NSE (`--script banner`) est souvent plus fiable que `-sV` seul. Certains services renvoient leur bannière complète au script alors qu'ils restent muets face aux sondes de version standard.

Si tu tombes sur un service exotique, consulte la doc du script avec `nmap --script-help <nom>` avant de le lancer à l'aveugle — certains scripts envoient des payloads qui peuvent déclencher des alertes.

## Mémo express

| Action | Commande |
|--------|----------|
| Scripts par défaut | `nmap <IP> -sC` |
| Catégorie spécifique | `nmap <IP> --script vuln` |
| Scripts nommés | `nmap <IP> --script banner,smtp-commands` |
| Scan agressif complet | `nmap <IP> -A` |
| Aide sur un script | `nmap --script-help <nom>` |
| Mettre à jour la base | `sudo nmap --script-updatedb` |
| Répertoire des scripts | `/usr/share/nmap/scripts/` |

***
