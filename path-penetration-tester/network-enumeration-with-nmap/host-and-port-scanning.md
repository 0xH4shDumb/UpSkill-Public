# Host and Port Scanning

## Pourquoi

Une fois que tu sais qu'une machine est vivante sur le réseau, la question suivante tombe naturellement : quels services tournent dessus ? Le scan de ports, c'est l'équivalent de frapper à chaque porte d'un immeuble pour voir lesquelles s'ouvrent. Chaque port ouvert est une surface d'attaque potentielle — un service qui écoute, qui répond, et qui a peut-être une faille.

Sans cette étape, tu avances à l'aveugle. Avec, tu dresses la carte complète de ce qui tourne sur la cible : SSH, HTTP, SMB, services exotiques sur des ports hauts… Tout ce qui pourra servir ensuite pour l'énumération de versions et la recherche de vulnérabilités.

## Comment ça marche

### Les états d'un port

Nmap classe chaque port scanné dans l'un de ces états :

| État | Signification |
| --- | --- |
| `open` | Le port accepte les connexions — un service écoute activement. |
| `closed` | Le port répond (RST en TCP) mais aucun service n'est derrière. |
| `filtered` | Impossible de déterminer l'état — un pare-feu avale ou rejette les paquets. |
| `unfiltered` | Le port est accessible (scan ACK) mais on ne sait pas s'il est ouvert ou fermé. |
| `open|filtered` | Nmap hésite entre ouvert et filtré — typique en UDP quand aucune réponse ne revient. |
| `closed|filtered` | Nmap ne parvient pas à trancher entre fermé et filtré. |

### TCP : SYN scan vs Connect scan

Pense au SYN scan (`-sS`) comme quelqu'un qui entrouvre une porte pour voir s'il y a de la lumière, puis la referme aussitôt. Il envoie un paquet SYN, attend le SYN-ACK, puis coupe avec un RST sans jamais finaliser la poignée de main TCP. Résultat : rapide, discret, et rarement logué côté cible.

Le Connect scan (`-sT`), lui, ouvre la porte en grand : il termine le three-way handshake complet (SYN → SYN-ACK → ACK). C'est plus fiable mais aussi plus bruyant — le système cible enregistre une connexion complète dans ses logs.

### UDP : un monde de silence

Le scan UDP (`-sU`) est fondamentalement différent. Pas de handshake, pas de confirmation de réception. Tu envoies un paquet et tu attends. Trois cas de figure :
- **Réponse UDP** → le port est ouvert
- **ICMP port unreachable** → le port est fermé
- **Rien du tout** → `open|filtered`, impossible de trancher

C'est pour ça que les scans UDP sont lents : Nmap doit attendre le timeout sur chaque port silencieux.

## En pratique

### Scanner les ports TCP les plus courants

```bash
# Depuis Exegol — scan rapide des 10 ports les plus fréquents
sudo nmap <IP_CIBLE> --top-ports=10
```

Nmap s'appuie sur sa base interne de fréquence pour choisir les ports les plus souvent ouverts. Pratique pour un premier aperçu rapide.

### SYN scan sur un port précis avec trace réseau

```bash
# Depuis Exegol — observer les échanges paquet par paquet
sudo nmap <IP_CIBLE> -p 21 --packet-trace -Pn -n --disable-arp-ping
```

L'option `--packet-trace` affiche chaque paquet envoyé et reçu. Tu vois exactement ce qui se passe au niveau réseau — indispensable pour comprendre pourquoi un port est marqué dans un état donné.

### TCP Connect scan avec justification

```bash
# Depuis Exegol — scan Connect complet avec raison de l'état
sudo nmap <IP_CIBLE> -p 443 -sT --packet-trace -Pn -n --disable-arp-ping --reason
```

Le flag `--reason` t'indique explicitement pourquoi Nmap a classé le port dans tel état (syn-ack reçu, rst reçu, pas de réponse…).

### Repérer un port filtré

Un pare-feu peut réagir de deux façons face à un scan :

```bash
# Depuis Exegol — port filtré silencieux (drop)
sudo nmap <IP_CIBLE> -p 139 --packet-trace -n -Pn --disable-arp-ping
```

Aucune réponse ne revient : Nmap retente plusieurs fois avant de conclure `filtered`. C'est lent.

```bash
# Depuis Exegol — port filtré avec rejet ICMP explicite
sudo nmap <IP_CIBLE> -p 445 --packet-trace -n -Pn --disable-arp-ping
```

Le pare-feu renvoie un message ICMP type 3/code 3 (port unreachable). C'est plus rapide à résoudre pour Nmap, mais ça donne aussi plus d'informations à l'attaquant.

### Scan UDP

```bash
# Depuis Exegol — scan UDP rapide (top 100 ports)
sudo nmap <IP_CIBLE> -F -sU
```

```bash
# Depuis Exegol — scan UDP ciblé avec trace et justification
sudo nmap <IP_CIBLE> -sU -Pn -n --disable-arp-ping --packet-trace -p 137 --reason
```

### Détection de version rapide

```bash
# Depuis Exegol — identifier le service derrière un port
sudo nmap <IP_CIBLE> -Pn -n --disable-arp-ping --packet-trace -p 445 --reason -sV
```

Le flag `-sV` pousse Nmap à interroger le service pour récupérer sa bannière et déterminer sa version exacte — bien plus utile qu'un simple numéro de port.

## Pièges & galères

- **SYN scan nécessite les droits root** : sans `sudo`, Nmap bascule automatiquement en Connect scan (`-sT`), plus lent et plus bruyant. Pense-y si tes scans traînent.
- **UDP = patience** : un scan UDP complet (`-p-`) peut durer des heures. Limite-toi aux ports connus (`-F` ou `--top-ports`) sauf besoin spécifique.
- **Un port `filtered` ne veut pas dire "rien ici"** : ça signifie qu'un équipement réseau bloque ta visibilité. Le service peut très bien tourner derrière. Change de technique (ACK scan, source port 53…) avant de conclure.
- **`open|filtered` en UDP** : c'est l'état le plus frustrant. Aucune réponse ne revient, et Nmap ne peut pas trancher. Essaie un scan de version (`-sV`) sur ce port pour forcer une interaction applicative.
- **La résolution DNS ralentit tout** : ajoute `-n` systématiquement pour éviter les lookups DNS inutiles qui ajoutent de la latence à chaque hôte.

## Retour terrain

En pentest interne, le réflexe est de lancer un `--top-ports=1000` en SYN scan sur tout le sous-réseau pour avoir une cartographie rapide, puis de revenir en mode ciblé (`-p-`) sur les machines intéressantes. Le piège classique, c'est d'oublier l'UDP : des services critiques comme SNMP (161), DNS (53) ou TFTP (69) ne tournent qu'en UDP et passent complètement sous le radar si tu ne les cherches pas.

L'option `--reason` est sous-utilisée, mais elle fait gagner un temps fou en troubleshooting. Quand un port affiche un état inattendu, la raison te dit immédiatement si c'est un RST, un timeout ou un ICMP — et ça oriente ta stratégie de contournement.

## Mémo express

| Option | Rôle |
| --- | --- |
| `-sS` | SYN scan — rapide et discret (défaut avec sudo) |
| `-sT` | TCP Connect — handshake complet, plus fiable mais logué |
| `-sU` | UDP scan — lent, indispensable pour SNMP/DNS/TFTP |
| `-sV` | Détection de version des services |
| `--top-ports=N` | Scanner les N ports les plus fréquents |
| `-p 22,80,443` | Cibler des ports spécifiques |
| `-p-` | Scanner tous les 65535 ports |
| `--packet-trace` | Afficher chaque paquet envoyé/reçu |
| `--reason` | Expliquer pourquoi un port est dans cet état |
| `-n` | Désactiver la résolution DNS |
| `-Pn` | Sauter la découverte d'hôte (pas de ping) |
| `--disable-arp-ping` | Forcer l'utilisation d'ICMP au lieu d'ARP |

***
