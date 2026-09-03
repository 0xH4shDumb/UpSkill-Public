# Contournement de pare-feu et IDS/IPS

## Pourquoi

En pentest, tu tomberas quasi systématiquement sur des pare-feux ou des systèmes de détection d'intrusion. Un scan Nmap "classique" se fait alors bloquer, filtrer, ou pire : il déclenche une alerte et tu te fais repérer avant même d'avoir commencé. Savoir contourner ces protections, c'est la différence entre un scan qui te renvoie `filtered` sur tous les ports et un scan qui te donne de vraies informations exploitables.

## Comment ça marche

Trois acteurs entrent en jeu côté défense, et il faut bien les distinguer :

| Composant | Rôle |
|---|---|
| **Firewall** | Filtre le trafic réseau selon des règles prédéfinies — il laisse passer ou bloque, point. |
| **IDS** (Intrusion Detection System) | Observe le trafic de manière passive et lève une alerte si quelque chose sent mauvais. Il ne bloque rien. |
| **IPS** (Intrusion Prevention System) | Même principe que l'IDS, mais en mode actif : il coupe la connexion suspecte directement. |

Pense au firewall comme un videur qui vérifie ta carte d'identité, l'IDS comme une caméra de surveillance qui te filme, et l'IPS comme un vigile qui te plaque au sol dès que tu as l'air louche.

### Le scan ACK pour cartographier les règles

Le scan SYN (`-sS`) te dit si un port est ouvert, fermé ou filtré. Mais il ne te dit pas *pourquoi* il est filtré. C'est là qu'intervient le scan ACK (`-sA`) : il envoie un paquet ACK (comme si une connexion existait déjà) et observe la réaction.

- Si le firewall renvoie un RST → le paquet a traversé → port **unfiltered** (pas de règle de filtrage sur ce port)
- Si rien ne revient ou ICMP unreachable → le paquet a été bloqué → port **filtered**

En croisant les résultats SYN et ACK, tu déduis les règles du pare-feu.

### Les decoys pour noyer ton IP

Avec `-D RND:5`, Nmap génère 5 adresses IP aléatoires et envoie les mêmes paquets depuis chacune d'elles en plus de la tienne. Côté logs du défenseur, ton IP réelle se retrouve noyée dans un tas d'adresses — beaucoup plus dur à identifier.

### Le port source DNS pour passer les filtres

Beaucoup de pare-feux laissent passer le trafic venant du port 53 (DNS) parce que les réponses DNS doivent pouvoir revenir. En forçant ton port source à 53 avec `--source-port 53`, tu exploites cette confiance aveugle pour traverser des filtres qui bloqueraient autrement tes paquets.

## En pratique

### Comparer SYN scan et ACK scan

```bash
# Depuis Exegol — SYN scan pour voir l'état des ports
sudo nmap <IP_CIBLE> -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace
```

Résultat typique : le port 22 répond `open`, les ports 21 et 25 apparaissent `filtered`.

```bash
# Depuis Exegol — ACK scan pour déduire les règles firewall
sudo nmap <IP_CIBLE> -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

Résultat : le port 22 passe en `unfiltered` (le firewall le laisse passer), les ports 21 et 25 restent `filtered`. Tu sais maintenant que le firewall a des règles spécifiques sur 21 et 25, mais pas sur 22.

### Masquer son IP avec des decoys

```bash
# Depuis Exegol — scan SYN avec 5 IPs leurres
sudo nmap <IP_CIBLE> -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
```

Dans les logs côté cible, 6 adresses différentes apparaissent comme source du scan. Bonne chance pour retrouver la vraie.

### Contourner un filtre via le port source DNS

```bash
# Depuis Exegol — scan classique (bloqué)
sudo nmap <IP_CIBLE> -p 50000 -sS -Pn -n --disable-arp-ping --packet-trace
```

Le port apparaît `filtered`. Maintenant, même scan mais avec le port source 53 :

```bash
# Depuis Exegol — scan en se faisant passer pour du trafic DNS
sudo nmap <IP_CIBLE> -p 50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
```

Le port passe en `open`. Le firewall a laissé passer le paquet parce qu'il venait du port 53.

### Confirmer avec Netcat

```bash
# Depuis Exegol — connexion directe en forçant le port source
ncat -nv --source-port 53 <IP_CIBLE> 50000
```

Si la connexion s'établit et qu'une bannière s'affiche, c'est confirmé : le filtre est contourné.

## Pièges & galères

- **Les decoys ne marchent que si les IPs générées sont plausibles.** Si elles tombent dans des plages non routables ou visiblement fausses, un analyste les repère immédiatement.
- **Le port source 53 ne fonctionne pas partout.** Les pare-feux modernes inspectent le contenu des paquets (DPI) et vérifient que ce qui vient du port 53 ressemble vraiment à du DNS.
- **Le scan ACK ne te dit pas si un port est ouvert** — seulement s'il est filtré ou non. Ne confonds pas `unfiltered` et `open`.
- **Attention au bruit généré.** Un scan avec decoys multiplie le nombre de paquets envoyés — un IDS basé sur le volume de trafic va quand même réagir.

## Retour terrain

En situation réelle, la technique du port source 53 reste étonnamment efficace sur des réseaux d'entreprise mal segmentés. J'ai souvent vu des pare-feux qui autorisent aveuglément tout ce qui vient du port 53, probablement pour éviter de casser la résolution DNS interne. Le scan ACK est sous-utilisé par beaucoup de pentesters — pourtant, c'est le meilleur moyen de comprendre la logique de filtrage avant de chercher à la contourner. Commence toujours par cartographier avant de foncer.

## Mémo express

| Technique | Commande | Effet |
|---|---|---|
| SYN scan | `-sS` | État des ports (open/filtered/closed) |
| ACK scan | `-sA` | Cartographie des règles firewall |
| Decoys | `-D RND:5` | Noie ton IP parmi des leurres |
| Port source DNS | `--source-port 53` | Exploite la confiance du firewall envers le DNS |
| Confirmation manuelle | `ncat -nv --source-port 53 <IP> <PORT>` | Vérifie le contournement en se connectant |

***
