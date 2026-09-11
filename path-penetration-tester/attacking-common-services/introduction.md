# Introduction

Les services réseau (FTP, SMB, bases de données, messagerie, RDP, DNS) sont omniprésents dans les environnements d'entreprise. Chacun d'eux expose une surface d'attaque spécifique, allant des défauts de configuration aux vulnérabilités protocolaires. Ce module passe en revue les techniques d'attaque les plus courantes sur ces services, de l'énumération initiale jusqu'à l'exécution de commandes à distance.

## Pourquoi

En test d'intrusion, les services réseau représentent souvent le premier point d'entrée dans un système. Un partage SMB ouvert, un FTP en accès anonyme, ou une base de données mal configurée peut suffire pour prendre pied sur le réseau cible. Comprendre comment identifier et exploiter ces faiblesses est une compétence fondamentale du path CPTS.

## Les services couverts

| Service | Ports par défaut | Intérêt offensif |
|---|---|---|
| FTP | TCP/21 | Accès anonyme, brute force, bounce attack |
| SMB | TCP/139, TCP/445 | Null sessions, RCE, hash capture, relay |
| MySQL | TCP/3306 | Lecture/écriture de fichiers, UDF |
| MSSQL | TCP/1433 | `xp_cmdshell`, impersonation, linked servers |
| RDP | TCP/3389 | Password spray, session hijacking, PtH |
| DNS | UDP/TCP 53 | Zone transfer, subdomain takeover, spoofing |
| SMTP | TCP/25, 587 | Énumération d'utilisateurs, open relay, phishing |
| POP3/IMAP | TCP/110, 143, 993, 995 | Accès aux boîtes mail, brute force |

## Le cadre Source, Process, Privileges, Destination

Chaque attaque sur un service réseau suit un schéma récurrent, utile pour structurer l'analyse :

1. **Source** : le point d'entrée de l'attaque (input utilisateur, requête réseau, données manipulées)
2. **Process** : le traitement effectué par le service sur ces données
3. **Privileges** : le niveau de droits avec lequel le service exécute ce traitement
4. **Destination** : là où aboutit le résultat (fichier local, processus, connexion réseau)

Ce cycle peut se répéter : le résultat d'une première itération (par exemple, l'obtention d'un hash) devient la source d'une seconde (cracking ou relay du hash pour obtenir un accès). En test d'intrusion, identifier ces enchaînements permet de construire des chaînes d'exploitation efficaces.

## Mauvaises configurations récurrentes

Avant de chercher des vulnérabilités complexes, vérifier les bases. Les mêmes types de défauts reviennent sur la majorité des services :

### Authentification

| Problème | Impact | Services concernés |
|---|---|---|
| Accès anonyme activé | Lecture (parfois écriture) sans identifiants | FTP, SMB, MSSQL |
| Identifiants par défaut | Accès complet au service | Tous |
| Mots de passe faibles | Vulnérable au brute force / spray | Tous |
| Absence de politique de verrouillage | Brute force sans limite | RDP, SMB, services mail |

### Droits excessifs

- Comptes de service avec privilèges administrateur
- Partages réseau accessibles en lecture/écriture à tous
- Bases de données avec l'utilisateur `sa` ou `root` sans restrictions
- Services mail configurés en open relay

{% hint style="info" %}
La règle du moindre privilège s'applique à tous les niveaux : comptes utilisateurs, comptes de service, droits d'accès aux partages, permissions SQL. En audit, tester systématiquement ce que permet un accès sans identifiants, puis avec des identifiants faibles ou par défaut.
{% endhint %}

## Interagir avec les services

Avant d'attaquer un service, il faut savoir s'y connecter. Voici les outils de base depuis un poste Linux (votre machine d'attaque) :

| Service | Outil(s) | Commande type |
|---|---|---|
| FTP | `ftp`, `lftp` | `ftp <IP_CIBLE>` |
| SMB | `smbclient`, `smbmap`, `rpcclient` | `smbclient -N -L //<IP_CIBLE>` |
| MySQL | `mysql` | `mysql -u user -p -h <IP_CIBLE>` |
| MSSQL | `mssqlclient.py`, `sqsh` | `mssqlclient.py user@<IP_CIBLE>` |
| RDP | `xfreerdp`, `rdesktop` | `xfreerdp /v:<IP_CIBLE> /u:user /p:pass` |
| DNS | `dig`, `nslookup`, `host` | `dig axfr @<IP_CIBLE> domaine.htb` |
| SMTP | `telnet`, `swaks`, `smtp-user-enum` | `telnet <IP_CIBLE> 25` |
| POP3 | `telnet`, `curl` | `curl pop3://<IP_CIBLE>/1 -u user:pass` |
| IMAP | `telnet`, `curl`, `evolution` | `curl imap://<IP_CIBLE> -u user:pass` |

{% hint style="success" %}
Sur la plupart des distributions offensives, la plupart de ces outils sont préinstallés. Pour les bases de données, `dbeaver` offre une interface graphique compatible MySQL, MSSQL, PostgreSQL et Oracle si on préfère explorer visuellement les données.
{% endhint %}

## Quoi chercher

Une fois connecté à un service, les informations sensibles se trouvent souvent dans des endroits prévisibles :

- **Partages de fichiers** (SMB, FTP) : configurations, scripts avec des identifiants en dur, sauvegardes, clés SSH, documents internes
- **Bases de données** : tables `users`, `credentials`, `config`, données personnelles (PII)
- **Messagerie** : mots de passe échangés par mail, informations de configuration, organigrammes
- **DNS** : sous-domaines internes, enregistrements TXT avec des tokens ou des clés

{% hint style="warning" %}
En audit réel, documenter chaque donnée sensible découverte mais ne pas exfiltrer plus que nécessaire pour prouver l'impact. Les données personnelles (PII) nécessitent un traitement particulier dans le rapport.
{% endhint %}

## Mémo express

| Étape | Action | Outils |
|---|---|---|
| 1. Identifier les services | Scan de ports et versions | `nmap -sCV` |
| 2. Tester l'accès anonyme | Connexion sans identifiants | `smbclient -N`, `ftp anonymous@` |
| 3. Énumérer | Lister partages, utilisateurs, bases | `smbmap`, `rpcclient`, `enum4linux` |
| 4. Brute force / spray | Tester les identifiants faibles | `crackmapexec`, `hydra`, `medusa` |
| 5. Exploiter les protocoles | Techniques spécifiques par service | Voir les pages dédiées |
| 6. Chercher les données | Parcourir les fichiers / tables / mails | Navigation manuelle + scripts |

***
