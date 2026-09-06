# Durcissement Windows

Apres avoir passe un module entier a exploiter des failles de configuration, des privileges excessifs et des services mal securises, il est essentiel de comprendre comment s'en proteger. Un durcissement methodique elimine la majorite des vecteurs d'escalade couverts dans les pages precedentes.

## Pourquoi

Chaque recommandation de cette page correspond directement a un vecteur d'attaque exploite dans ce module. Un pentester qui comprend les mesures defensives peut evaluer leur presence lors d'un audit, identifier les lacunes et formuler des recommandations actionnables. Le durcissement n'est pas un exercice theorique : il transforme un systeme exploitable en cible resistante.

## Comment ca marche

### Image de base securisee

Deployer des systemes a partir d'une image de base standardisee permet d'eliminer les logiciels preinstalles non necessaires (bloatware) et d'assurer une configuration homogene dans l'environnement. L'image doit inclure :

1. Les applications necessaires aux taches quotidiennes
2. Les configurations de securite validees pour l'environnement
3. Les mises a jour majeures et mineures testees et approuvees

| Outil | Usage |
|---|---|
| **WDS** (Windows Deployment Services) | Deploiement d'images via le reseau |
| **SCCM** (System Center Configuration Manager) | Gestion de configuration et deploiement |
| **MDT** (Microsoft Deployment Toolkit) | Creation et deploiement d'images personnalisees |

### Mises a jour et correctifs

Les exploits les plus impactants ciblent des versions connues de composants vulnerables. Un systeme non patche est vulnerable a EternalBlue, PrintNightmare, HiveNightmare et des dizaines d'autres CVE exploitables en quelques commandes.

Le processus de mise a jour Windows suit un cycle en cinq etapes :

1. L'orchestrateur de mise a jour interroge les serveurs Microsoft Update ou le serveur WSUS interne
2. L'orchestrateur identifie les mises a jour applicables a la configuration du poste
3. Le telechargement s'effectue en arriere-plan
4. L'agent d'installation applique les correctifs
5. Un redemarrage finalise les modifications (necessaire pour les services et drivers)

{% hint style="info" %}
En environnement d'entreprise, un serveur WSUS centralise la distribution des mises a jour. Cela evite que chaque poste telecharge individuellement les correctifs depuis Internet et permet de tester les mises a jour avant deploiement.
{% endhint %}

### Gestion des configurations (Group Policy)

Les Group Policy Objects (GPO) permettent de gerer centralement les configurations de securite pour les utilisateurs et les ordinateurs du domaine. Les parametres se configurent via la console GPMC (Group Policy Management Console) ou via PowerShell.

| Categorie de configuration | Exemples de parametres |
|---|---|
| **Politique de mot de passe** | Historique, longueur minimale, complexite, expiration |
| **Verrouillage de compte** | Seuil de tentatives, duree de verrouillage |
| **Audit** | Journalisation des evenements de connexion, creation de processus |
| **Restriction d'applications** | AppLocker, Software Restriction Policies |
| **UAC** | Niveau de consentement, secure desktop |
| **Services** | Comptes de service, type de demarrage |

### Gestion des utilisateurs

La surface d'attaque est proportionnelle au nombre de comptes et de privileges distribues.

| Mesure | Detail |
|---|---|
| **Limiter les comptes** | Minimiser le nombre de comptes locaux et administrateurs |
| **Surveiller les connexions** | Journaliser les tentatives de connexion valides et invalides |
| **Politique de mots de passe forte** | Privilegier les passphrases longues avec historique (eviter la reutilisation) |
| **Authentification a deux facteurs** | Reduire l'impact des credentials compromis |
| **Groupes minimaux** | Ne pas placer les utilisateurs dans des groupes excessifs (Backup Operators, Server Operators) |
| **Principe du moindre privilege** | Chaque compte ne doit avoir que les droits necessaires a ses taches |

### Audit et referentiels

Les audits periodiques completent les verifications automatisees. Plusieurs referentiels fournissent des bases de comparaison.

| Referentiel | Organisme | Usage |
|---|---|---|
| **DISA STIGs** | DoD | Guides techniques de securite par OS et application |
| **CIS Benchmarks** | CIS | Configurations recommandees par plateforme |
| **Microsoft Security Compliance Toolkit** | Microsoft | Baselines de securite officielles |
| **ISO 27001** | ISO | Cadre de gestion de la securite de l'information |
| **PCI DSS** | PCI SSC | Requis pour le traitement de donnees de paiement |

{% hint style="warning" %}
Un audit de configuration n'est pas un remplacement pour un pentest. Un score de conformite eleve ne garantit pas l'absence de chaines d'exploitation complexes. Les deux approches sont complementaires.
{% endhint %}

### Journalisation et detection

| Composant | Configuration recommandee |
|---|---|
| **Journaux de securite** | Activer l'audit des connexions, des creations de processus (Event ID 4688) et des modifications de privileges |
| **Audit des lignes de commande** | Activer "Include command line in process creation events" |
| **Sysmon** | Deployer Sysmon pour une visibilite granulaire sur les processus, le reseau et le registre |
| **Transfert de journaux** | Centraliser les journaux vers un SIEM (Splunk, ELK, Sentinel) |
| **Windows Defender ATP** | Detection avancee des menaces (Server 2019+) |

## En pratique

### Verifier la politique de mot de passe

```cmd
# - Politique locale
net accounts

# Resultats typiques :
# Minimum password length: 8
# Maximum password age (days): 90
# Lockout threshold: 5
```

### Verifier l'etat de l'UAC

```cmd
# - UAC active ?
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA

# - Niveau UAC
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

### Audit des services avec des comptes privilegies

```powershell
# - Lister les services qui tournent en SYSTEM
Get-WmiObject win32_service | Where-Object {$_.StartName -eq "LocalSystem"} | Select-Object Name, PathName, StartMode

# - Identifier les services avec des permissions faibles
accesschk.exe /accepteula -quvcw * | findstr /i "BUILTIN\Users"
```

### Audit des groupes privilegies

```powershell
# - Verifier les membres des groupes sensibles
net localgroup "Backup Operators"
net localgroup "Server Operators"
net localgroup "DnsAdmins"
net localgroup "Hyper-V Administrators"
net localgroup "Print Operators"

# - En Active Directory
Get-ADGroupMember -Identity "Domain Admins" -Recursive | Select-Object Name
Get-ADGroupMember -Identity "DnsAdmins" | Select-Object Name
```

### Checklist de durcissement rapide

```powershell
# - Verifier les binaires dans des chemins non quotes
wmic service get name,pathname | findstr /i /v "c:\windows\\" | findstr /i /v """

# - Verifier les permissions sur les executables de services
Get-WmiObject win32_service | Select-Object Name, PathName | ForEach-Object { icacls $_.PathName.Trim('"') 2>$null }

# - Verifier Always Install Elevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul

# - Verifier les credentials dans les fichiers de config
findstr /SIM /C:"password" C:\inetpub\*.config C:\*.xml 2>nul
```

### Recommandations post-audit typiques

| Constat | Recommandation |
|---|---|
| Services avec des ACL faibles | Restreindre les permissions (retirer Authenticated Users de SERVICE_ALL_ACCESS) |
| Chemins de service non quotes | Ajouter des guillemets dans la configuration du service |
| Always Install Elevated active | Desactiver la politique dans les deux ruches (HKCU et HKLM) |
| Comptes dans des groupes privilegies sans justification | Retirer les comptes et documenter les besoins reels |
| Pas d'audit des processus | Activer "Audit Process Creation" et "Include command line" |
| Systemes legacy non segmentes | Isoler les systemes en fin de vie sur des VLAN dedies avec des regles de pare-feu strictes |
| UAC desactive | Reactiver l'UAC au niveau par defaut minimum |
| Pas de LAPS | Deployer LAPS pour randomiser les mots de passe d'administrateur local |

{% hint style="success" %}
Le deploiement de LAPS (Local Administrator Password Solution) est l'une des mesures les plus impactantes pour limiter le lateral movement. Il garantit que chaque poste a un mot de passe administrateur local unique et regulierement renouvele.
{% endhint %}

## Pieges et galeres

- **Durcissement excessif** : des politiques trop restrictives cassent des applications. Tester chaque changement dans un environnement de pre-production
- **LAPS et recuperation** : sans procedure de recuperation documentee, un mot de passe LAPS perdu peut rendre un poste inaccessible. Documenter le processus de recuperation
- **Audit des lignes de commande** : l'activation de l'audit des lignes de commande genere un volume important de journaux. S'assurer que l'infrastructure de stockage est dimensionnee en consequence
- **AppLocker et exceptions** : AppLocker mal configure avec trop d'exceptions perd son utilite. Privilegier le mode whitelist strict
- **Systemes legacy et segmentation** : la segmentation reseau ne corrige pas la vulnerabilite, elle limite l'exposition. Planifier la migration ou le remplacement a terme

## Memo express

| Mesure | Outil / Methode |
|---|---|
| Mises a jour centralisees | WSUS / SCCM |
| Configuration centralisee | Group Policy (GPMC) |
| Politique de mot de passe | GPO > Account Policies > Password Policy |
| Audit des evenements | GPO > Audit Policy > Process Creation |
| Journalisation avancee | Sysmon + transfert vers SIEM |
| Baselines de securite | CIS Benchmarks / DISA STIGs |
| Mot de passe admin local unique | LAPS (Local Administrator Password Solution) |
| Restriction d'applications | AppLocker en mode whitelist |
| Verification des services | `accesschk.exe /accepteula -quvcw *` |
| Verification de l'UAC | `REG QUERY ...Policies\System /v EnableLUA` |

***
