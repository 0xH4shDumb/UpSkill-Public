# Lab — Identifier l'OS d'une cible

## Scénario

Tu es en phase de reconnaissance sur un réseau interne. Une machine répond aux scans basiques, mais le client veut savoir quel système d'exploitation tourne dessus. L'objectif est simple en apparence : déterminer l'OS à distance, sans accès authentifié.

## Approche

Nmap propose deux angles complémentaires pour deviner l'OS d'une cible :

- **OS fingerprinting (`-O`)** : Nmap envoie une série de paquets craftés et compare les réponses (taille de fenêtre TCP, options IP, comportement ICMP…) à sa base de signatures. Ça marche bien sur des machines avec plusieurs ports ouverts, mais ça peut rater si la cible est trop filtrée.
- **Banner grabbing (`--script=banner`)** : on récupère les bannières renvoyées par les services actifs. Un serveur SSH qui annonce `OpenSSH_7.6p1 Ubuntu-4ubuntu0.7` te donne l'OS sur un plateau, sans même avoir besoin du fingerprint.

En pratique, tu combines les deux pour maximiser tes chances : le fingerprint donne une estimation probabiliste, et les bannières confirment (ou corrigent) le résultat.

## Commandes

Depuis Exegol, lance un scan combinant détection d'OS et récupération de bannières :

```bash
sudo nmap <IP_CIBLE> -sS -Pn -n --disable-arp-ping -O --script=banner
```

| Option | Rôle |
|---|---|
| `-sS` | SYN scan — discret, pas de connexion complète |
| `-Pn` | Pas de ping préalable — on suppose la cible active |
| `-n` | Pas de résolution DNS — gain de temps |
| `--disable-arp-ping` | Évite les requêtes ARP (utile à travers un VPN) |
| `-O` | Active la détection d'OS par fingerprinting |
| `--script=banner` | Récupère les bannières des services ouverts |

Exemple de sortie typique :

```bash
PORT      STATE SERVICE
22/tcp    open  ssh
|_banner: SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.7
80/tcp    open  http
10001/tcp open  scp-config
```

Ici, la bannière SSH te confirme directement qu'il s'agit d'une machine **Ubuntu**. Le fingerprint OS de Nmap peut ne pas trouver de correspondance exacte (`No exact OS matches`), mais la bannière suffit à conclure.

## Ce qu'on en retient

- Le fingerprinting OS (`-O`) n'est pas infaillible — il a besoin d'au moins un port ouvert et un port fermé pour fonctionner correctement. Si la cible est très filtrée, il ne donnera rien d'exploitable.
- Les bannières de services sont souvent plus fiables pour identifier l'OS que le fingerprinting lui-même. Un simple `SSH-2.0-OpenSSH_X.Xp1 Ubuntu-...` suffit.
- Combine toujours les deux approches : `-O` pour l'estimation globale, `--script=banner` pour la confirmation par les services.
- Pense à vérifier aussi le **TTL** dans les réponses (`--packet-trace` ou `--reason`) : un TTL de 64 pointe vers Linux, 128 vers Windows. C'est un indice supplémentaire quand le fingerprint hésite.

***
