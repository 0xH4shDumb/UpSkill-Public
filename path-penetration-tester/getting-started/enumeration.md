# Enumeration et reconnaissance

L'enumeration est la phase qui consomme le plus de temps dans un pentest, et c'est normal. Chaque service decouvert, chaque version identifiee, chaque repertoire web trouve elargit la surface d'attaque et oriente les choix d'exploitation. Cette page couvre les outils fondamentaux de cette phase.

## Pourquoi

Sans enumeration rigoureuse, on passe a cote de services exposes, de versions vulnerables ou de repertoires caches qui constituent les vecteurs d'entree. Les pentesters experimentes repetent souvent que l'enumeration represente 80% du travail. Ce n'est pas une exageration.

## Comment ca marche

### Nmap

Nmap est le scanner de ports et de services de reference. Il permet de decouvrir les hotes actifs, les ports ouverts, les versions des services et d'executer des scripts d'enumeration automatisee (NSE).

#### Types de scans

| Scan | Option | Description |
|---|---|---|
| **SYN scan** | `-sS` | Scan par defaut (rapide, furtif, ne complete pas la connexion TCP) |
| **Connect scan** | `-sT` | Connexion TCP complete (utile sans privileges root) |
| **UDP scan** | `-sU` | Decouvre les services UDP (lent mais necessaire) |
| **Version detection** | `-sV` | Interroge les services pour identifier le logiciel et sa version |
| **Script scan** | `-sC` | Execute les scripts NSE par defaut |
| **OS detection** | `-O` | Tente d'identifier le systeme d'exploitation |

#### Scans courants

```bash
# - Scan rapide des ports les plus courants
nmap -sV --open <IP_CIBLE>

# - Scan complet de tous les ports TCP
nmap -sV -sC -p- <IP_CIBLE>

# - Scan UDP des 20 ports les plus courants
nmap -sU --top-ports 20 <IP_CIBLE>

# - Scan avec sauvegarde dans tous les formats
nmap -sV -sC -p- -oA scan_complet <IP_CIBLE>
```

{% hint style="info" %}
L'option `-oA` sauvegarde les resultats en trois formats (normal, grepable, XML). Le format grepable (`.gnmap`) est pratique pour extraire rapidement des informations avec `grep` et `awk`. Le XML peut etre importe dans des outils comme Metasploit.
{% endhint %}

#### Scripts NSE

Les scripts NSE (Nmap Scripting Engine) etendent les capacites de Nmap. Ils sont classes en categories (auth, brute, discovery, exploit, vuln, etc.).

```bash
# - Lister les scripts disponibles pour un service
ls /usr/share/nmap/scripts/ | grep smb

# - Enumeration HTTP
nmap --script=http-enum -p 80,443 <IP_CIBLE>

# - Scan de vulnerabilites
nmap --script=vuln -p 80 <IP_CIBLE>

# - Script specifique
nmap --script=smb-os-discovery -p 445 <IP_CIBLE>
```

### Banner grabbing

Le banner grabbing consiste a recuperer les bannieres des services pour identifier le logiciel, sa version et parfois le systeme d'exploitation. Plusieurs outils permettent de le faire.

{% tabs %}
{% tab title="Netcat" %}
```bash
nc -nv <IP_CIBLE> 22
# SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.1
```
{% endtab %}
{% tab title="cURL" %}
```bash
# - Headers HTTP
curl -I http://<IP_CIBLE>

# - Headers detailles
curl -v http://<IP_CIBLE>
```
{% endtab %}
{% tab title="Nmap" %}
```bash
nmap -sV --script=banner -p 21,22,80 <IP_CIBLE>
```
{% endtab %}
{% endtabs %}

### Enumeration web

Une fois qu'un serveur web est identifie, l'enumeration web cherche a decouvrir les repertoires caches, les fichiers de configuration accessibles, les sous-domaines et les technologies utilisees.

#### GoBuster

GoBuster est un outil de brute-force de repertoires et de sous-domaines. Il est rapide et supporte plusieurs modes de fonctionnement.

{% tabs %}
{% tab title="Repertoires" %}
```bash
gobuster dir -u http://<IP_CIBLE> \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,html,txt \
  -t 50
```
{% endtab %}
{% tab title="Sous-domaines (DNS)" %}
```bash
gobuster dns -d domaine.tld \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```
{% endtab %}
{% tab title="vHosts" %}
```bash
gobuster vhost -u http://<IP_CIBLE> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain
```
{% endtab %}
{% endtabs %}

#### ffuf

ffuf est une alternative a GoBuster, souvent preferee pour sa flexibilite. Il utilise le mot-cle `FUZZ` pour indiquer ou injecter les valeurs du dictionnaire.

```bash
# - Enumeration de repertoires
ffuf -u http://<IP_CIBLE>/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302

# - Enumeration de sous-domaines
ffuf -u http://FUZZ.domaine.tld \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -H "Host: FUZZ.domaine.tld"

# - Filtrer les reponses par taille
ffuf -u http://<IP_CIBLE>/FUZZ \
  -w wordlist.txt \
  -fs 1234
```

#### Identification des technologies

```bash
# - whatweb : identification rapide des technologies web
whatweb http://<IP_CIBLE>

# - curl : recuperation des headers de reponse
curl -I http://<IP_CIBLE>
```

### Codes de reponse HTTP

Les codes de reponse HTTP indiquent le resultat d'une requete. Les connaitre permet d'interpreter correctement les resultats d'enumeration.

| Code | Signification | Interpretation en pentest |
|---|---|---|
| **200** | OK | La ressource existe et est accessible |
| **301/302** | Redirection | La ressource a ete deplacee, suivre la redirection |
| **403** | Interdit | La ressource existe mais l'acces est refuse |
| **404** | Non trouve | La ressource n'existe pas |
| **405** | Methode non autorisee | La methode HTTP utilisee n'est pas acceptee |
| **500** | Erreur serveur | Erreur interne, peut indiquer une injection reussie |

{% hint style="warning" %}
Un code 403 ne signifie pas que la ressource est inaccessible. Il indique simplement que le serveur refuse l'acces avec les parametres actuels. Un changement de methode HTTP, un header specifique ou un contournement d'authentification peut parfois reveler le contenu.
{% endhint %}

## En pratique

### Workflow d'enumeration type

```bash
# - 1. Scan rapide pour identifier les ports ouverts
nmap -sV --open -oA quick_scan <IP_CIBLE>

# - 2. Scan complet en arriere-plan
nmap -sV -sC -p- -oA full_scan <IP_CIBLE> &

# - 3. Enumeration web sur les ports HTTP decouverts
gobuster dir -u http://<IP_CIBLE> \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,html,txt -t 50

# - 4. Banner grabbing sur les services non HTTP
nc -nv <IP_CIBLE> 21
nc -nv <IP_CIBLE> 22

# - 5. Identification des technologies web
whatweb http://<IP_CIBLE>
curl -I http://<IP_CIBLE>
```

### Enumeration FTP

Quand le port 21 est ouvert, verifier si l'acces anonyme est autorise.

```bash
# - Connexion FTP anonyme
ftp <IP_CIBLE>
# Login: anonymous
# Password: (vide)

# - Lister les fichiers
ls -la

# - Telecharger un fichier
get fichier.txt
```

## Pieges et galeres

- **Oublier les ports non standard** : un scan limite aux 1000 ports par defaut peut manquer un service web sur le port 8080, 8443 ou un service SSH sur le port 2222. Toujours faire un scan `-p-` (tous les ports)
- **Wordlist inadaptee** : la qualite de l'enumeration web depend du dictionnaire. `common.txt` est un bon debut, mais `directory-list-2.3-medium.txt` couvre davantage de cas. Adapter la wordlist au contexte (CMS, langage, framework)
- **Ne pas sauvegarder les scans** : relancer un scan de 20 minutes parce qu'on n'a pas sauvegarde les resultats est un gaspillage. Toujours utiliser `-oA` avec Nmap
- **Enumeration trop agressive** : un grand nombre de threads (`-t 100`) peut faire tomber un service fragile ou declencher un WAF. Commencer avec des valeurs raisonnables (20-50 threads)
- **Ignorer les redirections** : un code 301/302 indique souvent un repertoire valide. GoBuster et ffuf suivent les redirections par defaut, mais verifier les options si les resultats semblent incomplets

## Memo express

| Outil | Usage | Commande rapide |
|---|---|---|
| **Nmap** | Scan de ports et services | `nmap -sV -sC -p- -oA scan <IP_CIBLE>` |
| **NSE scripts** | Enumeration ciblee | `nmap --script=http-enum <IP_CIBLE>` |
| **GoBuster** | Brute-force repertoires/DNS | `gobuster dir -u <URL> -w wordlist.txt` |
| **ffuf** | Fuzzing web flexible | `ffuf -u <URL>/FUZZ -w wordlist.txt` |
| **whatweb** | Identification technologique | `whatweb http://<IP_CIBLE>` |
| **Netcat** | Banner grabbing | `nc -nv <IP_CIBLE> <PORT>` |

***
