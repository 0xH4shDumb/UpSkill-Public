# Protocoles de gestion distante Windows

Les environnements Windows offrent plusieurs mécanismes de gestion à distance : RDP pour l'accès graphique, WinRM pour PowerShell distant, et WMI pour la gestion et l'exécution de commandes. Ces services sont activés par défaut sur Windows Server et constituent des cibles prioritaires en test d'intrusion.

## Pourquoi

En environnement Active Directory, la gestion à distance est la norme. Les administrateurs utilisent RDP, WinRM et WMI quotidiennement. Pour un attaquant, ces services sont des portes d'entrée directes : un couple de credentials valides suffit pour obtenir un shell interactif, exécuter des commandes, ou pivoter vers d'autres machines du domaine.

## Comment ça marche

### RDP (Remote Desktop Protocol)

RDP est le protocole Microsoft pour l'accès graphique à distance. Il fonctionne sur le port TCP 3389 (et optionnellement UDP 3389 pour l'accélération).

| Caractéristique | Détail |
|---|---|
| Port | TCP/UDP 3389 |
| Chiffrement | TLS/SSL |
| Authentification | NLA (Network Level Authentication) par défaut |
| Risque | Certificats auto-signés vulnérables au MITM |

{% hint style="info" %}
NLA (Network Level Authentication) force l'authentification avant d'établir la session graphique. Sans NLA, un attaquant peut se connecter à l'écran de login et tenter un brute-force sans contrainte de bande passante.
{% endhint %}

### WinRM (Windows Remote Management)

WinRM est le protocole de gestion à distance en ligne de commande, basé sur SOAP/HTTP(S). Il permet l'exécution de commandes PowerShell à distance.

| Caractéristique | Détail |
|---|---|
| Ports | TCP 5985 (HTTP), TCP 5986 (HTTPS) |
| Usage | PowerShell Remoting, Invoke-Command, WinRS |
| Outil offensif | Evil-WinRM |

### WMI (Windows Management Instrumentation)

WMI donne accès en lecture/écriture à la quasi-totalité des composants système Windows : processus, services, registre, réseau, matériel. Il initialise la communication sur TCP 135 puis utilise des ports dynamiques.

## En pratique

### RDP

{% tabs %}
{% tab title="Scan" %}
```bash
# depuis Exegol - scan et enumeration RDP
sudo nmap -sV -sC -p3389 --script rdp* <IP_CIBLE>
```

Les scripts `rdp-enum-encryption` et `rdp-ntlm-info` sont particulièrement utiles : le premier liste les couches de chiffrement supportées, le second extrait le hostname et le domaine via le challenge NTLM.
{% endtab %}

{% tab title="Audit de sécurité" %}
```bash
# depuis Exegol - vérification des paramètres de sécurité RDP
rdp-sec-check.pl <IP_CIBLE>
```
{% endtab %}

{% tab title="Connexion" %}
```bash
# depuis Exegol - connexion RDP depuis Linux
xfreerdp /u:user /p:password /v:<IP_CIBLE>
```

```bash
# depuis Exegol - avec partage de dossier local
xfreerdp /u:user /p:password /v:<IP_CIBLE> /drive:share,/tmp/loot
```
{% endtab %}
{% endtabs %}

### WinRM

```bash
# depuis Exegol - scan des ports WinRM
sudo nmap -sV -sC -p5985,5986 <IP_CIBLE>
```

```bash
# depuis Exegol - connexion via Evil-WinRM
evil-winrm -i <IP_CIBLE> -u user -p password
```

{% hint style="success" %}
Evil-WinRM est l'outil de référence en pentest pour WinRM. Il fournit un shell PowerShell interactif avec des fonctionnalités intégrées : upload/download de fichiers, chargement de scripts, bypass AMSI.
{% endhint %}

### WMI

```bash
# depuis Exegol - exécution de commandes via WMI
wmiexec.py user:password@<IP_CIBLE> "whoami"
```

```bash
# depuis Exegol - shell interactif WMI
wmiexec.py user:password@<IP_CIBLE>
```

{% hint style="warning" %}
WMI utilise des ports dynamiques après l'initialisation sur TCP 135. En environnement filtré, la communication peut échouer si les ports hauts ne sont pas ouverts. C'est un problème courant quand on passe par un tunnel ou un pivot.
{% endhint %}

## Retour terrain

RDP est le service le plus visible et le plus surveillé. WinRM et WMI, en revanche, passent souvent sous le radar des équipes de sécurité. Evil-WinRM est devenu l'outil standard pour le mouvement latéral en AD, parce qu'il fonctionne sur un port légitime (5985) avec du trafic chiffré. WMI via Impacket (wmiexec.py) est une alternative quand WinRM n'est pas activé.

En pratique, un scan des ports 3389, 5985, 5986 et 135 sur l'ensemble du périmètre interne révèle souvent des machines avec des comptes locaux à mots de passe faibles, ou des comptes de service réutilisés sur plusieurs serveurs.

## Mémo express

| Commande | Usage |
|---|---|
| `sudo nmap -sV -sC -p3389 --script rdp* <IP>` | Scan RDP |
| `xfreerdp /u:user /p:pass /v:<IP>` | Connexion RDP |
| `evil-winrm -i <IP> -u user -p pass` | Shell WinRM |
| `wmiexec.py user:pass@<IP> "whoami"` | Commande via WMI |
| `sudo nmap -sV -sC -p5985,5986 <IP>` | Scan WinRM |

***
