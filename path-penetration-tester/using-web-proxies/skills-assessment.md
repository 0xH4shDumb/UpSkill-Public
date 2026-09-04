# Lab - Evaluation pratique

## Scenario

L'equipe de pentest teste une application web interne. Plusieurs situations se presentent, chacune necessitant une fonctionnalite differente du proxy web : modification de reponses, decodage de cookies, fuzzing de valeurs, et proxying d'outils externes. L'objectif est de choisir le bon outil pour chaque situation et de l'utiliser efficacement.

## Approche

1. **Boutons desactives** : intercepter la reponse HTTP et modifier le HTML pour reactiver les elements desactives, ou utiliser le HUD de ZAP (`Show/Enable`).
2. **Cookies encodes** : identifier le schema d'encodage (souvent multi-couche : Base64 + URL encoding + hex), decoder couche par couche avec le Decoder.
3. **Fuzzing de cookies** : quand un cookie est un hash MD5 incomplet, utiliser Intruder/Fuzzer pour tester toutes les valeurs possibles du caractere manquant, en appliquant le meme encodage sur chaque payload.
4. **Proxying d'outils** : router le trafic d'un module Metasploit a travers le proxy pour inspecter les requetes envoyees et comprendre pourquoi le module echoue.

## Commandes

### Intercepter et modifier une reponse

Activer l'interception des reponses dans le proxy, recharger la page avec `Ctrl+Shift+R`, puis modifier le HTML pour activer le bouton :

```html
<!-- Remplacer -->
<button disabled>Click me</button>

<!-- Par -->
<button>Click me</button>
```

### Decoder un cookie multi-couche

Dans le Decoder (Burp) ou l'Encoder/Decoder (ZAP, `Ctrl+E`) :

1. Coller la valeur du cookie
2. Decoder en URL
3. Decoder en Base64
4. Decoder en hex si necessaire
5. Repeter jusqu'a obtenir la valeur en clair

### Fuzzer un hash incomplet

Configurer Intruder avec la position sur le dernier caractere du hash :

```
Cookie: hash=<HASH_INCOMPLET>§a§
```

- Wordlist : `alphanum-case.txt` (SecLists)
- Payload Processing : appliquer les encodages dans l'ordre inverse du decodage (hex → Base64 → URL)
- Analyser les reponses pour identifier celle qui retourne un contenu different

### Proxyer un module Metasploit

```bash
msf6 > use auxiliary/scanner/http/coldfusion_locale_traversal
msf6 > set PROXIES HTTP:127.0.0.1:8080
msf6 > set RHOSTS <IP_CIBLE>
msf6 > run
```

Inspecter la requete dans l'historique du proxy pour identifier les repertoires appeles.

## Ce qu'on en retient

- Chaque situation appelle une fonctionnalite differente du proxy. Le reflexe est de se demander : est-ce que j'ai besoin d'intercepter, de rejouer, de fuzzer ou d'inspecter ?
- Le decodage multi-couche est frequent dans les applications reelles. Les cookies et tokens sont souvent encodes plusieurs fois pour echapper a la detection ou par design.
- Le proxying d'outils est un reflexe de debug. Quand un module ne fonctionne pas, voir la requete reelle dans Burp eclaire souvent le probleme immediatement.
- Combiner fuzzing et encodage dans les payload processors permet de construire des attaques sophistiquees sans ecrire de script.

***
