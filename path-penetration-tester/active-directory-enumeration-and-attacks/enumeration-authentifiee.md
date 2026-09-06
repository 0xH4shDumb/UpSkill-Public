# Enumeration authentifiee

Une fois un compte de domaine obtenu (via poisoning, password spraying ou fourni par le client), l'enumeration change de dimension. On accede a l'annuaire LDAP complet, aux partages, aux GPO, et on peut cartographier l'ensemble des chemins d'attaque avec BloodHound. C'est l'etape qui transforme un acces basique en plan d'attaque.

## Pourquoi

Un compte de domaine standard, meme sans privilege particulier, permet d'enumerer la quasi-totalite de l'AD : utilisateurs, groupes, ACL, GPO, sessions actives, SPN, delegations. Ces informations revelent les chemins d'escalade que l'enumeration anonyme ne peut pas decouvrir. C'est la difference entre deviner et savoir ou aller.

## Comment ca marche

### BloodHound : cartographie des chemins d'attaque

BloodHound collecte les donnees AD et les visualise sous forme de graphe. Les chemins d'attaque deviennent visibles.

{% tabs %}
{% tab title="Collecte depuis Linux" %}
```bash
# - bloodhound-python (collecteur Python)
bloodhound-python -u '<USER>' -p '<PASSWORD>' \
    -d INLANEFREIGHT.LOCAL -ns <IP_DC> -c All

# - Les fichiers JSON sont generes dans le repertoire courant
# Importer dans l'interface BloodHound
```
{% endtab %}
{% tab title="Collecte depuis Windows" %}
```powershell
# - SharpHound (collecteur C#)
.\SharpHound.exe -c All --zipfilename bloodhound_data

# - Ou via PowerShell
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\temp
```
{% endtab %}
{% endtabs %}

Requetes BloodHound utiles :

- **Find Shortest Paths to Domain Admins** : chemin le plus court vers le DA
- **Find All Kerberoastable Users** : comptes avec un SPN (cibles Kerberoasting)
- **Find Computers Where Domain Users Are Local Admin** : postes ou tous les utilisateurs sont admin
- **Shortest Paths from Owned Principals** : chemins depuis les comptes qu'on controle

### Enumeration avec PowerView

{% tabs %}
{% tab title="Utilisateurs et groupes" %}
```powershell
# - Importer PowerView
Import-Module .\PowerView.ps1

# - Lister tous les utilisateurs du domaine
Get-DomainUser | Select-Object samaccountname, description, memberof

# - Chercher des mots de passe dans les descriptions
Get-DomainUser | Where-Object {$_.description -ne $null} |
    Select-Object samaccountname, description

# - Enumerer les groupes privilegies
Get-DomainGroup -Identity "Domain Admins" -Recurse |
    Select-Object MemberName

# - Enumerer les groupes imbriques
Get-DomainGroupMember -Identity "Enterprise Admins" -Recurse
```
{% endtab %}
{% tab title="Ordinateurs et sessions" %}
```powershell
# - Lister les ordinateurs du domaine
Get-DomainComputer | Select-Object dnshostname, operatingsystem

# - Trouver les sessions actives (ou sont connectes les admins)
Find-DomainUserLocation

# - Trouver les postes ou on est admin local
Find-LocalAdminAccess
```
{% endtab %}
{% tab title="Partages et GPO" %}
```powershell
# - Enumerer les partages accessibles
Find-DomainShare -CheckShareAccess

# - Chercher des fichiers sensibles sur les partages
Find-InterestingDomainShareFile

# - Enumerer les GPO
Get-DomainGPO | Select-Object displayname, gpcfilesyspath
```
{% endtab %}
{% endtabs %}

### Enumeration depuis Linux

```bash
# - CrackMapExec : enum rapide
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' --shares
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' --users
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' --groups

# - Rechercher des fichiers sensibles sur les partages
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' \
    -M spider_plus --share 'SYSVOL'

# - LDAP : requetes detaillees
ldapsearch -x -H ldap://<IP_DC> -D '<USER>@INLANEFREIGHT.LOCAL' \
    -w '<PASSWORD>' -b "DC=INLANEFREIGHT,DC=LOCAL" \
    "(objectClass=user)" sAMAccountName description memberOf

# - Snaffler : chasseur de credentials sur les partages
# (version Windows recommandee pour les performances)
```

### Controles de securite a identifier

| Controle | Comment le detecter | Impact sur le pentest |
|---|---|---|
| Windows Defender / AV | `Get-MpComputerStatus` | Outils detectes, need obfuscation |
| AppLocker | `Get-AppLockerPolicy -Effective` | Restrictions d'execution |
| PowerShell Constrained Language | `$ExecutionContext.SessionState.LanguageMode` | Cmdlets limitees |
| LAPS (Local Admin Password Solution) | `Get-DomainComputer \| Select-Object ms-mcs-admpwd` | Mots de passe admin locaux uniques |
| Credential Guard | Registre ou systeminfo | Mimikatz bloque |
| SMB Signing | `crackmapexec smb <IP> --gen-relay-list` | Relay SMB impossible si actif |

### Living off the Land

Quand on est sur un poste Windows sans pouvoir importer d'outils, les commandes natives suffisent pour une enumeration de base.

```cmd
# - Informations sur le domaine
systeminfo | findstr /B "Domain"
set userdomain
echo %logonserver%

# - Enumeration des utilisateurs et groupes
net user /domain
net group /domain
net group "Domain Admins" /domain
net accounts /domain

# - Enumeration reseau
arp -a
route print
netstat -ano

# - Enumeration AD via dsquery (si disponible)
dsquery user -limit 0
dsquery computer -limit 0
dsquery group -name "Domain Admins"
```

```powershell
# - Module ActiveDirectory (si RSAT installe)
Import-Module ActiveDirectory
Get-ADUser -Filter * -Properties * | Select-Object Name, SamAccountName, Description
Get-ADGroup -Filter * | Select-Object Name
Get-ADDomain
```

## En pratique

```bash
# Workflow post-foothold

# 1 - Valider les identifiants et verifier les droits
crackmapexec smb <IP_DC> -u '<USER>' -p '<PASSWORD>'
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>'

# 2 - Collecter les donnees BloodHound
bloodhound-python -u '<USER>' -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL \
    -ns <IP_DC> -c All

# 3 - Enumerer les partages
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' --shares

# 4 - Chercher des comptes Kerberoastable
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' -dc-ip <IP_DC>

# 5 - Chercher des comptes ASREPRoastable
GetNPUsers.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' -dc-ip <IP_DC>

# 6 - Analyser les resultats BloodHound
# Importer les JSON, marquer les comptes owned, chercher les chemins
```

## Pieges et galeres

- **BloodHound trop bruyant** : la collecte SharpHound genere beaucoup de trafic LDAP et de connexions SMB. En mode evasif, utiliser `-c DCOnly` pour limiter la collecte au DC
- **Descriptions qui contiennent des mots de passe** : verifier systematiquement. C'est etonnamment courant, surtout pour les comptes de service
- **Sessions fantomes** : BloodHound peut montrer des sessions qui n'existent plus. Les donnees ne sont qu'un snapshot a un instant T
- **PowerShell bloque** : si Constrained Language Mode est actif, basculer sur les commandes `net` et `dsquery`, ou utiliser un binaire .NET (SharpView)

## Retour terrain

L'enumeration authentifiee est la phase ou le pentest prend vraiment forme. BloodHound est l'outil qui a revolutionne le pentest AD : il revele des chemins d'attaque que personne ne trouverait manuellement. Les descriptions de comptes avec des mots de passe en clair, les groupes imbriques avec des privileges inattendus, les GPO qui deploient des scripts avec des credentials en clair, tout se decouvre a cette etape.

La combinaison CrackMapExec + BloodHound + PowerView couvre 95% des besoins d'enumeration. Le reste est du Living off the Land quand les outils sont bloques.

## Memo express

| Technique | Outil Linux | Outil Windows |
|---|---|---|
| Cartographie AD | bloodhound-python | SharpHound |
| Enum utilisateurs | crackmapexec --users | Get-DomainUser (PowerView) |
| Enum groupes | crackmapexec --groups | Get-DomainGroup (PowerView) |
| Enum partages | crackmapexec --shares | Find-DomainShare (PowerView) |
| Sessions admin | N/A | Find-DomainUserLocation |
| Admin local | crackmapexec (bruteforce) | Find-LocalAdminAccess |
| Controles secu | N/A | Get-MpComputerStatus, AppLocker |

***
