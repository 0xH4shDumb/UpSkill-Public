# Introduction au transfert de fichiers

## Pourquoi

En pentest, le transfert de fichiers entre la machine d'attaque et la cible est une operation recurrente. Deployer un outil d'enumeration, exfiltrer un fichier de configuration, deposer une charge utile : chaque etape d'un engagement repose sur la capacite a deplacer des donnees d'un point A a un point B. Le probleme, c'est que ca ne marche jamais du premier coup dans un environnement durci.

## Comment ca marche

Un transfert de fichier, dans sa forme la plus simple, consiste a envoyer ou recevoir des donnees via un protocole reseau (HTTP, SMB, FTP, SSH) ou par un canal hors-bande (copier-coller base64, clipboard RDP). En pratique, plusieurs couches de securite viennent compliquer les choses.

### Contraintes courantes

| Mecanisme | Impact |
|---|---|
| Controle applicatif (AppLocker, WDAC) | Bloque l'execution de certains binaires ou scripts |
| Antivirus / EDR | Detecte et bloque les outils offensifs connus |
| Pare-feu | Filtre les ports en sortie ou en entree |
| IDS/IPS | Inspecte le contenu du trafic reseau |
| Proxy / filtrage web | Bloque certains domaines ou types de fichiers |

### Scenario typique

Lors d'un engagement, on obtient une RCE sur un serveur web IIS via un upload non restreint. On deploie une web shell, puis on tente d'envoyer un outil d'escalade de privileges. PowerShell est bloque par la politique applicative. On tente `certutil` pour telecharger depuis un depot externe, mais le proxy bloque GitHub et les hebergeurs de fichiers. Le FTP en sortie est filtre par le pare-feu. Finalement, on monte un partage SMB avec Impacket, car le port 445 est ouvert en sortie. Le fichier transite.

Ce genre de situation est la norme, pas l'exception. Maitriser plusieurs methodes de transfert, et savoir basculer rapidement de l'une a l'autre, fait partie des competences de base d'un pentester.

## En pratique

Ce module couvre les principales methodes de transfert, organisees par contexte :

- **Windows** : PowerShell (WebClient, Invoke-WebRequest), SMB, FTP, encodage base64, upload via serveur web
- **Linux** : wget, curl, SCP, /dev/tcp, execution fileless via pipe
- **Via du code** : Python, PHP, Ruby, Perl, JavaScript (cscript), VBScript
- **Methodes alternatives** : Netcat/Ncat, PowerShell Remoting (WinRM), montage RDP
- **Transferts securises** : chiffrement AES (PowerShell), OpenSSL
- **Serveur HTTP d'upload** : mise en place d'un Nginx avec support PUT
- **Living off the Land** : utilisation de binaires natifs (LOLBAS, GTFOBins) pour eviter de deposer de nouveaux fichiers
- **Detection et evasion** : comprendre comment les transferts sont detectes et comment adapter ses techniques

Chaque section fournit des commandes pretes a l'emploi, utilisables depuis un environnement Exegol ou equivalent.

## Retour terrain

La premiere methode qui vient en tete est rarement celle qui fonctionne dans un environnement restreint. Avoir un repertoire mental de 5 a 6 techniques de transfert, avec leurs variantes OS, permet d'avancer sans rester bloque. En mission, le temps perdu a chercher une methode de transfert qui passe est du temps en moins pour l'exploitation.

## Memo express

| Besoin | Methode rapide |
|---|---|
| Transfert Windows, port 445 ouvert | `impacket-smbserver` + `copy` |
| Transfert Linux, HTTP disponible | `python3 -m http.server` + `wget`/`curl` |
| Pas de reseau direct | Encodage base64 + copier-coller |
| Execution sans fichier sur disque | Pipe vers `bash` ou `IEX` en PowerShell |
| Besoin de chiffrement | OpenSSL (`enc -aes256`) ou `Invoke-AESEncryption` |
| Tout est bloque | Explorer LOLBAS/GTFOBins pour des binaires natifs |

***
