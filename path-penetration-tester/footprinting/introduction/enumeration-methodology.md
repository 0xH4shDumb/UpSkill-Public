# Methodologie d'enumeration

Avoir des outils et des techniques, c'est necessaire. Mais sans methodologie structuree, on finit par tourner en rond, tester les memes choses plusieurs fois, et passer a cote de chemins d'attaque evidents. Cette page presente un cadre en six couches qui permet d'organiser la phase de footprinting de maniere logique et reproductible.

## Pourquoi

Un test d'intrusion a une duree limitee. Quatre semaines, parfois moins. Dans ce cadre contraint, il faut etre methodique pour maximiser la couverture sans perdre de temps sur des impasses. Sans structure, on se retrouve a scanner des ports au hasard, a lancer des exploits sans comprendre le contexte, et a rater des vecteurs d'attaque parce qu'on n'a pas regarde au bon endroit.

La methodologie ne garantit pas de tout trouver. Mais elle garantit de ne pas gaspiller le temps disponible.

## Comment ca marche

### Le modele en six couches

L'enumeration se decompose en six couches successives. Chaque couche represente un niveau de profondeur dans la connaissance de la cible. On peut les voir comme des parois a traverser pour se rapprocher du coeur de l'infrastructure.

| Couche | Nom | Ce qu'on cherche | Exemples d'informations |
|---|---|---|---|
| 1 | Presence Internet | La surface publique de l'entreprise | Domaines, sous-domaines, IPs, ASN, netblocks, vHosts |
| 2 | Passerelle | Les dispositifs de protection | Pare-feu, DMZ, IDS/IPS, VPN, proxy, segmentation |
| 3 | Services accessibles | Les interfaces exposees | Ports, protocoles, versions, bannières |
| 4 | Processus | Les flux internes | PIDs, donnees traitees, communications inter-services |
| 5 | Privileges | Les permissions en place | Groupes, utilisateurs, restrictions, ACLs |
| 6 | Configuration OS | L'environnement systeme | Type d'OS, patchs, fichiers sensibles, configuration reseau |

{% hint style="info" %}
Les couches 1 et 2 sont surtout pertinentes pour les tests externes. En interne (typiquement en environnement Active Directory), on demarre souvent directement a la couche 3.
{% endhint %}

### La metaphore du labyrinthe

Chaque test d'intrusion peut se voir comme un labyrinthe dont les murs correspondent aux couches. Une vulnerabilite, c'est une fissure dans un mur. Certaines fissures ne menent nulle part, d'autres ouvrent un passage direct vers les couches suivantes. Le travail du pentester est de tester chaque fissure pour determiner laquelle offre la progression la plus efficace, pas de s'acharner sur le mur le plus epais.

## En pratique

Prenons le cas d'un test d'intrusion externe en boite noire. Voici comment les couches s'appliquent concretement :

### Couche 1 : Presence Internet

On commence par identifier la surface d'attaque publique. C'est de la reconnaissance, principalement passive au debut.

- Recherche de domaines et sous-domaines (crt.sh, amass, subfinder)
- Identification des plages IP et ASN (whois, BGP)
- Detection de services cloud (S3, Azure Blob)
- Inventaire des vHosts et des technologies front-end

### Couche 2 : Passerelle

On cherche a comprendre ce qui protege les services identifies. Pas pour les contourner tout de suite, mais pour adapter la strategie.

- Detection de pare-feux (ACK scan, analyse des reponses)
- Identification de WAF, reverse proxies, load balancers
- Test de filtrage par port et par protocole

### Couche 3 : Services accessibles

C'est la couche la plus dense en termes d'enumeration. On identifie et on caracterise chaque service expose.

- Scan de ports (Nmap, masscan)
- Recuperation de bannieres et fingerprinting
- Identification des versions et des configurations

{% hint style="success" %}
La majorite du travail d'enumeration concret se passe a cette couche. C'est ici qu'on decouvre les services FTP, SMB, DNS, SMTP, bases de donnees, et tout ce qui fait l'objet des pages suivantes de ce module.
{% endhint %}

### Couche 4 : Processus

Une fois un acces obtenu (ou en analysant les services de l'exterieur), on s'interesse aux flux de donnees.

- Quels services communiquent entre eux ?
- Quels fichiers sont transferes, et entre quelles machines ?
- Y a-t-il des taches planifiees ou des jobs de synchronisation ?

### Couche 5 : Privileges

On analyse les droits associes aux services et aux comptes decouverts.

- Quels utilisateurs et groupes existent ?
- Quelles restrictions sont en place ?
- Un compte de service a-t-il des privileges excessifs ?

### Couche 6 : Configuration OS

La derniere couche concerne l'environnement systeme de la machine compromise ou analysee.

- Type et version de l'OS
- Niveau de patch
- Configuration reseau (interfaces, routes, DNS)
- Fichiers sensibles (credentials, cles, logs)

## Retour terrain

La valeur de cette methodologie, c'est qu'elle empeche de se disperser. Quand on bloque a une couche, on remonte a la precedente pour chercher un angle d'attaque different. Quand on progresse trop vite, on prend le temps de consolider les informations collectees avant de passer a la suite.

{% hint style="warning" %}
Cette methodologie n'est pas un processus lineaire rigide. C'est un cadre de reference. Sur le terrain, on navigue constamment entre les couches en fonction de ce qu'on decouvre. L'important, c'est de ne jamais perdre de vue dans quelle couche on se trouve et ce qu'il reste a explorer.
{% endhint %}

## Memo express

| Couche | Question cle |
|---|---|
| 1 - Presence Internet | Quelle est la surface publique de la cible ? |
| 2 - Passerelle | Qu'est-ce qui protege les services ? |
| 3 - Services | Quels services sont exposes et dans quelle version ? |
| 4 - Processus | Quels flux de donnees circulent entre les services ? |
| 5 - Privileges | Quels droits sont associes aux comptes et services ? |
| 6 - OS | Quel est l'environnement systeme sous-jacent ? |

***
