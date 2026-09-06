# Durcissement et audit AD

Un pentest AD ne se termine pas par la compromission du domaine. Les recommandations de durcissement sont la partie la plus precieuse du rapport pour le client. Ce chapitre couvre les mesures defensives essentielles et les techniques d'audit pour evaluer la posture de securite AD.

## Pourquoi

Les entreprises investissent dans des pentests pour identifier leurs faiblesses et les corriger. Un rapport qui dit "on a eu le DA" sans expliquer comment se proteger est incomplet. En tant que pentester, maitriser les mesures defensives permet de formuler des recommandations precises et de mieux comprendre les obstacles qu'on rencontrera en environnement mature.

## Comment ca marche

### Recommandations prioritaires

| Priorite | Mesure | Attaques bloquees |
|---|---|---|
| Critique | **Desactiver LLMNR et NBT-NS** via GPO | Poisoning, capture de hashes |
| Critique | **Patcher les DC** regulierement | NoPac, PrintNightmare, PetitPotam |
| Critique | **Deployer LAPS** pour les mots de passe admin locaux | Pass-the-Hash lateral |
| Haute | **Activer SMB Signing** sur tous les hotes | Relay SMB |
| Haute | **Politique de mots de passe forte** (15+ caracteres, rotation) | Password spraying, Kerberoasting |
| Haute | **Comptes de service gMSA** au lieu de comptes manuels | Kerberoasting |
| Haute | **Protected Users group** pour les comptes privilegies | Delegation, credential theft |
| Moyenne | **Tiering model** (T0/T1/T2) | Mouvement lateral vers les DC |
| Moyenne | **Auditer les ACL** regulierement | ACL abuse |
| Moyenne | **Desactiver le Print Spooler** sur les DC | PrintNightmare |
| Moyenne | **Reduire ms-DS-MachineAccountQuota** a 0 | NoPac |

### Desactiver LLMNR et NBT-NS

```
LLMNR :
  Computer Configuration → Administrative Templates → Network → DNS Client
  → Turn OFF Multicast Name Resolution → Enabled

NBT-NS :
  Desactiver via DHCP (option 001) ou manuellement sur chaque interface
  (Network Adapter → IPv4 → Advanced → WINS → Disable NetBIOS over TCP/IP)
```

### Hardening Kerberos

- **Comptes gMSA** : mots de passe de 240 caracteres, rotation automatique toutes les 30 jours. Impossibles a Kerberoaster
- **AES uniquement** : desactiver RC4 pour les comptes de service (ralentit le cracking, mais attention a la compatibilite)
- **Pre-authentification obligatoire** : verifier que `DONT_REQUIRE_PREAUTH` n'est active sur aucun compte (ASREPRoasting)

```powershell
# - Trouver les comptes sans pre-auth
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} |
    Select-Object Name, SamAccountName

# - Trouver les comptes avec RC4 Kerberos
Get-ADUser -Filter {AllowReversiblePasswordEncryption -eq $True}
```

### Audit des ACL

```powershell
# - PingCastle (audit automatise)
.\PingCastle.exe --healthcheck --server <DC>

# - Rapport HTML avec score de risque et recommandations

# - ADRecon (extraction detaillee pour analyse)
.\ADRecon.ps1 -DomainController <DC> -Collect All
# Genere un rapport Excel avec toutes les donnees AD

# - Group3r (audit des GPO)
.\Group3r.exe -f <SYSVOL_PATH>
```

### Monitoring et detection

| Event ID | Description | Ce qu'il detecte |
|---|---|---|
| 4769 | Kerberos Service Ticket Request | Kerberoasting (etype 0x17 en masse) |
| 4768 | Kerberos TGT Request | ASREPRoasting, brute force Kerberos |
| 4625 | Failed Logon | Password spraying |
| 4662 | Directory Service Access | DCSync (GUIDs de replication) |
| 4672 | Special Privileges Assigned | Connexion avec privileges admin |
| 4724 | Password Reset Attempt | Reset de mot de passe non autorise |
| 5136 | Directory Service Object Modified | Modification d'ACL, SPN, delegation |

{% hint style="success" %}
La detection du Kerberoasting repose sur le monitoring de l'Event ID 4769 avec le code de chiffrement 0x17 (RC4). Un volume anormal de demandes TGS pour differents SPN depuis le meme compte en peu de temps est un indicateur fort.
{% endhint %}

### Tiering model (modele de tiers)

Le tiering model segmente l'environnement en niveaux :

- **Tier 0** : Domain Controllers, comptes DA, serveurs d'identite (AD CS, AD FS). Acces ultra restreint.
- **Tier 1** : Serveurs applicatifs, bases de donnees. Administres par des comptes dedies, differents du Tier 0.
- **Tier 2** : Postes de travail. Les admins Tier 2 ne se connectent jamais a un serveur Tier 1 ou Tier 0.

Le principe fondamental : un compte admin d'un tier superieur ne doit jamais s'authentifier sur un hote d'un tier inferieur. Pas de session DA sur un poste de travail.

## En pratique

```powershell
# Checklist d'audit rapide

# 1 - PingCastle
.\PingCastle.exe --healthcheck --server <DC>

# 2 - Comptes Kerberoastable
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} |
    Select-Object Name, ServicePrincipalName

# 3 - Comptes sans pre-auth
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True}

# 4 - ms-DS-MachineAccountQuota
Get-ADObject -Identity "DC=INLANEFREIGHT,DC=LOCAL" -Properties ms-DS-MachineAccountQuota

# 5 - Print Spooler sur les DC
Get-Service -Name Spooler -ComputerName <DC>

# 6 - LAPS deploye
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd |
    Where-Object {$_.'ms-Mcs-AdmPwd' -ne $null} | Measure-Object

# 7 - SMB Signing
# Depuis Linux :
crackmapexec smb 172.16.5.0/23 --gen-relay-list relay_targets.txt
```

## Retour terrain

PingCastle est devenu l'outil de reference pour l'audit AD. Son rapport HTML avec un score de risque est comprehensible par les equipes non techniques et permet de prioriser les actions de remediation. En pentest, le lancer apres la compromission donne un excellent support pour les recommandations du rapport.

Les mesures les plus impactantes en termes de cout/benefice : desactiver LLMNR/NBT-NS (gratuit, bloque le poisoning), deployer LAPS (gratuit, bloque le mouvement lateral via PtH), et passer les comptes de service en gMSA (effort moderate, bloque le Kerberoasting). Ces trois mesures seules elevent significativement le niveau de securite AD.

## Memo express

| Outil | Usage | Commande |
|---|---|---|
| PingCastle | Audit AD automatise | `PingCastle.exe --healthcheck --server <DC>` |
| ADRecon | Extraction de donnees AD | `ADRecon.ps1 -DomainController <DC>` |
| Group3r | Audit GPO | `Group3r.exe -f <SYSVOL>` |
| BloodHound | Cartographie des chemins d'attaque | `SharpHound.exe -c All` |
| Purple Knight | Audit complementaire | Interface graphique |

***
