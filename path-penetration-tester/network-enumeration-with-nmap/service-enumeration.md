# Service Enumeration

## Pourquoi

Savoir qu'un port est ouvert, c'est bien. Savoir ce qui tourne derrière, c'est ce qui te permet d'avancer. Un port 80 peut héberger un Apache obsolète, un Nginx durci ou un reverse proxy qui masque tout. Sans détection de version, tu tires à l'aveugle — tu perds du temps sur des pistes mortes et tu rates les failles évidentes.

L'énumération de services te donne trois choses : le nom du service, sa version exacte, et parfois des informations bonus planquées dans la bannière (nom d'hôte, OS, voire des données sensibles). C'est la base pour chercher des CVE et orienter ton exploitation.

## Comment ça marche

### Détection de version avec `-sV`

Quand tu lances un scan de version, Nmap ne se contente pas de regarder le numéro de port. Il envoie des probes spécifiques au service et analyse les réponses pour identifier précisément ce qui tourne. Le processus suit plusieurs étapes :

1. **Connexion TCP** — Nmap établit une connexion complète sur le port
2. **Récupération de bannière** — Il attend que le service envoie sa bannière d'accueil (si le service en a une)
3. **Probes actifs** — Si la bannière ne suffit pas, Nmap envoie des requêtes spécifiques tirées de sa base `nmap-service-probes`
4. **Matching** — La réponse est comparée aux signatures connues pour identifier service + version

### Banner grabbing

Certains services annoncent spontanément leur identité dès la connexion : c'est la bannière. Un serveur SMTP te dira `220 mail.example.com ESMTP Postfix (Ubuntu)`, un SSH te balancera `SSH-2.0-OpenSSH_8.9p1`. Ces informations ne figurent pas toujours dans la sortie standard de Nmap — d'où l'intérêt de récupérer les bannières manuellement.

## En pratique

### Scan de version complet

```bash
# Depuis Exegol — scan de tous les ports avec détection de version
sudo nmap -p- -sV <IP_CIBLE>
```

L'option `-p-` scanne l'intégralité des 65535 ports. Sur un réseau lent, ça peut prendre un moment. Deux options pour suivre la progression :

```bash
# Depuis Exegol — affichage des stats toutes les 5 secondes
sudo nmap -p- -sV --stats-every=5s <IP_CIBLE>
```

```bash
# Depuis Exegol — mode verbeux, affiche chaque port dès sa découverte
sudo nmap -p- -sV -v <IP_CIBLE>
```

Tu peux aussi appuyer sur `[Espace]` pendant un scan en cours pour afficher l'état d'avancement sans l'interrompre.

### Banner grabbing avec Nmap

Pour voir le détail des échanges réseau et les bannières brutes :

```bash
# Depuis Exegol — scan avec trace des paquets
sudo nmap -p- -sV -Pn -n --disable-arp-ping --packet-trace <IP_CIBLE>
```

Dans la sortie, cherche les lignes `NSOCK` — elles contiennent les bannières renvoyées par les services, parfois plus bavardes que ce que Nmap affiche dans son tableau final.

### Banner grabbing manuel avec Netcat + Tcpdump

Nmap fait du bon boulot, mais il ne montre pas tout. Pour récupérer la bannière brute telle que le service l'envoie, connecte-toi directement avec `nc` tout en capturant le trafic :

```bash
# Depuis Exegol — terminal 1 : capture du trafic
sudo tcpdump -i eth0 host <IP_CIBLE> and port 25
```

```bash
# Depuis Exegol — terminal 2 : connexion manuelle au service SMTP
nc -nv <IP_CIBLE> 25
```

Tu verras la bannière complète s'afficher (`220 hostname ESMTP Postfix (Ubuntu)`) et dans tcpdump, le détail du handshake TCP suivi de l'échange applicatif. C'est la méthode la plus fiable quand Nmap retourne `tcpwrapped` ou que le service ne matche aucune signature.

| Drapeau TCP | Signification |
|---|---|
| `SYN` | Demande d'ouverture de connexion |
| `SYN-ACK` | Le serveur accepte |
| `ACK` | Connexion établie |
| `PSH-ACK` | Données transmises avec acquittement |

## Pièges & galères

- **`tcpwrapped` partout** — Le service utilise TCP Wrapper et refuse ta connexion avant même d'envoyer une bannière. Essaie depuis un port source autorisé (`--source-port 53`) ou connecte-toi manuellement avec `nc`
- **Bannières trompeuses** — Rien n'empêche un admin de modifier la bannière de son service pour afficher une fausse version. Croise toujours avec d'autres indicateurs (comportement du service, scripts NSE)
- **Scan de version lent** — `-sV` avec `-p-` sur un réseau distant, ça peut durer une éternité. Commence par un SYN scan rapide pour lister les ports ouverts, puis lance `-sV` uniquement sur ceux-là
- **Services sur ports non standard** — Un SSH sur le port 2222, un HTTP sur 8443… sans `-sV`, Nmap affiche le nom associé au port dans sa base, pas le vrai service

## Retour terrain

En pentest, le scan de version est souvent la première chose que tu lances après la découverte d'hôtes. Le réflexe classique : un SYN scan rapide (`-sS -p-`) pour la couverture, puis un `-sV` ciblé sur les ports ouverts. En interne, tu peux te permettre d'être agressif (`-T4`). En externe, modère le rythme — un scan de version génère du trafic applicatif qui se voit dans les logs.

Le banner grabbing manuel avec `nc` reste ton meilleur ami quand Nmap ne te donne pas assez d'infos. Sur un engagement réel, j'ai vu des services remonter des infos critiques dans leur bannière (version exacte d'un logiciel vulnérable, nom de domaine interne) que Nmap avait tronquées dans son affichage.

## Mémo express

| Objectif | Commande |
|---|---|
| Scan de version tous ports | `nmap -p- -sV <IP_CIBLE>` |
| Version + progression | `nmap -p- -sV --stats-every=5s <IP_CIBLE>` |
| Version + verbeux | `nmap -p- -sV -v <IP_CIBLE>` |
| Banner grabbing Nmap | `nmap -p- -sV -Pn -n --packet-trace <IP_CIBLE>` |
| Banner grabbing Netcat | `nc -nv <IP_CIBLE> <PORT>` |
| Capture trafic | `tcpdump -i eth0 host <IP_CIBLE>` |

***
