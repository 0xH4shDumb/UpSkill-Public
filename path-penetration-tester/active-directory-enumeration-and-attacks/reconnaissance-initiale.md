# Reconnaissance et enumeration initiale

Avant de toucher a un clavier en interne, la reconnaissance externe fournit des elements critiques pour la suite du test : format des usernames, breach data, services exposes. En interne, l'enumeration sans identifiants permet d'identifier les hotes, les services et les premieres pistes pour obtenir un foothold.

## Pourquoi

Un pentest AD ne commence pas par un exploit. Il commence par la comprehension de l'environnement : qui sont les utilisateurs, ou sont les Domain Controllers, quels services sont exposes, quelle est la politique de mots de passe. Ces informations guident toutes les etapes suivantes. Sans enumeration methodique, on passe a cote de chemins d'attaque evidents.

## Comment ca marche

### Reconnaissance externe

Avant meme d'avoir un acces interne, certaines informations sont accessibles publiquement.

| Source | Ce qu'on cherche |
|---|---|
| BGP Toolkit / ARIN / RIPE | Plages IP, ASN, infrastructure |
| LinkedIn | Format des noms d'utilisateurs, technologies utilisees |
| Google Dorks | Documents PDF/DOCX (metadonnees avec usernames) |
| GitHub | Credentials dans le code, configurations exposees |
| HaveIBeenPwned / Dehashed | Breach data avec emails et mots de passe |

```bash
# - Enumerer les sous-domaines
dig axfr inlanefreight.com @ns1.inlanefreight.com

# - Rechercher des documents avec metadonnees
# Google Dork : site:inlanefreight.com filetype:pdf

# - Chercher des credentials sur GitHub
# Trufflehog pour scanner les repos
trufflehog git https://github.com/inlanefreight/
```

{% hint style="info" %}
Les metadonnees des documents PDF/Office contiennent souvent le champ "Author" avec le username interne (format `prenom.nom`, `pnom`, etc.). C'est suffisant pour construire une liste de usernames valides.
{% endhint %}

### Enumeration passive interne

Une fois sur le reseau interne (VM d'attaque, laptop branche, VPN), on commence par ecouter le trafic.

```bash
# - Ecouter le trafic avec Wireshark
sudo wireshark

# - Responder en mode analyse (passive, sans poisoning)
sudo responder -I eth0 -A
```

On capture des requetes ARP, MDNS, LLMNR et NBT-NS qui revelent les noms d'hotes, les adresses IP et parfois les noms de domaine.

### Enumeration active sans identifiants

```bash
# - Identifier les hotes actifs
sudo nmap -sn 172.16.5.0/23

# - Scanner les services sur les hotes decouverts
sudo nmap -sC -sV -p- 172.16.5.5

# - Identifier le Domain Controller
#   Un DC expose generalement : DNS (53), Kerberos (88), LDAP (389/636),
#   SMB (445), RPC (135), GC (3268/3269)
```

{% tabs %}
{% tab title="Enumeration SMB" %}
```bash
# - Tester une session NULL
smbclient -N -L //<IP_DC>

# - Enumeration RPC anonyme
rpcclient -U "" -N <IP_DC>
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo

# - enum4linux-ng (plus complet)
enum4linux-ng -A <IP_DC>
```
{% endtab %}
{% tab title="Enumeration LDAP" %}
```bash
# - Bind anonyme LDAP
ldapsearch -x -H ldap://<IP_DC> -s base namingcontexts

# - Si le bind anonyme fonctionne, enumerer les utilisateurs
ldapsearch -x -H ldap://<IP_DC> -b "DC=INLANEFREIGHT,DC=LOCAL" \
    "(objectClass=user)" sAMAccountName

# - windapsearch (plus pratique)
python3 windapsearch.py -d inlanefreight.local --dc-ip <IP_DC> -U
```
{% endtab %}
{% tab title="Enumeration Kerberos" %}
```bash
# - Enumerer les utilisateurs valides via Kerberos (pre-auth)
kerbrute userenum -d inlanefreight.local \
    --dc <IP_DC> /opt/wordlists/jsmith.txt

# - Les utilisateurs valides retournent un AS-REP
# - Les utilisateurs invalides retournent KDC_ERR_C_PRINCIPAL_UNKNOWN
```
{% endtab %}
{% endtabs %}

### Points de donnees a collecter

| Element | Pourquoi |
|---|---|
| Liste d'utilisateurs | Cibles pour le password spraying |
| Politique de mots de passe | Calibrer le spray sans verrouiller les comptes |
| Domain Controllers | Cibles principales, services critiques |
| Serveurs critiques | SQL, Exchange, partages de fichiers |
| Format des usernames | Construire des listes pertinentes |
| Sous-reseaux | Cartographier l'infrastructure |

## En pratique

```bash
# Workflow complet d'enumeration initiale

# 1 - Ecoute passive (pendant qu'on fait le reste)
sudo responder -I eth0 -A &

# 2 - Decouverte des hotes
sudo nmap -sn 172.16.5.0/23 -oA discovery

# 3 - Identification du DC
sudo nmap -sC -sV -p 53,88,135,389,445,636,3268 172.16.5.5

# 4 - Enumeration anonyme
enum4linux-ng -A 172.16.5.5
rpcclient -U "" -N 172.16.5.5

# 5 - Enumeration Kerberos
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 users.txt

# 6 - Recuperer la politique de mots de passe (si session NULL)
crackmapexec smb 172.16.5.5 --pass-pol
```

## Pieges et galeres

{% tabs %}
{% tab title="Enumeration" %}
- **Session NULL refusee** : de plus en plus de DC modernes bloquent les sessions SMB anonymes. Passer par Kerberos (Kerbrute) pour enumerer les utilisateurs
- **LDAP anonymous bind desactive** : idem, essayer Kerberos comme alternative
- **Faux negatifs Nmap** : les firewalls internes peuvent bloquer les ping. Utiliser `-Pn` pour scanner sans decouverte d'hote
{% endtab %}
{% tab title="Reconnaissance" %}
- **Scope flou** : toujours clarifier le perimetre par ecrit avant de commencer. Scanner un hote hors scope peut avoir des consequences graves
- **VPN et poisoning** : si on est connecte via VPN, le poisoning LLMNR/NBT-NS ne fonctionne pas (pas dans le meme broadcast domain)
{% endtab %}
{% endtabs %}

## Retour terrain

L'enumeration initiale est souvent la phase la plus longue et la plus frustrante d'un pentest AD. Les sessions NULL sont de plus en plus rares, et les DC modernes bloquent beaucoup de techniques anonymes. Kerbrute est devenu l'outil de reference pour l'enumeration d'utilisateurs quand les methodes classiques echouent. La combinaison LinkedIn + Kerbrute + password spraying calibre est le chemin le plus fiable vers un premier acces.

## Memo express

| Technique | Commande | Prerequis |
|---|---|---|
| Decouverte hotes | `nmap -sn 172.16.5.0/23` | Acces reseau |
| Enum SMB anonyme | `enum4linux-ng -A <DC>` | Session NULL autorisee |
| Enum Kerberos | `kerbrute userenum -d DOMAIN --dc <DC> users.txt` | Aucun |
| Enum LDAP anonyme | `ldapsearch -x -H ldap://<DC> -b "DC=..." "(objectClass=user)"` | Bind anonyme autorise |
| Politique MDP | `crackmapexec smb <DC> --pass-pol` | Session NULL ou creds |
| Ecoute passive | `sudo responder -I eth0 -A` | Meme broadcast domain |

***
