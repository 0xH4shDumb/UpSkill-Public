# Extraction de credentials sur le reseau

Au-dela des fichiers locaux, le reseau lui-meme regorge de credentials exploitables : partages SMB contenant des fichiers de configuration, trafic non chiffre transitant en clair, community strings SNMP. Savoir fouiller efficacement ces sources peut debloquer un engagement entier.

## Pourquoi

Dans un environnement d'entreprise, les partages reseau sont omnipresents. Les equipes IT, RH et finance y deposent regulierement des fichiers contenant des credentials en clair (scripts, configurations, documents d'onboarding). Cote trafic, certains services herites (HTTP, FTP, SNMP, LDAP sans TLS) transmettent encore des identifiants sans chiffrement. Intercepter ou analyser ces flux revient a ramasser des credentials sans meme toucher a un systeme.

## Comment ca marche

### Partages reseau

Les partages SMB/CIFS sont accessibles avec un simple compte de domaine. L'objectif est d'identifier les fichiers contenant des mots de passe, des tokens ou des configurations sensibles. On recherche :

| Cible | Exemples |
|---|---|
| **Mots-cles dans le contenu** | `passw`, `token`, `secret`, `key`, `cred` |
| **Extensions sensibles** | `.ini`, `.cfg`, `.env`, `.xlsx`, `.ps1`, `.bat`, `.xml` |
| **Noms de fichiers evocateurs** | `config`, `cred`, `initial`, `onboarding`, `deploy` |
| **Partages prioritaires** | IT, Admin, Finance (plus de chances de contenir des credentials que "Photos") |

{% hint style="info" %}
Adapter les mots-cles a la langue de l'organisation. Une entreprise francophone stockera plus probablement un fichier nomme `identifiants.xlsx` que `credentials.xlsx`. De meme, une entreprise allemande utilisera `Benutzer` plutot que `User`.
{% endhint %}

### Trafic reseau

Malgre la generalisation du TLS, certains protocoles transmettent encore des donnees en clair.

| Protocole non chiffre | Equivalent chiffre | Risque |
|---|---|---|
| HTTP | HTTPS | Formulaires de login, cookies |
| FTP | FTPS / SFTP | Identifiants en clair |
| SNMP v1/v2 | SNMPv3 | Community strings |
| LDAP | LDAPS | Bind credentials |
| SMTP / POP3 / IMAP | SMTPS / POP3S / IMAPS | Identifiants mail |
| SMB (< 3.0) | SMB 3.0 avec TLS | Hashs NTLMv1/v2 |

## En pratique

### Recherche dans les partages SMB

{% tabs %}
{% tab title="Snaffler (Windows)" %}
Snaffler est un outil C# qui s'execute sur une machine jointe au domaine. Il decouvre automatiquement les partages accessibles et recherche les fichiers interessants.

```powershell
# - Lancer Snaffler en mode standard
Snaffler.exe -s

# - Filtrer par utilisateurs AD et partages specifiques
Snaffler.exe -s -u -i IT,Finance
```

Les resultats sont classes par severite (Red, Yellow, Green). Un fichier `unattend.xml` contenant des credentials d'administrateur sera classe en Red.
{% endtab %}
{% tab title="MANSPIDER (Linux)" %}
MANSPIDER permet de scanner des partages SMB depuis Linux via Docker.

```bash
# - Rechercher des fichiers contenant "passw" dans leur contenu
docker run --rm -v ./manspider:/root/.manspider \
  blacklanternsecurity/manspider <IP_CIBLE> \
  -c 'passw' -u '<USER>' -p '<PASSWORD>'

# - Les fichiers correspondants sont telecharges dans ./manspider/loot
```
{% endtab %}
{% tab title="NetExec (Linux)" %}
NetExec peut spider les partages et rechercher des patterns dans le contenu des fichiers.

```bash
# - Spider un partage specifique et chercher "passw"
netexec smb <IP_CIBLE> -u '<USER>' -p '<PASSWORD>' \
  --spider IT --content --pattern "passw"
```
{% endtab %}
{% tab title="PowerHuntShares (Windows)" %}
PowerHuntShares genere un rapport HTML complet des partages accessibles et de leur contenu sensible.

```powershell
# - Scanner les partages du domaine
Import-Module .\PowerHuntShares.ps1
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public
```

Le rapport HTML classe les trouvailles par criticite : critical, high, medium, low.
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Ces outils generent un volume important de resultats. Beaucoup sont des faux positifs. Il faut systematiquement valider manuellement les fichiers identifies avant de conclure qu'ils contiennent des credentials exploitables.
{% endhint %}

### Analyse de trafic reseau

{% tabs %}
{% tab title="Wireshark" %}
Wireshark permet d'analyser des captures reseau (live ou fichiers PCAP) pour extraire des credentials transmises en clair.

**Filtres utiles :**

| Filtre | Usage |
|---|---|
| `http` | Trafic HTTP non chiffre |
| `http.request.method == "POST"` | Requetes POST (formulaires de login) |
| `http contains "passw"` | Paquets contenant le mot "passw" |
| `ftp` | Trafic FTP (identifiants en clair) |
| `dns` | Resolution DNS |
| `tcp.port == 161` | Trafic SNMP |
| `ip.addr == <IP>` | Filtrer par adresse IP |

```bash
# - Rechercher des credentials dans un fichier PCAP
# Menu Edit > Find Packet > String > "passw"
# Ou via le filtre : http contains "passw"
```
{% endtab %}
{% tab title="PCredz" %}
PCredz extrait automatiquement les credentials depuis un fichier de capture ou du trafic live.

```bash
# - Analyser un fichier PCAP
./Pcredz -f capture.pcapng -t -v

# Extrait automatiquement :
# - Identifiants FTP, POP, SMTP, IMAP
# - Community strings SNMP
# - Hashs NTLMv1/v2 (SMB, LDAP, MSSQL, HTTP)
# - Hashs Kerberos (AS-REQ Pre-Auth)
# - Credentials HTTP (Basic, NTLM, formulaires)
# - Numeros de cartes bancaires
```
{% endtab %}
{% endtabs %}

### Recherche manuelle rapide

```bash
# - Recherche basique dans les partages (PowerShell)
Get-ChildItem -Recurse -Include *.ini,*.cfg,*.env,*.xlsx \\<SERVEUR>\<PARTAGE> |
  Select-String -Pattern "passw|secret|token" |
  Select-Object Path, LineNumber, Line

# - Recherche dans les fichiers d'un partage (Linux)
smbclient //<IP_CIBLE>/IT -U '<USER>%<PASSWORD>' -c 'recurse; prompt; mget *.txt *.ini *.cfg'
grep -rni "passw\|secret\|token" ./
```

## Pieges et galeres

- **Volume de donnees** : scanner tous les partages d'un domaine avec des milliers de fichiers peut prendre des heures. Cibler les partages IT et Admin en priorite
- **Faux positifs** : un fichier contenant le mot "password" dans sa documentation n'est pas forcement un credential. Toujours verifier le contexte
- **Permissions** : certains partages ne sont accessibles qu'avec des comptes specifiques. Tester avec chaque compte compromis
- **Trafic chiffre** : sur un reseau moderne, la majorite du trafic est chiffre. L'analyse de trafic en clair est surtout pertinente sur des environnements herites ou mal configures
- **Detection** : le spidering massif de partages genere du bruit. Les EDR et les SIEM peuvent detecter un volume anormal d'acces aux fichiers

## Memo express

| Commande | Usage |
|---|---|
| `Snaffler.exe -s` | Scanner les partages (Windows) |
| `docker run ... manspider <IP> -c 'passw'` | Scanner les partages (Linux) |
| `netexec smb <IP> --spider <SHARE> --content --pattern "passw"` | Spider un partage avec NetExec |
| `Invoke-HuntSMBShares` | Rapport HTML des partages (PowerShell) |
| `./Pcredz -f capture.pcapng` | Extraire les credentials d'un PCAP |
| Wireshark : `http contains "passw"` | Chercher des credentials dans le trafic HTTP |

***
