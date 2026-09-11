# Introduction a l'enumeration et aux attaques AD

Active Directory (AD) reste le socle d'identite et d'acces de la majorite des environnements Windows d'entreprise. Sa complexite, combinee aux mauvaises configurations courantes, en fait une cible privilegiee lors de tests d'intrusion internes. Ce module couvre l'ensemble de la chaine d'attaque AD, de l'enumeration initiale sans identifiants jusqu'a la compromission du domaine.

## Pourquoi

Environ 43% des entreprises utilisent AD pour la gestion des identites et des acces. Microsoft cumule plus de 2000 CVE ces dernieres annees. AD est concu pour faciliter l'acces a l'information, ce qui le rend difficile a securiser correctement. En pentest, c'est souvent le chemin le plus rapide vers le Domain Admin, meme quand aucune vulnerabilite logicielle exploitable n'existe sur le perimetre.

## Comment ca marche

### Qu'est-ce qu'Active Directory

AD est un service d'annuaire hierarchique qui centralise la gestion des ressources d'une organisation : utilisateurs, ordinateurs, groupes, partages, GPO, approbations. Il fournit trois fonctions essentielles :

- **Authentification** : verifier l'identite des utilisateurs (Kerberos, NTLM)
- **Autorisation** : controler l'acces aux ressources (ACL, groupes, GPO)
- **Comptabilite** : tracer les actions (logs d'evenements, audit)

### Scenarios d'attaque reels

{% tabs %}
{% tab title="Kerberoasting + SCF" %}
Sur un engagement, un premier acces SYSTEM sur un hote joint au domaine a permis d'extraire des TGS Kerberos (Kerberoasting). Apres cracking d'un ticket, les identifiants obtenus donnaient un acces en ecriture sur des partages. Un fichier SCF depose sur ces partages a force un utilisateur Domain Admin a s'authentifier, capturant son hash NetNTLMv2 via Responder. Compromission totale.
{% endtab %}
{% tab title="Password Spraying" %}
Une session SMB NULL a permis de recuperer la liste des utilisateurs et la politique de mots de passe. Apres plusieurs tentatives calibrees (sans verrouillage), le mot de passe `Spring@18` a fonctionne sur un compte. BloodHound a revele que ce compte avait des droits d'admin local sur un hote ou un Domain Admin avait une session active. Extraction du TGT Kerberos avec Rubeus, puis pass-the-ticket vers le DA.
{% endtab %}
{% tab title="Enumeration creative" %}
Sans aucun acces initial, Kerbrute a permis d'enumerer 516 utilisateurs valides a partir de listes LinkedIn et de wordlists. Un password spray avec `Welcome2021` a donne un premier compte. BloodHound a revele que tous les utilisateurs du domaine avaient un acces RDP sur un poste. Depuis ce poste, un second spray a donne plusieurs comptes, dont un membre du Help Desk avec des droits GenericAll sur Enterprise Key Admins. Escalade via Shadow Credentials jusqu'au DCSync.
{% endtab %}
{% endtabs %}

### Boite a outils

| Outil | Plateforme | Usage principal |
|---|---|---|
| BloodHound / SharpHound | Cross-platform | Cartographie des chemins d'attaque AD |
| Responder | Linux | Poisoning LLMNR/NBT-NS, capture de hashes |
| Inveigh | Windows | Equivalent de Responder en PowerShell/C# |
| Kerbrute | Cross-platform | Enumeration d'utilisateurs et password spraying via Kerberos |
| Impacket | Linux | Suite d'outils AD (GetUserSPNs, secretsdump, psexec, etc.) |
| CrackMapExec / NetExec | Linux | Enumeration et attaques via SMB, WinRM, LDAP, MSSQL |
| Rubeus | Windows | Manipulation de tickets Kerberos |
| PowerView / SharpView | Windows | Enumeration AD detaillee |
| Mimikatz | Windows | Extraction de credentials, pass-the-hash, pass-the-ticket |
| ldapsearch / windapsearch | Linux | Requetes LDAP manuelles |
| enum4linux-ng | Linux | Enumeration SMB/RPC |
| DomainPasswordSpray | Windows | Password spraying interne |
| Hashcat | Cross-platform | Cracking de hashes offline |
| PingCastle | Windows | Audit de securite AD |

{% hint style="info" %}
Sur la plupart des distributions offensives, la majorite de ces outils sont preinstalles. Pour un poste Windows, les outils se trouvent generalement dans `C:\Tools`. L'important est de savoir utiliser les deux plateformes : certaines situations (managed workstation, VDI) imposent de travailler exclusivement depuis Windows.
{% endhint %}

## En pratique

La methodologie d'attaque AD suit une progression iterative :

```
1. Enumeration passive (ecoute reseau, Wireshark, Responder -A)
2. Enumeration active sans identifiants (Nmap, Kerbrute, SMB NULL)
3. Obtention d'un premier acces (poisoning, password spraying)
4. Enumeration authentifiee (BloodHound, PowerView, LDAP)
5. Escalade de privileges (Kerberoasting, ACL abuse, misconfigs)
6. Mouvement lateral (pass-the-hash, pass-the-ticket)
7. Compromission du domaine (DCSync, Golden Ticket)
```

Chaque etape alimente la suivante. Les identifiants obtenus a l'etape 3 debloquent l'enumeration de l'etape 4, qui revele les chemins d'attaque pour l'etape 5. C'est un processus cyclique : on revient souvent en arriere pour enumerer a nouveau avec les nouvelles informations obtenues.

## Retour terrain

L'AD est immense. On ne le maitrise pas en une nuit. La cle est de developper une methodologie repeatable : toujours commencer par les memes etapes, documenter chaque decouverte, et ne pas se precipiter sur la premiere piste sans avoir termine l'enumeration de base. En pentest, les chemins les plus interessants sont souvent ceux qu'on decouvre en combinant plusieurs elements apparemment insignifiants (un compte avec un SPN, une ACL mal configuree, une session active sur le mauvais hote).

## Memo express

| Phase | Objectif | Outils cles |
|---|---|---|
| Recon externe | Emails, format username, breach data | OSINT, LinkedIn, Dehashed |
| Enum passive | Identifier les hotes et services | Wireshark, Responder -A |
| Enum active | Utilisateurs, DC, services | Nmap, Kerbrute, enum4linux |
| Foothold | Premier acces authentifie | Responder, password spraying |
| Enum authentifiee | Chemins d'attaque | BloodHound, PowerView, LDAP |
| Escalade | Domain Admin | Kerberoasting, ACL abuse, DCSync |

***
