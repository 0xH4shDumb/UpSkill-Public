# Concepts fondamentaux

Avant de lancer le moindre scan ou d'ecrire la premiere ligne d'un exploit, il faut maitriser le vocabulaire et les concepts de base du test d'intrusion. Cette page pose les fondations necessaires pour aborder sereinement les modules suivants.

## Pourquoi

Un pentester qui confond un reverse shell avec un bind shell, ou qui ne sait pas sur quel port tourne un service, perd un temps considerable en situation reelle. Maitriser ces notions permet de comprendre les outils, de lire les rapports d'audit et de communiquer efficacement avec une equipe technique.

## Comment ca marche

### Types de shells

Un shell est une interface qui permet d'interagir avec un systeme d'exploitation via des commandes. En pentest, on distingue trois grandes categories.

| Type | Principe | Utilisation typique |
|---|---|---|
| **Reverse shell** | La cible se connecte a la machine de l'attaquant | Contournement de firewall (la connexion sortante est rarement filtree) |
| **Bind shell** | La cible ouvre un port en ecoute, l'attaquant s'y connecte | Quand la cible est directement accessible (pas de NAT ou de firewall entrant) |
| **Web shell** | Script depose sur un serveur web qui execute des commandes via HTTP | Acces initial sur une application web vulnerable |

{% hint style="info" %}
Le reverse shell est le plus utilise en pratique. La plupart des firewalls autorisent les connexions sortantes, ce qui rend ce vecteur plus fiable qu'un bind shell qui necessite un port ouvert en entree.
{% endhint %}

### Ports et protocoles

Chaque service reseau ecoute sur un port specifique. Les ports se repartissent en trois plages.

| Plage | Designation | Exemple |
|---|---|---|
| 0 - 1023 | Ports privilegies (well-known) | 22 (SSH), 80 (HTTP), 443 (HTTPS) |
| 1024 - 49151 | Ports enregistres | 3306 (MySQL), 8080 (HTTP alternatif) |
| 49152 - 65535 | Ports dynamiques / prives | Utilises pour les connexions sortantes |

Les protocoles de transport principaux sont TCP (fiable, oriente connexion) et UDP (rapide, sans garantie de livraison). En pentest, TCP represente la majorite des services cibles, mais des protocoles comme DNS (port 53), SNMP (port 161) ou TFTP (port 69) fonctionnent en UDP et ne doivent pas etre negliges.

#### Ports courants a connaitre

| Port | Service | Protocole |
|---|---|---|
| 21 | FTP | TCP |
| 22 | SSH | TCP |
| 23 | Telnet | TCP |
| 25 | SMTP | TCP |
| 53 | DNS | TCP/UDP |
| 80 | HTTP | TCP |
| 110 | POP3 | TCP |
| 139/445 | SMB | TCP |
| 143 | IMAP | TCP |
| 443 | HTTPS | TCP |
| 3306 | MySQL | TCP |
| 3389 | RDP | TCP |
| 5985 | WinRM | TCP |

### Serveurs web

Un serveur web est le logiciel qui repond aux requetes HTTP des clients. Les trois serveurs les plus repandus sont Apache, Nginx et IIS (Microsoft). Chaque serveur a ses specificites en termes de configuration, de modules et de surface d'attaque.

En pentest, identifier le serveur web et sa version est une des premieres etapes de la reconnaissance. Cette information oriente la recherche d'exploits connus et permet de comprendre la structure de l'application hebergee.

### OWASP Top 10

L'OWASP (Open Web Application Security Project) publie regulierement un classement des dix categories de vulnerabilites web les plus critiques. Ce referentiel sert de base a de nombreux audits de securite applicative.

| Rang | Categorie | Description courte |
|---|---|---|
| A01 | Controle d'acces defaillant | Acces a des ressources ou fonctions non autorisees |
| A02 | Defaillances cryptographiques | Donnees sensibles exposees ou mal protegees |
| A03 | Injection | Code malveillant injecte dans une requete (SQL, OS, LDAP) |
| A04 | Conception non securisee | Failles d'architecture non couvertes par les controles |
| A05 | Mauvaise configuration | Parametres par defaut, services inutiles, headers absents |
| A06 | Composants vulnerables | Bibliotheques ou frameworks avec des failles connues |
| A07 | Authentification defaillante | Mots de passe faibles, sessions mal gerees |
| A08 | Integrite des donnees et du logiciel | Mises a jour non verifiees, pipelines CI/CD compromis |
| A09 | Journalisation insuffisante | Absence de logs exploitables pour la detection |
| A10 | SSRF (Server-Side Request Forgery) | Le serveur effectue des requetes vers des ressources internes |

{% hint style="warning" %}
L'OWASP Top 10 n'est pas une liste exhaustive. Il s'agit d'un consensus sur les risques les plus repandus. Des vulnerabilites hors de ce classement (race conditions, XXE, deserialisation) restent tout aussi dangereuses selon le contexte.
{% endhint %}

### Apprentissage guide vs exploratoire

Deux approches complementaires structurent la montee en competences en securite offensive.

L'**apprentissage guide** propose un parcours structure avec des modules, des exercices et des corrections. Il construit methodiquement les connaissances et evite les lacunes. L'inconvenient est qu'il peut manquer de realisme par rapport aux situations d'un vrai pentest.

L'**apprentissage exploratoire** place le praticien face a des cibles inconnues (CTF, labs, machines vulnerables). Il developpe la reflexion, la methodologie personnelle et la capacite a travailler sans filet. L'inconvenient est le risque de passer a cote de concepts fondamentaux si les bases ne sont pas solides.

{% hint style="success" %}
Alterner les deux approches est la strategie la plus efficace. Les modules structurent les connaissances, les labs les mettent a l'epreuve. Un concept appris en cours prend tout son sens quand on l'exploite sur une machine reelle.
{% endhint %}

## Pieges et galeres

- **Confondre port et service** : un service peut tourner sur un port non standard. Toujours verifier avec un scan de version (`-sV`), ne pas supposer que le port 80 heberge forcement du HTTP
- **Negliger UDP** : les scans par defaut se limitent souvent a TCP. Des services critiques comme SNMP ou TFTP ne seront jamais decouverts sans un scan UDP explicite
- **Se focaliser sur l'OWASP Top 10** : ce classement est un point de depart, pas une checklist exhaustive. Les vulnerabilites les plus interessantes d'un pentest se trouvent parfois dans la logique metier
- **Ne faire que des labs sans theorie** : attaquer des machines sans comprendre les concepts sous-jacents conduit a du copier-coller de commandes. La comprehension du "pourquoi" est ce qui differencie un bon pentester

## Memo express

| Concept | A retenir |
|---|---|
| **Reverse shell** | La cible se connecte a l'attaquant (le plus courant) |
| **Bind shell** | L'attaquant se connecte a la cible (port ouvert requis) |
| **Web shell** | Script HTTP pour executer des commandes |
| **TCP** | Fiable, oriente connexion, majorite des services |
| **UDP** | Rapide, sans connexion, DNS/SNMP/TFTP |
| **OWASP Top 10** | Referentiel des vulnerabilites web les plus courantes |
| **Ports privilegies** | 0-1023, necesitent des droits root/admin |

***
