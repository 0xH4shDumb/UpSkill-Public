# Introduction et decouverte d'applications

Les applications web sont presentes dans tous les environnements qu'on audite en pentest. CMS, portails internes, depots de code, outils de monitoring, systemes de ticketing, serveurs d'applications Java... la liste est longue. Certaines de ces applications sont directement exploitables via des vulnerabilites connues. D'autres offrent des fonctionnalites internes (console de scripts, upload de modules, editeur de templates) qui permettent d'obtenir une execution de code a distance si on dispose d'un acces administrateur.

Ce module couvre les applications les plus frequemment rencontrees en pentest interne et externe, avec pour chacune une approche d'enumeration, d'exploitation de failles connues et d'abus de fonctionnalites natives.

## Pourquoi

Les applications web representent souvent la surface d'attaque la plus large lors d'un test d'intrusion. Avec la generalisation du teletravail, de plus en plus d'applications sont exposees sur Internet, parfois involontairement. Une seule application mal configuree (identifiants par defaut, version obsolete, fonctionnalite d'administration exposee) peut servir de point d'entree vers le reseau interne.

Les statistiques le confirment : 72% des organisations interrogees dans une etude Barracuda ont subi au moins une breche liee a une vulnerabilite applicative. Les attaquants le savent, et les pentesters doivent le savoir aussi.

## Comment ca marche

### Panorama des applications courantes

| Categorie | Applications typiques |
|---|---|
| CMS (gestion de contenu) | WordPress, Joomla, Drupal |
| Serveurs d'applications | Apache Tomcat, JBoss, WebLogic |
| SIEM / Monitoring | Splunk, PRTG Network Monitor, Nagios, Zabbix |
| CI/CD et developpement | Jenkins, GitLab, Confluence |
| Ticketing / Support | osTicket, Zendesk, OTRS |
| Gestion de configuration | Puppet, Ansible, ManageEngine |

### Methodologie de decouverte

La decouverte d'applications suit un processus iteratif :

**1. Scan de ports web**

```bash
# - Scan des ports web courants sur le perimetre
nmap -p 80,443,8000,8080,8180,8443,8888,10000 --open -oA web_discovery -iL scope.txt
```

On commence par les ports les plus frequents, puis on elargit progressivement (top 10000, voire tous les ports TCP) selon la taille du perimetre.

**2. Capture d'ecrans automatisee**

Parcourir manuellement des centaines d'hotes est inefficace. Des outils comme EyeWitness et Aquatone prennent des captures d'ecran de chaque service web decouvert et generent un rapport HTML consultable dans le navigateur.

{% tabs %}
{% tab title="EyeWitness" %}
```bash
# - Capture d'ecrans a partir d'un scan Nmap
eyewitness --web -x web_discovery.xml -d rapport_eyewitness
```

EyeWitness accepte les sorties XML de Nmap et Nessus. Il categorise les applications detectees (CMS, pages de login, cibles a haute valeur) et suggere les identifiants par defaut quand il reconnait une application.
{% endtab %}
{% tab title="Aquatone" %}
```bash
# - Capture d'ecrans avec Aquatone
cat web_discovery.xml | aquatone -nmap
```

Aquatone accepte les sorties XML de Nmap et Masscan. Il genere un rapport HTML avec clustering des pages similaires, ce qui est utile pour identifier rapidement les doublons et se concentrer sur les cibles uniques.
{% endtab %}
{% endtabs %}

**3. Analyse du rapport**

Le rapport est organise par categories. Les "High Value Targets" (Tomcat Manager, Jenkins, Splunk) sont les plus interessantes. On note chaque application identifiee avec son URL, sa version quand elle est visible, et les identifiants par defaut connus.

{% hint style="success" %}
Ne pas se precipiter sur la premiere application exploitable. L'enumeration complete revele souvent des cibles plus juteuses enfouies dans le rapport. Sur un pentest externe, une application ManageEngine oubliee avec les identifiants par defaut `admin:admin` peut donner un acces Domain Admin en quelques minutes.
{% endhint %}

**4. Enumeration approfondie**

Pour chaque application identifiee, on enchaine avec un scan de service detaille :

```bash
# - Identification precise des services sur un hote
nmap --open -sV -p- <IP_CIBLE>
```

### Approche generale d'attaque

Pour chaque application decouverte, la methodologie est la meme :

1. **Identifier** l'application et sa version (pages d'erreur, fichiers README/CHANGELOG, headers HTTP, meta tags)
2. **Enumerer** les fonctionnalites exposees (pages d'administration, API, repertoires accessibles)
3. **Tester les identifiants par defaut** (la cause la plus frequente de compromission)
4. **Chercher les vulnerabilites connues** pour la version identifiee (searchsploit, CVE databases)
5. **Abuser les fonctionnalites natives** si on dispose d'un acces administrateur (editeur de templates, upload de modules, console de scripts)

{% hint style="info" %}
En pentest, les vulnerabilites les plus impactantes ne sont pas toujours les plus complexes. Un Tomcat avec `tomcat:tomcat` comme identifiants donne un RCE en quelques secondes via l'upload d'un WAR malveillant. Pas besoin d'exploit 0-day.
{% endhint %}

## En pratique

```bash
# 1 - Scanner les ports web du perimetre
nmap -p 80,443,8000,8080,8180,8443,8888,10000 --open -oA web_discovery -iL scope.txt

# 2 - Generer le rapport visuel
eyewitness --web -x web_discovery.xml -d rapport

# 3 - Parcourir le rapport et noter les applications
# Categoriser : CMS, serveurs d'applications, monitoring, CI/CD, ticketing

# 4 - Scanner en detail les hotes interessants
nmap --open -sV -p- <IP_CIBLE>

# 5 - Pour chaque application : identifier, enumerer, tester, exploiter
```

## Pieges et galeres

{% tabs %}
{% tab title="Decouverte" %}
- **Vhosts** : certaines applications ne sont accessibles que via un nom de domaine specifique. Si le scan Nmap ne revele qu'une page par defaut, penser a tester les sous-domaines connus via le fichier `/etc/hosts`
- **Ports non standards** : des applications comme Splunk (8000/8089), PRTG (8080), Jenkins (8000) ou GitLab (8081) tournent rarement sur les ports 80/443. Elargir le scan
- **Reverse proxies** : un serveur Nginx en facade peut masquer un Tomcat ou un Jenkins derriere lui. Les pages d'erreur et les headers `Server` aident a identifier la couche applicative reelle
{% endtab %}
{% tab title="Enumeration" %}
- **Versions masquees** : les administrateurs desactivent parfois l'affichage de la version. Chercher dans les fichiers `README.txt`, `CHANGELOG.txt`, les manifestes XML, les sources HTML et les fichiers JavaScript
- **Scanners insuffisants** : WPScan, droopescan et autres scanners automatises ne trouvent pas tout. L'enumeration manuelle (plugins, themes, utilisateurs) reste indispensable
- **Faux positifs** : un scanner peut confondre un CMS avec un autre. Toujours verifier manuellement
{% endtab %}
{% endtabs %}

## Retour terrain

La decouverte d'applications est la phase qui determine le succes d'un pentest. Sur un perimetre bien durci (pas de services RPC exposes, pas de SMB null session), les applications web sont souvent la seule porte d'entree.

EyeWitness genere des rapports qui peuvent faire des centaines de pages sur un grand perimetre. Il faut prendre le temps de tout parcourir. Les trouvailles les plus interessantes sont souvent enfouies au milieu du rapport : un formulaire d'upload sans validation, une application de monitoring avec les identifiants par defaut, un depot Git avec des credentials en clair dans un ancien commit.

La combinaison enumeration manuelle + outils automatises est la cle. Les scanners trouvent les vulnerabilites evidentes, l'humain trouve les mauvaises configurations et les enchainements logiques.

## Memo express

| Etape | Outil / Commande | Objectif |
|---|---|---|
| Scan web | `nmap -p 80,443,8000,8080...` | Identifier les services web |
| Screenshots | `eyewitness --web -x scan.xml` | Vue d'ensemble visuelle |
| Screenshots | `aquatone -nmap` | Alternative a EyeWitness |
| Scan detaille | `nmap --open -sV -p-` | Versions de services |
| Fingerprint CMS | `wpscan`, `droopescan`, curl + grep | Identifier le CMS et sa version |
| Identifiants par defaut | Tester manuellement | Premier reflexe sur chaque appli |
| Exploits connus | `searchsploit <application>` | Trouver les PoC publics |

***
