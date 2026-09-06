# Introduction au pivoting et au tunneling

Le pivoting est l'art de traverser des reseaux segmentes en utilisant un hote compromis comme relais. En pentest, on se retrouve regulierement dans une situation ou l'on a compromis un serveur qui dispose de deux interfaces reseau : l'une accessible depuis notre machine d'attaque, l'autre connectee a un reseau interne hors de portee. Le pivoting permet de transformer cet hote en passerelle pour atteindre les cibles du reseau interne.

## Pourquoi

La segmentation reseau est une mesure de securite fondamentale. Les serveurs critiques (controleurs de domaine, bases de donnees, applications internes) ne sont jamais directement accessibles depuis Internet. Pour les atteindre, il faut pivoter a travers un ou plusieurs hotes intermediaires. C'est une competence indispensable pour tout pentest interne ou toute evaluation Active Directory.

## Comment ca marche

### Pivoting, tunneling et mouvement lateral

Ces trois concepts sont lies mais distincts :

| Concept | Definition | Objectif |
|---|---|---|
| **Pivoting** | Utiliser un hote compromis pour acceder a un reseau autrement inaccessible | Traverser les frontieres reseau |
| **Tunneling** | Encapsuler du trafic reseau dans un autre protocole (SSH, DNS, ICMP, HTTP) | Contourner les firewalls et masquer le trafic |
| **Mouvement lateral** | Se deplacer entre les hotes d'un meme reseau pour elever ses privileges | Elargir l'acces au sein du meme segment |

En pratique, on combine souvent les trois : mouvement lateral pour compromettre un hote dual-homed, pivoting pour atteindre le reseau suivant, tunneling pour faire passer le trafic a travers les firewalls.

### Concepts reseau essentiels

#### Interfaces reseau et adressage

Un hote avec plusieurs interfaces reseau (NICs) est un candidat ideal pour le pivoting. La premiere chose a faire apres avoir compromis un hote est de verifier ses interfaces :

{% tabs %}
{% tab title="Linux" %}
```bash
# - Lister les interfaces reseau
ifconfig
# ou
ip addr show
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# - Lister les interfaces reseau
ipconfig /all
```
{% endtab %}
{% endtabs %}

Un hote avec une interface sur `10.129.x.x` (reseau de l'attaquant) et une autre sur `172.16.5.x` (reseau interne) est un pivot parfait.

#### Table de routage

La table de routage indique comment l'hote envoie le trafic vers les differents reseaux :

```bash
# - Afficher la table de routage
netstat -r
# ou
ip route
```

Chaque entree montre un sous-reseau de destination, la passerelle associee et l'interface de sortie. Un reseau present dans la table de routage est potentiellement joignable depuis cet hote.

#### Ports et protocoles

Les firewalls filtrent le trafic par port et protocole. En pivoting, on exploite les ports autorises (SSH/22, HTTP/80, HTTPS/443, DNS/53) pour faire transiter notre trafic. Si un firewall bloque le port 4444, on encapsule notre reverse shell dans du SSH ou du HTTP pour contourner la restriction.

### Les termes du pivoting

L'hote compromis qui sert de relais porte plusieurs noms selon le contexte :

- **Pivot host** : terme generique
- **Jump host** : terme reseau (bastion)
- **Foothold** : premier acces dans le reseau
- **Proxy** : quand il fait transiter le trafic

### Methodologie

```bash
# 1 - Compromettre un hote accessible
# 2 - Enumerer les interfaces reseau (ifconfig / ipconfig)
# 3 - Identifier les reseaux internes joignables
# 4 - Etablir un tunnel (SSH, Chisel, Meterpreter, etc.)
# 5 - Scanner le reseau interne via le tunnel
# 6 - Pivoter vers les cibles identifiees
# 7 - Repeter pour chaque segment reseau decouvert
```

## En pratique

```bash
# 1 - Apres compromission, verifier les interfaces
ifconfig  # ou ipconfig sur Windows

# 2 - Verifier la table de routage
netstat -r

# 3 - Identifier les hotes actifs sur le reseau interne
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done

# 4 - Etablir un tunnel SSH dynamique
ssh -D 9050 user@<IP_PIVOT>

# 5 - Scanner via proxychains
proxychains nmap -sT -Pn 172.16.5.19
```

## Pieges et galeres

- **Proxychains et scans partiels** : proxychains ne supporte que les scans TCP complets (`-sT`). Les scans SYN (`-sS`), UDP et les ping sweeps ICMP ne fonctionnent pas a travers un proxy SOCKS
- **Firewall Windows et ICMP** : Windows Defender bloque les requetes ICMP par defaut. Un ping sweep peut donner zero resultat meme si les hotes sont la. Utiliser un scan TCP sur les ports courants a la place
- **Latence** : chaque saut de pivot ajoute de la latence. Les scans Nmap complets sur un sous-reseau entier via un tunnel sont tres lents. Scanner des hotes individuels ou des plages reduites
- **Double NAT** : certaines configurations reseau utilisent du NAT a plusieurs niveaux. Documenter chaque saut avec un schema reseau aide a ne pas se perdre

## Retour terrain

Le pivoting est la competence qui fait la difference entre un pentest superficiel et un pentest complet. Sur la plupart des missions internes, le chemin vers le Domain Controller passe par au moins un pivot. Le schema classique : compromettre un serveur web dual-homed, pivoter vers le reseau serveur, puis atteindre le DC.

L'outil Draw.io (ou diagrams.net) est un allie precieux pour documenter la topologie au fur et a mesure des decouvertes. Sur un engagement complexe avec 3 ou 4 sauts de pivot, le schema reseau est indispensable pour ne pas perdre le fil.

## Memo express

| Concept | Description |
|---|---|
| Pivoting | Traverser les segments reseau via un hote compromis |
| Tunneling | Encapsuler le trafic dans un protocole autorise |
| Mouvement lateral | Se deplacer au sein d'un meme segment |
| Pivot host | Hote dual-homed servant de relais |
| Proxychains | Redirige le trafic TCP via un proxy SOCKS |
| `-sT -Pn` | Flags Nmap obligatoires pour les scans via proxy |

***
