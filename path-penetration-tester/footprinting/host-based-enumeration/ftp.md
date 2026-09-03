# FTP (File Transfer Protocol)

FTP fait partie des protocoles les plus anciens encore en service. On le croise régulièrement en test d'intrusion, souvent mal configuré, et il reste une source d'informations (voire un point d'entrée) qu'il ne faut pas négliger.

## Pourquoi

Malgré son âge, FTP reste déployé dans beaucoup d'environnements : partage de fichiers interne, dépôt de sauvegardes, transfert entre applications. Le problème fondamental du protocole, c'est qu'il transmet tout en clair, identifiants compris. Un accès anonyme mal restreint ou un serveur exposé sur Internet sans chiffrement, et on se retrouve avec une porte ouverte sur des données parfois critiques.

En pentest, FTP est souvent l'un des premiers services à vérifier : accès anonyme, bannière de version, fichiers accessibles. C'est rapide à tester et le retour sur investissement est bon.

## Comment ça marche

FTP fonctionne au niveau de la couche application TCP/IP, au même titre que HTTP ou SMTP. Il utilise **deux canaux séparés** pour communiquer :

- **Port 21 (canal de commande)** : le client envoie ses instructions (connexion, listing, téléchargement) et reçoit les réponses du serveur.
- **Port 20 (canal de données)** : utilisé exclusivement pour le transfert effectif des fichiers.

### Modes de connexion

{% tabs %}
{% tab title="Mode actif" %}
En mode actif, le client informe le serveur du port sur lequel il attend la connexion de données. Le serveur initie alors la connexion vers le client.

Le problème : les pare-feux côté client bloquent souvent ces connexions entrantes non sollicitées. C'est pour cette raison que le mode actif pose problème dans la plupart des environnements modernes.
{% endtab %}

{% tab title="Mode passif" %}
En mode passif, c'est le serveur qui fournit un port au client pour établir la connexion de données. Le client se connecte au serveur, pas l'inverse.

C'est le mode privilégié dans les environnements protégés par un pare-feu, et c'est celui que la majorité des clients FTP utilisent par défaut aujourd'hui.
{% endtab %}
{% endtabs %}

### Authentification et accès anonyme

Par défaut, FTP exige un couple identifiant/mot de passe. Mais certains serveurs autorisent un accès **anonyme** : on se connecte avec le login `anonymous` et un mot de passe vide (ou une adresse email arbitraire). Cet accès permet souvent de lire le contenu du serveur, et parfois même d'y écrire.

{% hint style="danger" %}
FTP transmet les identifiants et les données en clair. Sur un réseau non segmenté, une simple capture de trafic suffit pour intercepter des credentials. C'est un vecteur classique de compromission en environnement interne.
{% endhint %}

### TFTP : la variante minimaliste

TFTP (Trivial File Transfer Protocol) est une version allégée de FTP, conçue pour des usages très spécifiques :

- Fonctionne sur **UDP** (et non TCP)
- Aucune authentification
- Pas de navigation dans les répertoires
- Limité aux réseaux locaux de confiance

On le croise principalement pour le boot réseau (PXE) ou la mise à jour de firmware sur des équipements réseau. Les commandes disponibles se limitent à `connect`, `get`, `put`, `quit`, `status` et `verbose`.

### Configuration serveur (vsFTPd)

Sur les systèmes Linux, **vsFTPd** (Very Secure FTP Daemon) est l'implémentation la plus courante. Sa configuration se trouve dans `/etc/vsftpd.conf`.

| Directive | Effet |
|---|---|
| `anonymous_enable=NO` | Désactive l'accès anonyme |
| `local_enable=YES` | Autorise l'authentification par comptes locaux |
| `write_enable=YES` | Active les permissions d'écriture |
| `chroot_local_user=YES` | Confine chaque utilisateur dans son répertoire personnel |
| `hide_ids=YES` | Masque les UID/GID dans les listings de fichiers |

{% hint style="info" %}
Le fichier `/etc/ftpusers` contient la liste des utilisateurs **interdits** de connexion FTP. Sur beaucoup de systèmes, root y figure par défaut.
{% endhint %}

## En pratique

### Connexion anonyme

```bash
# Depuis Exegol - tentative de connexion anonyme
ftp <IP_CIBLE>
Name: anonymous
Password:
```

Si l'accès anonyme est activé, on obtient un shell FTP. Les commandes de base :

```bash
ftp> ls          # - listing du répertoire courant
ftp> ls -R       # - listing récursif
ftp> get fichier.txt   # - télécharger un fichier
ftp> put payload.txt   # - uploader un fichier (si l'écriture est permise)
```

### Téléchargement massif

Pour récupérer l'intégralité d'un serveur FTP accessible en anonyme :

```bash
# Depuis Exegol - téléchargement récursif complet
wget -m --no-passive ftp://anonymous:anonymous@<IP_CIBLE>
```

L'option `-m` (mirror) télécharge de manière récursive en conservant l'arborescence.

### Scan et énumération avec Nmap

```bash
# Depuis Exegol - scan de version + scripts par défaut
sudo nmap -sV -sC -p21 <IP_CIBLE>
```

Les scripts NSE les plus utiles pour FTP :

| Script | Fonction |
|---|---|
| `ftp-anon` | Détecte si l'accès anonyme est autorisé |
| `ftp-syst` | Récupère les informations système du serveur |
| `ftp-vsftpd-backdoor` | Teste la présence d'une backdoor connue dans certaines versions de vsFTPd |

Pour tracer les échanges au niveau des paquets :

```bash
# Depuis Exegol - scan avec trace des paquets
sudo nmap -sV -sC -p21 <IP_CIBLE> --script-trace
```

### Connexion chiffrée et interaction manuelle

Quand FTPS (FTP over TLS) est en place, on peut inspecter le certificat et les métadonnées avec OpenSSL :

```bash
# Depuis Exegol - connexion TLS sur le canal de commande
openssl s_client -connect <IP_CIBLE>:21 -starttls ftp
```

Le certificat peut révéler le nom de l'organisation, des noms d'hôtes internes, ou d'autres métadonnées utiles.

Pour une interaction manuelle brute sans chiffrement :

```bash
# Depuis Exegol - connexion directe via netcat
nc -nv <IP_CIBLE> 21
```

```bash
# Depuis Exegol - connexion directe via telnet
telnet <IP_CIBLE> 21
```

## Pièges et galères

{% hint style="warning" %}
**Accès anonyme ne veut pas dire accès total.** Certains serveurs FTP autorisent la connexion anonyme mais restreignent fortement les répertoires visibles. Il faut tester les sous-dossiers et vérifier les permissions d'écriture, pas se contenter du listing racine.
{% endhint %}

{% hint style="danger" %}
**Transmission en clair.** C'est le risque principal du protocole. Même avec des identifiants valides, utiliser FTP sans TLS sur un réseau partagé expose les credentials à toute personne capable de capturer du trafic. En pentest, c'est une trouvaille à signaler systématiquement.
{% endhint %}

{% hint style="warning" %}
**Mode actif vs pare-feu.** Si une connexion FTP s'établit mais que les commandes `ls` ou `get` échouent, c'est souvent un problème de mode actif/passif. Tester avec `-p` (mode passif) dans le client FTP ou `--no-passive` avec `wget` selon le contexte.
{% endhint %}

## Retour terrain

{% hint style="success" %}
Toujours commencer par un test d'accès anonyme. C'est la vérification la plus rapide et celle qui rapporte le plus souvent. Un `ftp-anon` positif dans un scan Nmap suffit pour justifier une investigation plus poussée.
{% endhint %}

- En environnement interne, les serveurs FTP sont souvent des reliques d'une époque où la sécurité réseau n'était pas une priorité. On y trouve régulièrement des sauvegardes, des scripts avec des credentials en dur, ou des fichiers de configuration.
- La bannière du serveur FTP est généralement bavarde : version du daemon, parfois le nom d'hôte ou l'OS. C'est une information utile pour identifier des vulnérabilités connues.
- Quand le serveur supporte TLS, le certificat est une mine d'informations passives (noms de domaines, organisation, dates de validité).

## Mémo express

| Commande | Fonction |
|---|---|
| `ftp <IP_CIBLE>` | Connexion au serveur FTP |
| `ftp> ls -R` | Listing récursif |
| `ftp> get fichier` | Télécharger un fichier |
| `ftp> put fichier` | Uploader un fichier |
| `wget -m --no-passive ftp://anonymous:anonymous@<IP>` | Téléchargement miroir complet |
| `sudo nmap -sV -sC -p21 <IP_CIBLE>` | Scan de version + scripts NSE |
| `openssl s_client -connect <IP>:21 -starttls ftp` | Inspection du certificat TLS |
| `nc -nv <IP_CIBLE> 21` | Interaction manuelle brute |

***
