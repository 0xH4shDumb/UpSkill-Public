# DNS (Domain Name System)

Le DNS est le protocole qui convertit les noms de domaine en adresses IP. C'est la couche qui fait fonctionner à peu près tout sur Internet, et c'est aussi l'une des plus riches en informations pour un attaquant. Un serveur DNS mal configuré peut exposer la totalité de la cartographie interne d'une infrastructure.

## Pourquoi

Pendant un test d'intrusion, le DNS est souvent le premier service interrogé. Il ne se limite pas à résoudre des noms : il révèle les serveurs de messagerie, les nameservers, les sous-domaines, les intégrations tierces et parfois même des données sensibles planquées dans des enregistrements TXT. Un transfert de zone autorisé par erreur, et c'est l'inventaire complet du réseau interne qui tombe.

En phase de reconnaissance, croiser les résultats DNS avec d'autres sources (certificats, OSINT, Shodan) permet de construire une carte précise de l'infrastructure cible sans envoyer un seul paquet directement.

## Comment ca marche

### Architecture distribuée

Le DNS repose sur une hiérarchie de serveurs interconnectés. Il n'y a pas de base de données centrale : la résolution d'un nom traverse plusieurs niveaux, de la racine jusqu'au serveur qui fait autorité sur le domaine ciblé.

Pour un nom comme `ws01.dev.example.com`, la résolution passe par :

```
. (racine)
├── com (TLD)
│   └── example (domaine de second niveau)
│       └── dev (sous-domaine)
│           └── ws01 (hote)
```

### Types de serveurs

| Type | Role |
|---|---|
| Root Server | Serveurs racine qui orientent vers les TLD (`.com`, `.org`, etc.). 13 groupes dans le monde, coordonnés par l'ICANN. |
| Authoritative Name Server | Fait autorité pour une zone donnée. Ses réponses sont considérées comme définitives. |
| Non-Authoritative Server | Renvoie des réponses en cache, obtenues via d'autres serveurs. |
| Caching Server | Stocke temporairement les réponses DNS pour accélérer les résolutions suivantes. |
| Forwarding Server | Transmet les requêtes à un autre serveur DNS au lieu de les résoudre lui-même. |
| Resolver | Point d'entrée local (machine, box, réseau d'entreprise) qui initie la résolution. |

### Types d'enregistrements

| Enregistrement | Role |
|---|---|
| `A` | Associe un domaine à une adresse IPv4 |
| `AAAA` | Associe un domaine à une adresse IPv6 |
| `MX` | Indique les serveurs de messagerie du domaine |
| `NS` | Spécifie les nameservers de la zone |
| `TXT` | Données textuelles libres : SPF, DKIM, DMARC, vérifications tierces |
| `CNAME` | Alias vers un autre nom de domaine (Canonical Name) |
| `PTR` | Résolution inverse : IP vers nom de domaine |
| `SOA` | Informations sur la zone DNS (administrateur, numéro de série, TTL) |

{% hint style="info" %}
Les enregistrements TXT sont une mine d'or en reconnaissance. On y trouve régulièrement des preuves d'intégration avec Google Workspace, Atlassian, Mailgun, Office 365 ou d'autres services tiers qui donnent des pistes d'attaque.
{% endhint %}

### Sécurité et chiffrement

Par défaut, le DNS fonctionne en clair sur le port UDP/TCP 53. Toutes les requêtes et réponses sont visibles en transit, ce qui permet à un attaquant en position de MITM ou à un FAI de surveiller (voire modifier) les résolutions.

Plusieurs protocoles visent à combler ce manque :

- **DNS over TLS (DoT)** : chiffrement TLS sur le port 853
- **DNS over HTTPS (DoH)** : requêtes DNS encapsulées dans du HTTPS
- **DNSCrypt** : chiffrement et authentification des échanges DNS

{% hint style="warning" %}
Ces protections concernent le transit entre le client et le resolver. Un serveur DNS autoritaire mal configuré reste exploitable indépendamment du chiffrement côté client.
{% endhint %}

## Comment c'est configuré

### BIND9

BIND9 est l'implémentation DNS la plus répandue sous Linux. Sa configuration s'articule autour de trois éléments :

- `named.conf` : configuration générale du service
- Fichiers de zone directe : résolution nom vers IP
- Fichiers de zone inverse : résolution IP vers nom

#### Déclaration d'une zone

```ini
zone "example.com" {
    type master;
    file "/etc/bind/db.example.com";
    allow-update { key rndc-key; };
};
```

#### Fichier de zone directe

```dns
$ORIGIN example.com.
@ IN SOA dns1.example.com. hostmaster.example.com. (
    2024060101 ; serial
    3600       ; refresh
    1800       ; retry
    604800     ; expire
    86400 )    ; minimum TTL

  IN NS  ns1.example.com.
  IN MX  10 mail.example.com.
www IN A  10.10.10.10
```

#### Fichier de zone inverse

```dns
$ORIGIN 10.10.10.in-addr.arpa.
@ IN SOA dns1.example.com. hostmaster.example.com. (
    2024060101 ; serial
    3600       ; refresh
    1800       ; retry
    604800     ; expire
    86400 )    ; minimum TTL

10 IN PTR www.example.com.
```

## Pieges et galeres

### Parametres sensibles

| Directive | Risque |
|---|---|
| `allow-query { any; };` | Le serveur répond à n'importe qui, y compris depuis Internet. Toute la zone devient publique. |
| `allow-transfer { any; };` | Autorise les transferts de zone AXFR sans restriction. Un attaquant récupère l'intégralité des enregistrements en une seule requête. |
| `allow-recursion { any; };` | Le serveur résout les requêtes pour tout le monde. Exploitable pour des attaques d'amplification DNS. |
| `zone-statistics yes;` | Expose des métadonnées sur l'activité de la zone (nombre de requêtes, types, etc.). |

{% hint style="danger" %}
Le transfert de zone (`allow-transfer { any; }`) est probablement la mauvaise configuration DNS la plus critique en pentest. Quand c'est ouvert, un simple `dig axfr` suffit à récupérer tous les enregistrements internes : noms d'hôtes, IPs, sous-domaines cachés. On tombe régulièrement sur des serveurs internes qui l'autorisent par défaut.
{% endhint %}

## En pratique

### Récupérer les nameservers d'un domaine

```bash
# Depuis Exegol - identifier les NS du domaine
dig ns example.com @<IP_CIBLE>
```

### Interroger tous les enregistrements disponibles

```bash
# Depuis Exegol - requete ANY
dig any example.com @<IP_CIBLE>
```

### Tenter un transfert de zone

```bash
# Depuis Exegol - transfert de zone AXFR
dig axfr example.com @<IP_CIBLE>
```

Si le transfert aboutit, on obtient la liste complète des enregistrements de la zone. Il faut aussi penser a tester les sous-domaines découverts : un transfert refusé sur le domaine principal peut fonctionner sur une zone enfant (`dev.example.com`, `internal.example.com`).

### Identifier la version du serveur DNS

```bash
# Depuis Exegol - requete Chaosnet pour la version
dig CH TXT version.bind @<IP_CIBLE>
```

Certains serveurs répondent avec leur version BIND exacte, ce qui oriente directement vers des CVE connues.

### Brute-force de sous-domaines

Quand le transfert de zone est bloqué, le brute-force de sous-domaines reste une option solide :

{% tabs %}
{% tab title="Boucle Bash" %}
```bash
# Depuis Exegol - enumeration par dictionnaire
for sub in $(cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt); do
    dig $sub.example.com @<IP_CIBLE> +short | grep -v "^$" && echo "$sub.example.com"
done
```
{% endtab %}

{% tab title="dnsenum" %}
```bash
# Depuis Exegol - enumeration automatisée
dnsenum --dnsserver <IP_CIBLE> --enum -p 0 -s 0 \
  -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  example.com
```

`dnsenum` tente automatiquement les transferts de zone, le brute-force et la résolution inverse.
{% endtab %}
{% endtabs %}

### Enumeration avec Nmap

```bash
# Depuis Exegol - scripts NSE DNS
sudo nmap -p53 -sV -sC <IP_CIBLE>
```

Les scripts NSE DNS de Nmap (`dns-nsid`, `dns-service-discovery`, `dns-brute`) permettent de compléter l'énumération automatiquement.

## Retour terrain

L'erreur classique, c'est de se contenter d'un transfert de zone raté et de passer au service suivant. Le DNS mérite qu'on s'y attarde. Les enregistrements TXT trahissent les intégrations tierces (Google, Atlassian, Microsoft), les enregistrements MX pointent vers l'infrastructure mail, et le brute-force de sous-domaines fait régulièrement apparaitre des environnements de développement ou des interfaces d'administration oubliées.

Pense aussi à vérifier les sous-domaines découverts : chacun peut avoir sa propre zone DNS, avec ses propres transferts autorisés. Un domaine principal bien verrouillé peut masquer une zone `internal.` ou `dev.` grande ouverte.

## Memo express

| Action | Commande |
|---|---|
| Nameservers | `dig ns example.com @<IP_CIBLE>` |
| Tous les enregistrements | `dig any example.com @<IP_CIBLE>` |
| Transfert de zone | `dig axfr example.com @<IP_CIBLE>` |
| Version du serveur | `dig CH TXT version.bind @<IP_CIBLE>` |
| Brute-force sous-domaines | `dnsenum --dnsserver <IP_CIBLE> --enum -f wordlist.txt example.com` |
| Scan Nmap | `sudo nmap -p53 -sV -sC <IP_CIBLE>` |

***
