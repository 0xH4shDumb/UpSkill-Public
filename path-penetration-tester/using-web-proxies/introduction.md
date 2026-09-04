# Introduction aux proxys web

## Pourquoi

En pentest web, la majorite du travail consiste a analyser et manipuler les requetes HTTP entre le navigateur et le serveur. Un proxy web se place au milieu de cette communication, capture chaque echange et permet de l'inspecter, de le modifier ou de le rejouer. Sans cet outil, on travaille a l'aveugle.

## Comment ca marche

Un proxy web fonctionne comme un intermediaire (man-in-the-middle) entre le navigateur et le serveur cible. Contrairement a un sniffer reseau comme Wireshark qui capture tout le trafic local, un proxy web se concentre sur les ports HTTP/HTTPS (80, 443) et permet d'interagir directement avec les requetes.

### Ce qu'un proxy web permet de faire

- Intercepter et modifier les requetes/reponses HTTP en temps reel
- Rejouer des requetes avec des parametres differents
- Scanner automatiquement des applications web
- Fuzzer des parametres, repertoires ou sous-domaines
- Encoder/decoder des donnees (Base64, URL, HTML)
- Mapper la structure complete d'une application

### Burp Suite vs ZAP

| Critere | Burp Suite | ZAP (OWASP) |
|---|---|---|
| Licence | Community (gratuit) + Pro (payant) | 100% gratuit et open-source |
| Fuzzer | Intruder (throttle en version gratuite) | ZAP Fuzzer (aucune limitation de vitesse) |
| Scanner | Pro uniquement | Inclus gratuitement |
| Crawler | Pro uniquement | Spider + Ajax Spider inclus |
| Navigateur integre | Chromium pre-configure | Firefox pre-configure |
| Extensions | BApp Store | ZAP Marketplace |

{% hint style="info" %}
En version gratuite, Burp Intruder est limite a 1 requete par seconde. Pour du fuzzing intensif sans payer, ZAP Fuzzer est une alternative solide.
{% endhint %}

## En pratique

### Installation

Les deux outils sont pre-installes sur Exegol et Kali. Sinon, les installer depuis leurs sites respectifs. Les deux necessitent un JRE (Java Runtime Environment), inclus dans les installeurs.

```bash
# Lancement Burp Suite
burpsuite

# Lancement ZAP
zaproxy

# Ou via le JAR
java -jar /chemin/vers/burpsuite.jar
java -jar /chemin/vers/zaproxy.jar
```

### Premier lancement de Burp

Au demarrage, Burp propose de creer un projet. En version Community, seuls les projets temporaires sont disponibles. Choisir `Temporary project`, puis `Use Burp Defaults` et cliquer sur `Start Burp`.

### Premier lancement de ZAP

ZAP propose de persister ou non la session. Pour une utilisation ponctuelle, choisir `No` (session temporaire).

{% hint style="success" %}
Pour activer le theme sombre : dans Burp, aller dans `Burp > Settings > User interface > Display > Theme > Dark`. Dans ZAP, `Tools > Options > Display > Look and Feel > Flat Dark`.
{% endhint %}

## Pieges et galeres

- Burp et ZAP utilisent le port 8080 par defaut. Si un autre service l'occupe, le proxy ne demarrera pas sans message explicite.
- En version gratuite de Burp, le projet n'est pas sauvegardable. Tout est perdu a la fermeture.
- Les deux outils consomment beaucoup de RAM sur les scans longs. Surveiller la memoire, surtout avec de grosses wordlists dans Intruder.

## Retour terrain

Le choix entre Burp et ZAP depend du contexte. Pour un engagement professionnel avec budget, Burp Pro offre un scanner mature et un Intruder rapide. Pour du lab, de la formation ou un budget serre, ZAP fournit les memes fonctionnalites sans limitation. En pratique, maitriser les deux est un avantage : certaines fonctionnalites sont mieux implementees dans l'un ou l'autre.

## Memo express

| Action | Burp | ZAP |
|---|---|---|
| Lancer | `burpsuite` | `zaproxy` |
| Port par defaut | 8080 | 8080 |
| Navigateur integre | `Proxy > Intercept > Open Browser` | Icone Firefox (barre d'outils) |
| Extensions | BApp Store | ZAP Marketplace |
| Fuzzer | Intruder (lent en free) | ZAP Fuzzer (pas de limite) |
| Scanner | Pro uniquement | Gratuit |

***
