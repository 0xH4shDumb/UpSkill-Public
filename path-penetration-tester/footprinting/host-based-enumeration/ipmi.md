# IPMI

IPMI (Intelligent Platform Management Interface) est un standard de gestion matérielle qui permet de surveiller et contrôler un serveur a distance, indépendamment de son système d'exploitation. On le retrouve dans pratiquement tous les serveurs d'entreprise, souvent activé par défaut avec des identifiants faibles. En pentest, c'est un vecteur sous-estimé qui peut donner un accès quasi physique a la machine.

## Pourquoi

La plupart des serveurs en datacenter embarquent un contrôleur de gestion (BMC) accessible via le réseau. Ce composant permet de redémarrer la machine, accéder a la console, modifier le BIOS ou réinstaller un OS, et ce même si le serveur est éteint ou planté. Les administrateurs l'utilisent pour la maintenance a distance, mais la surface d'attaque qu'il expose est rarement prise en compte dans le durcissement du réseau.

En test d'intrusion, un accès IPMI équivaut a un accès physique au serveur. C'est un saut en privilèges considérable, et il passe souvent sous le radar parce que les interfaces de gestion ne sont pas toujours inventoriées.

## Comment ça marche

### Architecture

Le coeur du système est le **BMC** (Baseboard Management Controller), un microcontrôleur intégré a la carte mère. Il dispose de sa propre alimentation et de sa propre interface réseau, ce qui lui permet de fonctionner indépendamment du reste du système.

Les principaux composants :

| Composant | Rôle |
|---|---|
| BMC | Microcontrôleur central, gère toutes les opérations IPMI |
| IPMB | Bus de communication interne sur la carte mère |
| ICMB | Communication entre châssis (multi-serveur) |
| Mémoire IPMI | Stocke les journaux système (SEL) et la configuration |
| Interface réseau | Accessible via UDP/623 sur le réseau local |

{% hint style="info" %}
Le BMC reste actif tant qu'il est alimenté, même si le serveur est éteint. Une alimentation de veille (standby power) suffit.
{% endhint %}

### Implémentations courantes

Chaque constructeur a sa propre implémentation du BMC :

| Constructeur | Produit | Interface web |
|---|---|---|
| Dell | iDRAC | HTTPS sur le port dédié |
| HP | iLO (Integrated Lights-Out) | HTTPS avec licence optionnelle |
| Supermicro | IPMI natif | Interface web basique |

Ces interfaces proposent en général un accès console (KVM over IP), la gestion de l'alimentation, le monitoring matériel et le déploiement d'images ISO a distance.

### Versions du protocole

| Version | Particularités |
|---|---|
| IPMI 1.5 | Authentification basique (mot de passe, MD5) |
| IPMI 2.0 | Ajout de RAKP (authentification par échange de clés), chiffrement AES, support de Serial over LAN |

## En pratique

### Découverte du service

IPMI écoute sur **UDP/623**. Un scan classique en TCP ne le trouvera pas.

```bash
# Depuis Exegol - scan UDP ciblé sur le port IPMI
sudo nmap -sU --script ipmi-version -p 623 <IP_CIBLE>
```

Sortie typique :

```
PORT    STATE SERVICE
623/udp open  asf-rmcp
| ipmi-version:
|   Version: IPMI-2.0
|   UserAuth: auth_user, non_null_user
|   PassAuth: password, md5, null
|_  Level: 2.0
```

Le champ `PassAuth` révèle les méthodes d'authentification supportées. La présence de `null` indique que des connexions sans mot de passe sont possibles.

Avec Metasploit :

```bash
# Depuis Exegol - identification de la version IPMI
msfconsole -q
use auxiliary/scanner/ipmi/ipmi_version
set RHOSTS <IP_CIBLE>
run
```

### Exploitation de la faille RAKP (IPMI 2.0)

C'est la faille la plus critique d'IPMI. Le protocole RAKP, utilisé dans IPMI 2.0, transmet un hash du mot de passe **avant** que l'authentification soit terminée. N'importe qui sur le réseau peut récupérer ce hash sans avoir besoin de s'authentifier.

```bash
# Depuis Exegol - dump des hash IPMI via Metasploit
msfconsole -q
use auxiliary/scanner/ipmi/ipmi_dumphashes
set RHOSTS <IP_CIBLE>
run
```

Le module retourne les noms d'utilisateur et leurs hash au format IPMI2. On peut ensuite les craquer :

```bash
# Depuis Exegol - craquage du hash IPMI avec Hashcat
hashcat -m 7300 ipmi_hash.txt /usr/share/wordlists/rockyou.txt
```

{% hint style="danger" %}
Cette faille est inhérente au protocole IPMI 2.0 lui-même. Elle ne peut pas être corrigée par un patch ou une mise a jour firmware. La seule mitigation est de restreindre l'accès réseau au BMC.
{% endhint %}

### Mots de passe par défaut

Avant même de tenter le dump RAKP, tester les identifiants par défaut :

| Produit | Utilisateur | Mot de passe |
|---|---|---|
| Dell iDRAC | `root` | `calvin` |
| HP iLO | `Administrator` | Chaîne aléatoire (sur l'étiquette physique) |
| Supermicro IPMI | `ADMIN` | `ADMIN` |

{% hint style="warning" %}
Sur HP iLO, le mot de passe par défaut est unique par serveur et imprimé sur une étiquette physique. En revanche, les administrateurs le changent souvent pour un mot de passe partagé plus simple a retenir, ce qui rend le brute-force plus réaliste.
{% endhint %}

## Pièges et galères

{% hint style="danger" %}
**UDP = scan lent.** Le scan UDP est intrinsèquement plus lent que le TCP. Sur un sous-réseau complet, un scan du port 623 peut prendre plusieurs minutes. Ne pas oublier le flag `-sU` sous peine de ne rien trouver.
{% endhint %}

{% hint style="warning" %}
**Hash non craquable.** Si le mot de passe est suffisamment complexe, le hash récupéré via RAKP ne donnera rien avec une wordlist standard. Mais le simple fait de récupérer les noms d'utilisateur est déja une information exploitable.
{% endhint %}

{% hint style="warning" %}
**Interface web séparée.** Le BMC expose souvent une interface web sur un port HTTP/HTTPS distinct (80, 443, ou un port custom). Ne pas se limiter au port 623 : scanner aussi les ports web sur l'IP de gestion.
{% endhint %}

{% hint style="warning" %}
**Réutilisation de mots de passe.** Le mot de passe IPMI est souvent le même que celui du compte administrateur local (SSH, RDP). Un hash craqué sur l'interface IPMI peut ouvrir des portes bien au-dela du BMC.
{% endhint %}

## Retour terrain

{% hint style="success" %}
**Segmentation réseau.** L'interface IPMI devrait se trouver sur un VLAN d'administration isolé. En pratique, on la trouve régulièrement sur le même réseau que le trafic de production. C'est la première chose a vérifier.
{% endhint %}

{% hint style="success" %}
**Toujours tenter le dump RAKP.** Même si les mots de passe par défaut ont été changés, la faille RAKP reste exploitable. C'est un test rapide qui ne génère quasiment aucun bruit sur le réseau.
{% endhint %}

- Un accès IPMI permet de monter une ISO a distance et de booter dessus, ce qui revient a un accès physique complet. Sur un serveur Windows, ça peut servir a réinitialiser le mot de passe administrateur local.
- Les journaux SEL (System Event Log) du BMC contiennent parfois des informations intéressantes : redémarrages suspects, erreurs d'authentification, alertes matérielles qui trahissent l'activité sur le serveur.
- Sur les grands parcs, automatiser le scan UDP 623 sur l'ensemble des plages de gestion. Il n'est pas rare de trouver des dizaines de BMC avec les credentials par défaut.

## Mémo express

| Outil / Commande | Usage |
|---|---|
| `nmap -sU --script ipmi-version -p 623` | Découvrir et identifier le service IPMI |
| `msf > ipmi_version` | Alternative Metasploit pour la détection |
| `msf > ipmi_dumphashes` | Extraire les hash IPMI (faille RAKP) |
| `hashcat -m 7300` | Craquer les hash IPMI2 |
| `ipmitool -I lanplus -H <IP> -U <user> -P <pass> chassis status` | Interaction directe avec le BMC |
| `ipmitool ... sel list` | Lire les journaux système du BMC |

***
