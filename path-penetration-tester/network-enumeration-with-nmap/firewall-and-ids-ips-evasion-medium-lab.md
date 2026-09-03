# Lab — Identifier un service DNS derrière un pare-feu

## Scénario

Tu fais face à une machine dont le port 53 en TCP est filtré par un pare-feu. Un scan SYN classique ne donne rien d'exploitable sur ce port. Pourtant, un service DNS tourne bel et bien — il écoute simplement sur UDP, un protocole que beaucoup de pentesters oublient de scanner.

L'objectif : déterminer la version exacte du service DNS malgré le filtrage.

## Approche

Le raisonnement est le suivant :

1. Un scan TCP standard sur le port 53 renvoie `filtered` — le firewall bloque les paquets SYN.
2. Le DNS fonctionne nativement en UDP (résolution classique). Il y a de fortes chances que le port 53/udp soit ouvert, même si le TCP est verrouillé.
3. On combine un scan UDP (`-sU`) avec la détection de version (`-sV`) pour interroger le service directement.
4. Pour maximiser les chances de passer les filtres, on utilise le port source 53 (`--source-port 53`) — les pare-feux laissent souvent passer les réponses DNS — et on injecte des decoys (`-D RND:5`) pour noyer notre IP réelle dans le bruit.

## Commandes

Depuis Exegol, commence par vérifier que le port TCP 53 est bien filtré :

```bash
sudo nmap <IP_CIBLE> -p 53 -sS -Pn -n --disable-arp-ping
```

Tu devrais obtenir un état `filtered` sur le port 53/tcp. Maintenant, lance le scan UDP avec détection de version, source port et decoys :

```bash
sudo nmap <IP_CIBLE> -p 53 -sU -sV -Pn -n --disable-arp-ping --source-port 53 -D RND:5
```

Si tout se passe bien, le résultat ressemble à ceci :

```bash
PORT   STATE    SERVICE VERSION
53/tcp filtered domain
53/udp open     domain  BIND 9.x.x
```

La version du serveur DNS apparaît dans la colonne `VERSION`. Tu peux aussi affiner avec `--packet-trace` pour visualiser les échanges et comprendre quels paquets passent le filtre.

## Ce qu'on en retient

- **UDP, le grand oublié** — se limiter au TCP revient à scanner la moitié du spectre. DNS, SNMP, TFTP, NTP : beaucoup de services critiques tournent exclusivement en UDP.
- **Le port source comme passe-partout** — utiliser le port 53 comme source exploite le fait que la plupart des firewalls autorisent les flux DNS retour. C'est l'une des techniques d'évasion les plus simples et les plus efficaces.
- **Combiner les techniques** — decoys + source port + scan UDP : chaque couche augmente tes chances de passer inaperçu tout en obtenant l'information voulue.
- **Toujours vérifier les deux protocoles** — quand un port TCP est filtré, tester l'équivalent UDP avant de conclure que le service est inaccessible.

***
