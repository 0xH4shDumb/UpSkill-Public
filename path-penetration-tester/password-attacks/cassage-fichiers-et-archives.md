# Cassage de fichiers et archives proteges

En pentest, il est frequent de tomber sur des fichiers proteges par mot de passe : archives ZIP ou RAR, documents Office, cles SSH chiffrees, disques BitLocker. John the Ripper et Hashcat permettent de casser ces protections en extrayant d'abord le hash du fichier, puis en le soumettant a une attaque par dictionnaire.

## Pourquoi

Un fichier Excel protege peut contenir une liste de credentials. Une archive ZIP chiffree peut cacher des donnees sensibles. Une cle SSH protegee par passphrase est inutilisable sans la casser. Ces fichiers sont souvent decouverts lors de l'exploration post-exploitation et peuvent etre le point de bascule d'un engagement.

## Comment ca marche

Le processus est toujours le meme : extraire le hash du fichier protege avec un outil `*2john`, puis casser ce hash avec John ou Hashcat.

```
Fichier protege --> *2john --> hash --> john/hashcat --> mot de passe
```

### Outils de conversion

| Outil | Type de fichier |
|---|---|
| `zip2john` | Archives ZIP |
| `rar2john` | Archives RAR |
| `ssh2john` | Cles SSH chiffrees |
| `office2john` | Documents Office (Word, Excel, PowerPoint) |
| `pdf2john` | Documents PDF |
| `bitlocker2john` | Volumes BitLocker |
| `wpa2john` | Handshakes WPA/WPA2 |
| `keepass2john` | Bases de donnees KeePass |

## En pratique

### Archives ZIP

```bash
# - Extraire le hash
zip2john fichier.zip > zip.hash

# - Casser avec John
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash

# - Afficher le resultat
john zip.hash --show
```

### Archives chiffrees avec OpenSSL

Certains formats (TAR, GZIP) ne gerent pas le chiffrement nativement. Ils sont parfois chiffres avec OpenSSL.

```bash
# - Identifier le type
file fichier.gzip
# openssl enc'd data with salted password

# - Brute force avec une boucle
for i in $(cat /usr/share/wordlists/rockyou.txt); do
  openssl enc -aes-256-cbc -d -in fichier.gzip -k $i 2>/dev/null | tar xz && echo "Mot de passe: $i" && break
done
```

### Documents Office et PDF

{% tabs %}
{% tab title="Office" %}
```bash
# - Extraire le hash d'un document Office
office2john Document.docx > office.hash

# - Casser
john --wordlist=/usr/share/wordlists/rockyou.txt office.hash
john office.hash --show
```
{% endtab %}
{% tab title="PDF" %}
```bash
# - Extraire le hash d'un PDF
pdf2john Document.pdf > pdf.hash

# - Casser
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
john pdf.hash --show
```
{% endtab %}
{% endtabs %}

### Cles SSH

```bash
# - Verifier si une cle est protegee par passphrase
ssh-keygen -yf id_rsa
# Enter passphrase: --> la cle est protegee

# - Extraire le hash
ssh2john id_rsa > ssh.hash

# - Casser
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
john ssh.hash --show
```

### Volumes BitLocker

BitLocker est le chiffrement de disque integre a Windows. Il protege les volumes avec un mot de passe ou un code de recuperation de 48 chiffres.

```bash
# - Extraire les hashs
bitlocker2john -i Volume.vhd > bitlocker.hashes
grep "bitlocker\$0" bitlocker.hashes > bitlocker.hash

# - Casser avec Hashcat (mode 22100)
hashcat -a 0 -m 22100 bitlocker.hash /usr/share/wordlists/rockyou.txt
```

{% tabs %}
{% tab title="Montage Windows" %}
Double-cliquer sur le fichier `.vhd`, Windows demande le mot de passe pour monter le disque.
{% endtab %}
{% tab title="Montage Linux" %}
```bash
# - Installer dislocker
sudo apt-get install dislocker

# - Creer les points de montage
sudo mkdir -p /media/bitlocker /media/bitlockermount

# - Monter le volume
sudo losetup -f -P Volume.vhd
sudo dislocker /dev/loop0p2 -u<MOTDEPASSE> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount

# - Acceder aux fichiers
ls /media/bitlockermount/
```
{% endtab %}
{% endtabs %}

### Recherche de fichiers chiffres sur un systeme compromis

```bash
# - Rechercher des fichiers potentiellement chiffres
for ext in .xls .xlsx .doc .docx .pdf .kdbx .pem .key; do
  echo "=== $ext ==="
  find / -name "*$ext" 2>/dev/null | grep -v "lib\|fonts\|share"
done

# - Rechercher des cles SSH
grep -rn 'BEGIN.*PRIVATE KEY' /home/ 2>/dev/null
```

## Pieges et galeres

- **Temps de cassage** : les documents Office 2013+ utilisent 100 000 iterations de SHA-512. C'est beaucoup plus lent que du NTLM. Prevoir du temps ou un GPU puissant
- **Faux positifs zip2john** : certains fichiers ZIP utilisent un chiffrement que John ne supporte pas (AES-256 avec WinZip). Verifier le type de chiffrement
- **Cles SSH modernes** : les cles ed25519 avec le nouveau format OpenSSH ne laissent aucune indication visible de chiffrement dans l'en-tete. Tester avec `ssh-keygen -yf`
- **BitLocker recovery key** : le code de recuperation de 48 chiffres est une alternative au mot de passe. Il est souvent stocke dans l'AD ou dans un fichier texte

## Memo express

| Commande | Usage |
|---|---|
| `zip2john file.zip > hash` | Hash d'une archive ZIP |
| `ssh2john id_rsa > hash` | Hash d'une cle SSH |
| `office2john doc.xlsx > hash` | Hash d'un document Office |
| `pdf2john doc.pdf > hash` | Hash d'un PDF |
| `bitlocker2john -i disk.vhd > hash` | Hash BitLocker |
| `john --wordlist=rockyou hash` | Casser avec John |
| `hashcat -a 0 -m <mode> hash wordlist` | Casser avec Hashcat |

***
