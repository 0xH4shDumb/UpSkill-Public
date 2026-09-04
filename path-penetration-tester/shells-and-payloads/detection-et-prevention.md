# Detection et prevention

## Pourquoi

Comprendre comment les shells et payloads sont detectes permet de mieux les deployer en pentest, et de mieux les identifier en defense. Un pentester qui connait les mecanismes de detection evite les erreurs grossieres. Un defenseur qui connait les techniques offensives sait ou regarder.

## Comment ca marche

### Le framework MITRE ATT&CK

Trois tactiques sont directement liees aux shells et payloads :

| Tactique | Pertinence |
|---|---|
| Initial Access (T1190) | Compromission via application web, service expose |
| Execution (TA0002) | Execution de code sur la cible (PowerShell, Metasploit) |
| Command & Control (TA0011) | Communication persistante avec la cible (HTTP, DNS) |

### Indicateurs a surveiller

**Upload de fichiers suspects :** les applications web acceptant des uploads sont des vecteurs privilegies. L'analyse des logs applicatifs, combinee a un antivirus et un WAF, permet de detecter les tentatives.

**Commandes inhabituelles :** `whoami`, `net user`, `systeminfo` executes par des comptes non administrateurs sont des signaux d'alerte. Les outils EDR enregistrent les lignes de commande et peuvent declencher des alertes.

**Sessions reseau anormales :** connexions sur des ports non standards (4444, 7777), traffic regulier de type "heartbeat" vers une IP externe, connexions en rafale. L'analyse NetFlow et un SIEM permettent de les identifier.

## En pratique

### Visibilite reseau

Un reverse shell non chiffre (Netcat sur port 4444) est visible en clair dans une capture Wireshark. Les commandes executees et leurs resultats transitent sans chiffrement.

**Mesures de detection :**
- Deployer des IDS/IPS aux points strategiques du reseau
- Activer l'inspection approfondie de paquets (DPI) sur les pare-feux
- Maintenir une cartographie reseau a jour pour reperer les anomalies
- Analyser les top talkers et les patterns de trafic inhabituels

### Protection des endpoints

- Antivirus et EDR actifs et a jour
- Pare-feu local configure (bloquer les connexions sortantes non necessaires)
- Journalisation PowerShell (ScriptBlock Logging, Module Logging)
- Sysmon pour la surveillance des processus et connexions reseau
- Politique de patch rigoureuse

### Mesures de mitigation

| Mesure | Objectif |
|---|---|
| Application sandboxing | Isoler les applications exposees |
| Principe du moindre privilege | Limiter les droits des comptes de service |
| Segmentation reseau | Placer les serveurs exposes en DMZ |
| Pare-feu applicatif (WAF) | Filtrer les requetes malveillantes |
| Whitelist d'applications | Bloquer l'execution de binaires non autorises |

## Pieges et galeres

- La detection basee uniquement sur les signatures (blacklist) est facile a contourner. Un attaquant qui modifie son payload ou son User-Agent passe sous le radar.
- Les logs sont inutiles s'ils ne sont pas analyses. Collecter sans surveiller donne une fausse impression de securite.
- Un shell chiffre (Meterpreter HTTPS, reverse shell SSL) est invisible pour un IDS qui ne fait pas d'interception SSL.

## Retour terrain

La defense en profondeur est la seule approche viable. Aucun outil unique ne suffit. La combinaison visibilite reseau + surveillance endpoint + segmentation + patch management cree les conditions pour detecter et contenir une intrusion. En pentest, cette connaissance permet d'evaluer le niveau de maturite du client et d'adapter ses techniques en consequence.

## Memo express

| Couche | Outils/mesures |
|---|---|
| Reseau | IDS/IPS, DPI, analyse NetFlow, SIEM |
| Endpoint | EDR, Sysmon, journalisation PowerShell, antivirus |
| Applicatif | WAF, whitelist d'applications, sandbox |
| Organisationnel | Patch management, moindre privilege, segmentation |

***
