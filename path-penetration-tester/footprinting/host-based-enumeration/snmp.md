# SNMP (Simple Network Management Protocol)

## Pourquoi

SNMP est un protocole omniprésent dans les infrastructures réseau. Il sert a superviser et administrer a distance les equipements : routeurs, switches, serveurs, imprimantes, objets connectes. Le probleme, c'est que sa securite repose en grande partie sur des chaines de caracteres appelees "community strings", souvent laissees par defaut. En pentest, un service SNMP mal configure peut livrer une quantite massive d'informations sur l'infrastructure cible : noms d'hotes, interfaces reseau, processus actifs, paquets installes, contacts administrateur.

C'est un vecteur d'enumeration souvent sous-estime, mais redoutablement efficace quand il est accessible.

## Comment ca marche

SNMP fonctionne sur un modele client-serveur. L'agent SNMP tourne sur l'equipement supervise et expose des donnees. Le manager SNMP (cote administrateur) interroge l'agent pour lire ou modifier ces donnees.

Les echanges passent par UDP :

| Port | Role |
|---|---|
| UDP 161 | Requetes classiques (get, set, walk) |
| UDP 162 | Traps (alertes envoyees par l'agent vers le manager) |

### La MIB et les OID

Pour organiser les donnees exposees, SNMP s'appuie sur une structure hierarchique appelee **MIB** (Management Information Base). Chaque element supervisable est identifie par un **OID** (Object Identifier), une suite de chiffres separes par des points. Par exemple, `1.3.6.1.2.1.1.1.0` correspond a la description systeme de l'equipement.

La MIB est un fichier texte au format ASN.1 qui decrit chaque objet : son type, ses droits d'acces, sa description. Plus l'OID est long, plus l'objet cible est specifique dans l'arborescence.

### Versions du protocole

{% tabs %}
{% tab title="SNMPv1" %}
Premiere version du protocole. Aucune authentification, aucun chiffrement. Les community strings circulent en clair sur le reseau. Encore presente sur de vieux equipements.
{% endtab %}

{% tab title="SNMPv2c" %}
Version "community-based". Elle apporte des ameliorations fonctionnelles (bulk requests, meilleure gestion d'erreurs) mais ne corrige pas le probleme fondamental : les community strings restent transmises en clair.

{% hint style="warning" %}
SNMPv2c est la version la plus repandue en entreprise. La fausse impression de securite qu'elle donne (parce que "plus recente que v1") en fait un piege classique.
{% endhint %}
{% endtab %}

{% tab title="SNMPv3" %}
Seule version qui introduit une vraie securite : authentification par utilisateur/mot de passe et chiffrement des echanges (mode `authPriv`). La contrepartie est une configuration plus complexe, ce qui explique que beaucoup d'administrateurs restent sur v2c.

{% hint style="success" %}
En audit, verifier la version SNMP deployee est une des premieres choses a faire. SNMPv1/v2c avec une community string par defaut, c'est un acces direct a l'inventaire reseau.
{% endhint %}
{% endtab %}
{% endtabs %}

### Community strings

Les community strings fonctionnent comme des mots de passe d'acces aux donnees SNMP. Deux valeurs par defaut reviennent constamment :

| Community string | Acces |
|---|---|
| `public` | Lecture seule |
| `private` | Lecture/ecriture |

{% hint style="danger" %}
En SNMPv1 et v2c, ces chaines transitent en clair sur le reseau. Un attaquant positionne sur le meme segment peut les intercepter avec un simple sniffeur. Si `private` est accessible, il peut modifier la configuration des equipements a distance.
{% endhint %}

## En pratique

### Parcourir l'arbre MIB avec snmpwalk

`snmpwalk` est l'outil de reference pour interroger un agent SNMP et parcourir l'ensemble de son arborescence MIB.

```bash
# Depuis Exegol - enumeration complete de l'arbre MIB
snmpwalk -v2c -c public <IP_CIBLE>
```

La sortie peut etre volumineuse. On y trouve typiquement :

```bash
iso.3.6.1.2.1.1.1.0 = STRING: "Linux target 5.4.0-xx-generic #xx-Ubuntu SMP ..."
iso.3.6.1.2.1.1.4.0 = STRING: "admin@entreprise.local"
iso.3.6.1.2.1.1.5.0 = STRING: "srv-internal"
iso.3.6.1.2.1.1.6.0 = STRING: "Datacenter B - Rack 12"
```

Ces informations revelent le systeme d'exploitation, le contact administrateur, le nom d'hote et la localisation physique de l'equipement.

### Brute-force des community strings avec onesixtyone

Quand la community string n'est pas `public`, on peut tenter un brute-force avec une wordlist dediee :

```bash
# Depuis Exegol - brute-force des community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP_CIBLE>
```

L'outil envoie des requetes SNMP avec chaque community string de la liste et affiche celles qui obtiennent une reponse.

### Enumeration ciblee avec braa

`braa` permet d'interroger des OID specifiques en masse, ce qui est plus rapide que `snmpwalk` quand on sait deja ce qu'on cherche :

```bash
# Depuis Exegol - interrogation d'une branche OID specifique
braa public@<IP_CIBLE>:.1.3.6.*
```

### Configuration par defaut (snmpd)

Sur un serveur Linux, le daemon SNMP se configure dans `/etc/snmp/snmpd.conf`. Un extrait typique :

```bash
sysLocation    Sitting on the Dock of the Bay
sysContact     Me <me@example.org>
sysServices    72
master  agentx
agentaddress  127.0.0.1,[::1]
view   systemonly  included   .1.3.6.1.2.1.1
rocommunity  public default -V systemonly
```

{% hint style="info" %}
La directive `rocommunity public default -V systemonly` limite la lecture a la branche `system` de la MIB. Mais si un administrateur elargit la vue ou ajoute `rwcommunity`, l'exposition devient bien plus large.
{% endhint %}

## Pieges et galeres

| Parametre dangereux | Consequence |
|---|---|
| `rwuser noauth` | Acces complet en lecture/ecriture sans aucune authentification |
| `rwcommunity <string> <IP>` | Acces en ecriture depuis une IP ou une plage, potentiellement trop large |
| `rocommunity public` sans restriction de vue | Expose l'ensemble de l'arbre MIB, y compris les processus, paquets, interfaces |

Un point souvent oublie : meme en lecture seule, l'enumeration SNMP peut reveler des informations critiques. Les processus actifs peuvent trahir des services non exposes par les scans de ports. Les paquets installes donnent la version exacte du systeme, facilitant la recherche de CVE. Les interfaces reseau exposent des segments auxquels on n'avait pas acces directement.

## Retour terrain

En situation reelle, SNMP est souvent le premier reflexe apres un scan de ports qui revele UDP 161 ouvert. Le scenario classique :

1. On teste `public` en community string avec `snmpwalk`
2. Si ca repond, on recupere tout l'arbre MIB
3. On extrait les informations cles : nom d'hote, contacts, interfaces, processus, paquets
4. Si `private` fonctionne aussi, on a potentiellement un acces en ecriture sur la configuration des equipements

Le brute-force avec `onesixtyone` est rapide et silencieux (UDP, pas de connexion TCP a etablir). C'est un outil a garder dans sa routine d'enumeration, surtout sur les reseaux internes ou les equipements reseau (switches, routeurs) sont rarement audites.

{% hint style="warning" %}
Les scanners de vulnerabilites automatises passent souvent a cote de SNMP parce qu'ils se concentrent sur TCP. Penser a toujours inclure un scan UDP cible (`nmap -sU -p161`) dans la phase de decouverte.
{% endhint %}

## Memo express

| Action | Commande |
|---|---|
| Enumeration complete | `snmpwalk -v2c -c public <IP_CIBLE>` |
| Brute-force community strings | `onesixtyone -c wordlist.txt <IP_CIBLE>` |
| Enumeration ciblee par OID | `braa public@<IP_CIBLE>:.1.3.6.*` |
| Scan Nmap UDP | `sudo nmap -sU -p161 -sV <IP_CIBLE>` |

***
