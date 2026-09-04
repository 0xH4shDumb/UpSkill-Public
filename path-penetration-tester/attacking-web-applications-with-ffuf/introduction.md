# Introduction au fuzzing web

## Pourquoi

Quand on arrive sur une application web, la page d'accueil ne montre qu'une fraction de ce qui existe sur le serveur. Les repertoires caches, les pages d'administration, les sous-domaines internes, les parametres non documentes : tout ca n'est pas indexe et ne sera pas decouvert par un simple clic. Le fuzzing web consiste a tester systematiquement des milliers de valeurs pour reveler ces ressources.

## Comment ca marche

Le fuzzing envoie des requetes HTTP en masse en remplacant une partie de l'URL (ou d'un header, ou d'un parametre) par des valeurs issues d'une wordlist. La reponse du serveur (code HTTP, taille, nombre de mots) permet de distinguer les ressources existantes des erreurs 404.

### ffuf en bref

`ffuf` (Fuzz Faster U Fool) est un fuzzer web ecrit en Go. Il est rapide (plusieurs milliers de requetes par seconde), flexible (positions multiples, filtres puissants), et supporte les requetes GET, POST, les headers personnalises et la recursion.

```bash
# Syntaxe de base
ffuf -w <wordlist>:FUZZ -u http://<IP_CIBLE>/FUZZ
```

Le mot-cle `FUZZ` est le marqueur de position. Il est remplace par chaque ligne de la wordlist. On peut utiliser plusieurs marqueurs (`FUZZ`, `FUZZ2`, etc.) pour fuzzer plusieurs positions simultanement.

### Options principales

| Option | Role |
|---|---|
| `-w` | Wordlist et mot-cle (ex: `/path/to/list:FUZZ`) |
| `-u` | URL cible avec position `FUZZ` |
| `-X` | Methode HTTP (`GET`, `POST`) |
| `-d` | Donnees POST |
| `-H` | Header personnalise |
| `-e` | Extensions a ajouter (ex: `.php,.html`) |
| `-t` | Nombre de threads (defaut: 40) |
| `-ic` | Ignorer les commentaires dans la wordlist |
| `-v` | Mode verbose (affiche les URLs completes) |
| `-o` | Ecrire les resultats dans un fichier |

### Wordlists essentielles (SecLists)

| Usage | Wordlist |
|---|---|
| Repertoires/fichiers | `Discovery/Web-Content/directory-list-2.3-small.txt` |
| Extensions | `Discovery/Web-Content/web-extensions.txt` |
| Sous-domaines | `Discovery/DNS/subdomains-top1million-5000.txt` |
| Parametres | `Discovery/Web-Content/burp-parameter-names.txt` |

{% hint style="info" %}
SecLists est disponible dans `/usr/share/seclists/` sur Exegol et Kali. Certaines wordlists contiennent des lignes de commentaires en debut de fichier. L'option `-ic` permet de les ignorer.
{% endhint %}

## Pieges et galeres

- Augmenter le nombre de threads (`-t 200`) accelere le scan mais peut declencher un ban IP, un rate limiter, ou un DoS involontaire. Rester sous 100 threads en engagement.
- Les wordlists contiennent parfois des lignes vides ou des caracteres speciaux qui generent des faux positifs. Nettoyer la liste en amont si necessaire.
- ffuf affiche par defaut les reponses avec les codes 200, 301, 302, 307, 401, 403. Un 403 n'est pas un echec : il confirme l'existence de la ressource.

## Retour terrain

ffuf est devenu l'outil de reference pour le fuzzing web, devant gobuster et wfuzz, grace a sa vitesse et sa flexibilite. En engagement, le premier reflexe sur une application web est de lancer un scan de repertoires avec une wordlist moyenne (directory-list-2.3-small). Si le scope le permet, enchainer avec un scan de sous-domaines et de vhosts. L'objectif n'est pas d'etre exhaustif mais d'identifier rapidement les surfaces d'attaque cachees.

## Memo express

| Action | Commande |
|---|---|
| Fuzzing de repertoires | `ffuf -w list:FUZZ -u http://<IP>/FUZZ` |
| Fuzzing de fichiers PHP | `ffuf -w list:FUZZ -u http://<IP>/FUZZ.php` |
| Fuzzing de sous-domaines | `ffuf -w list:FUZZ -u http://FUZZ.domaine.com` |
| Fuzzing de vhosts | `ffuf -w list:FUZZ -u http://<IP> -H "Host: FUZZ.domaine.com"` |
| Fuzzing de parametres GET | `ffuf -w list:FUZZ -u http://<IP>/page?FUZZ=val` |
| Fuzzing de parametres POST | `ffuf -w list:FUZZ -u http://<IP>/page -X POST -d "FUZZ=val"` |

***
