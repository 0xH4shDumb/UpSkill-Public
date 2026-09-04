# Transferts de fichiers securises

## Pourquoi

En pentest, on manipule regulierement des donnees sensibles : listes d'utilisateurs, hashes de mots de passe, fichiers de configuration contenant des secrets, dumps NTDS.dit. Transferer ces donnees en clair sur le reseau, c'est prendre le risque qu'elles soient interceptees par un IDS, un administrateur ou un attaquant tiers. Le chiffrement avant transfert est une precaution elementaire, et dans certains cas une obligation contractuelle.

{% hint style="danger" %}
Ne jamais exfiltrer de donnees personnelles (PII), financieres ou de secrets industriels sans accord explicite du client. Pour tester les mecanismes DLP, utiliser des fichiers factices qui imitent le format des donnees sensibles.
{% endhint %}

## Comment ca marche

Le principe est simple : chiffrer le fichier avant de le transferer, puis le dechiffrer a l'arrivee. Meme si le transfert est intercepte, le contenu reste illisible sans la cle. Deux approches couvrent les cas les plus courants : un script PowerShell pour Windows, et OpenSSL pour Linux.

## En pratique

### Chiffrement sous Windows (PowerShell AES)

Le script `Invoke-AESEncryption.ps1` permet de chiffrer et dechiffrer des fichiers ou des chaines de texte en AES.

**Importer le module :**

```powershell
Import-Module .\Invoke-AESEncryption.ps1
```

**Chiffrer un fichier :**

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "MotDePassefort!" -Path fichier.txt
```

Le fichier chiffre est cree sous le nom `fichier.txt.aes`.

**Dechiffrer :**

```powershell
Invoke-AESEncryption -Mode Decrypt -Key "MotDePassefort!" -Path fichier.txt.aes
```

**Chiffrer une chaine de texte :**

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "MotDePassefort!" -Text "contenu sensible"
```

{% hint style="warning" %}
Utiliser un mot de passe fort et unique par client. La reutilisation d'un mot de passe compromis mettrait en danger les donnees de tous les engagements.
{% endhint %}

### Chiffrement sous Linux (OpenSSL)

OpenSSL est present par defaut sur toutes les distributions Linux. Il permet un chiffrement AES-256 solide.

**Chiffrer un fichier :**

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
```

**Dechiffrer :**

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

| Option | Role |
|---|---|
| `-aes256` | Algorithme AES 256 bits |
| `-pbkdf2` | Derivation de cle securisee (evite les attaques par dictionnaire rapide) |
| `-iter 100000` | Nombre d'iterations pour le renforcement du mot de passe |

## Pieges et galeres

- Ne pas oublier de supprimer les fichiers en clair et les fichiers chiffres temporaires une fois le transfert termine. Les laisser trainer sur la cible ou sur la machine d'attaque est un risque.
- Le mot de passe de chiffrement ne doit pas transiter par le meme canal que le fichier chiffre. Utiliser un canal separe (message, autre connexion) pour partager la cle.
- OpenSSL sans `-pbkdf2` utilise une derivation de cle faible. Toujours ajouter cette option.
- Sur certains systemes anciens, la version d'OpenSSL ne supporte pas `-pbkdf2`. Verifier avec `openssl version`.

## Retour terrain

En pratique, le chiffrement OpenSSL couvre la majorite des besoins. C'est rapide, disponible partout, et suffisamment solide pour proteger des donnees le temps d'un engagement. Sur Windows, le script AES PowerShell est pratique quand on a deja une session PowerShell active, mais il necessite de deposer le script sur la cible, ce qui n'est pas toujours souhaitable.

L'alternative la plus propre reste d'utiliser directement un canal chiffre (SCP, SFTP, HTTPS) quand c'est possible. Le chiffrement explicite du fichier n'est necessaire que quand le canal de transfert lui-meme n'est pas securise.

## Memo express

| OS | Commande |
|---|---|
| Linux (chiffrer) | `openssl enc -aes256 -iter 100000 -pbkdf2 -in fichier -out fichier.enc` |
| Linux (dechiffrer) | `openssl enc -d -aes256 -iter 100000 -pbkdf2 -in fichier.enc -out fichier` |
| Windows (chiffrer) | `Invoke-AESEncryption -Mode Encrypt -Key "cle" -Path fichier` |
| Windows (dechiffrer) | `Invoke-AESEncryption -Mode Decrypt -Key "cle" -Path fichier.aes` |

***
