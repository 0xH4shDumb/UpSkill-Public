# Environnement de travail

Un pentest commence bien avant le premier scan. Le choix de la distribution, la configuration du VPN, l'organisation des fichiers et la prise de notes conditionnent l'efficacite de chaque engagement. Cette page couvre la mise en place d'un environnement de travail solide.

## Pourquoi

Un environnement mal configure fait perdre du temps et peut compromettre la securite du pentester lui-meme. Travailler depuis une VM dediee, se connecter via VPN et maintenir une arborescence de fichiers structuree sont des prerequis pour tout engagement professionnel.

## Comment ca marche

### Distribution de pentest

Une distribution de pentest est un systeme Linux preconfigure avec les outils necessaires a la securite offensive. Les principales options sont les suivantes.

| Distribution | Base | Particularite |
|---|---|---|
| **Kali Linux** | Debian | La plus repandue, large communaute, mise a jour reguliere des outils |
| **Parrot Security** | Debian | Plus legere que Kali, interface soignee, outils similaires |
| **BlackArch** | Arch Linux | Enorme catalogue d'outils, destinee aux utilisateurs avances |
| **votre machine d'attaque** | Docker | Conteneurs preconfigures, isolation forte, deploiement rapide |

{% hint style="success" %}
votre machine d'attaque est particulierement adapte aux engagements professionnels. Chaque mission demarre dans un conteneur vierge, ce qui elimine le risque de contamination entre les environnements clients. L'installation se fait via `pip install votre machine d'attaque` puis `votre machine d'attaque install`.
{% endhint %}

### Virtualisation

La virtualisation permet de faire tourner la distribution de pentest dans un environnement isole. Un hyperviseur cree des machines virtuelles (VM) qui partagent les ressources de la machine hote tout en restant independantes.

| Hyperviseur | Type | OS hote |
|---|---|---|
| **VirtualBox** | Type 2 (logiciel) | Windows, Linux, macOS |
| **VMware Workstation** | Type 2 (logiciel) | Windows, Linux |
| **Proxmox** | Type 1 (bare metal) | Installation directe sur le materiel |
| **Hyper-V** | Type 1 integre | Windows 10/11 Pro, Windows Server |

{% hint style="warning" %}
Chaque engagement doit etre realise depuis une VM fraichement installee. Reutiliser la meme VM d'un client a l'autre risque de melanger des artefacts (credentials, captures, scripts) entre deux missions.
{% endhint %}

Deux formats d'installation sont disponibles pour la plupart des distributions. Le fichier **ISO** permet une installation personnalisee (partitionnement, clavier, locale). Le fichier **OVA** est une appliance preconstruite qui se deploie en quelques minutes dans l'hyperviseur.

### Connexion VPN

Un VPN (Virtual Private Network) cree un tunnel chiffre entre la machine du pentester et le reseau cible. La connexion s'etablit avec OpenVPN et un fichier de configuration `.ovpn` fourni par le client ou la plateforme de lab.

```bash
# - Connexion au VPN
sudo openvpn client.ovpn
```

Une fois connecte, une interface `tun0` apparait avec une adresse IP dans le reseau cible.

```bash
# - Verification de l'interface VPN
ip -4 a show tun0
```

```bash
# - Verification de la table de routage
netstat -rn
```

{% hint style="info" %}
La ligne `Initialization Sequence Completed` dans la sortie d'OpenVPN confirme que la connexion est etablie. Si cette ligne n'apparait pas, verifier les logs pour identifier l'erreur (certificat expire, port bloque, fichier de configuration incorrect).
{% endhint %}

### Organisation du travail

#### Arborescence de fichiers

Une structure de dossiers coherente evite de perdre des donnees et facilite la redaction du rapport. Voici un modele eprouve.

```
Projets/
└── Client-X/
    ├── EPT/                    # External Penetration Test
    │   ├── evidence/
    │   │   ├── credentials/
    │   │   ├── data/
    │   │   └── screenshots/
    │   ├── logs/
    │   ├── scans/
    │   ├── scope/
    │   └── tools/
    └── IPT/                    # Internal Penetration Test
        ├── evidence/
        │   ├── credentials/
        │   ├── data/
        │   └── screenshots/
        ├── logs/
        ├── scans/
        ├── scope/
        └── tools/
```

#### Prise de notes

Documenter chaque etape au fil de l'eau est indispensable. Reconstituer les actions apres coup est chronophage et source d'erreurs. Plusieurs outils sont adaptes a la prise de notes technique.

| Outil | Points forts |
|---|---|
| **Obsidian** | Notes en Markdown, liens bidirectionnels, plugins, 100% local |
| **CherryTree** | Hierarchie arborescente, captures d'ecran integrees |
| **Notion** | Collaboratif, bases de donnees, templates |
| **GitBook** | Documentation structuree, publication web integree |

{% hint style="danger" %}
Les donnees clients ne doivent jamais etre synchronisees vers le cloud sans autorisation explicite. Privilegier un stockage local chiffre ou un service auto-heberge pour les engagements professionnels.
{% endhint %}

### Ressources d'apprentissage

#### Applications volontairement vulnerables

Ces plateformes permettent de pratiquer dans un environnement controle sans risque legal.

| Plateforme | Description |
|---|---|
| **DVWA** | Application PHP/MySQL couvrant les principales vulnerabilites web |
| **OWASP Juice Shop** | Application Node.js moderne, couvre l'ensemble de l'OWASP Top 10 |
| **Metasploitable 2** | VM Ubuntu vulnerable pour la pratique de l'exploitation reseau |
| **Metasploitable 3** | VM Windows avec de nombreuses vulnerabilites configurables |
| **VulnHub** | Collection de VM vulnerables telechargeable pour un lab local |

#### Ressources en ligne

| Type | Exemples |
|---|---|
| **Chaines YouTube** | IppSec (walkthroughs detailles), LiveOverflow (concepts techniques), STOK (bug bounty) |
| **Blogs techniques** | 0xdf hacks stuff, HackTricks, PayloadsAllTheThings |
| **Wargames** | OverTheWire (Linux CLI), UnderTheWire (PowerShell) |
| **Certifications** | OSCP (Offensive Security), CPTS (Hack The Box), eJPT (eLearnSecurity) |

## En pratique

### Deploiement rapide avec votre machine d'attaque

```bash
# - Installation d'votre machine d'attaque
pip install votre machine d'attaque

# - Telechargement de l'image par defaut
votre machine d'attaque install

# - Lancement d'un conteneur avec connexion VPN
votre machine d'attaque start mission1 full --vpn client.ovpn
```

### Verification de la connectivite VPN

```bash
# - Verifier l'adresse IP attribuee
ip -4 a show tun0

# - Verifier l'acces au reseau cible
ping -c 3 <IP_CIBLE>

# - Verifier la table de routage
ip route
```

## Pieges et galeres

- **VPN qui se deconnecte silencieusement** : la connexion OpenVPN peut se couper sans message visible. Garder le terminal OpenVPN ouvert et verifier regulierement que l'interface `tun0` existe toujours
- **Plusieurs fichiers VPN actifs** : avoir deux connexions VPN simultanees provoque des conflits de routage. Toujours fermer la connexion precedente avant d'en ouvrir une nouvelle
- **VM trop legere en RAM** : une VM avec moins de 4 Go de RAM aura du mal a faire tourner certains outils (Burp Suite, BloodHound). Prevoir au minimum 4 Go, idealement 8 Go
- **Notes prises apres coup** : la memoire deforme les etapes. Documenter en temps reel, meme si les notes sont brutes. Il est toujours possible de les restructurer apres
- **Outils non mis a jour** : les bases de donnees d'exploits evoluent constamment. Mettre a jour les outils et les wordlists avant chaque engagement (`sudo apt update && sudo apt upgrade` ou `votre machine d'attaque update`)

## Memo express

| Element | Detail |
|---|---|
| **Distribution recommandee** | votre machine d'attaque (conteneurs Docker) ou Kali/Parrot (VM classique) |
| **Hyperviseur** | VirtualBox (gratuit), VMware Workstation (gratuit), Proxmox (bare metal) |
| **Connexion VPN** | `sudo openvpn client.ovpn`, verifier `tun0` |
| **Arborescence** | Un dossier par client, sous-dossiers par type de test |
| **Notes** | Obsidian ou CherryTree, en Markdown, documentees en temps reel |
| **VM fraiche** | Nouvelle VM pour chaque engagement, jamais de reutilisation |

***
