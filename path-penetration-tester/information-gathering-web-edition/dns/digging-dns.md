# Interroger le DNS

## Pourquoi

Savoir interroger le DNS manuellement, c'est la base de la reconnaissance réseau. Avant de lancer des outils automatisés, il faut maîtriser les commandes qui permettent de poser les bonnes questions aux serveurs DNS. `dig`, `nslookup` et `host` sont les trois outils incontournables pour ça.

## Comment ça marche

### Les outils de requête DNS

| Outil | Points forts | Cas d'usage typique |
|---|---|---|
| `dig` | Sortie détaillée, contrôle fin sur tous les types de requêtes | Analyse DNS manuelle, debug, transferts de zone |
| `nslookup` | Simple, rapide, disponible partout (y compris Windows) | Vérification rapide de résolution A, MX |
| `host` | Sortie concise, idéal pour le scripting | Requêtes rapides A, AAAA, MX |
| `dnsenum` | Énumération automatisée, brute-force de sous-domaines | Découverte de sous-domaines à grande échelle |
| `fierce` | Recherche récursive, détection de wildcard | Identification de cibles secondaires |
| `dnsrecon` | Multi-techniques, export JSON/CSV | Énumération DNS complète |
| `theHarvester` | OSINT, collecte d'emails et de noms de domaine | Renseignement ciblé |

## En pratique

### dig : l'outil de référence

`dig` (Domain Information Groper) est l'outil le plus polyvalent pour interroger le DNS. Il permet de cibler n'importe quel type d'enregistrement et de contrôler finement le comportement de la requête.

#### Commandes essentielles

| Commande | Description |
|---|---|
| `dig <DOMAINE>` | Requête A par défaut |
| `dig <DOMAINE> A` | Enregistrement IPv4 |
| `dig <DOMAINE> AAAA` | Enregistrement IPv6 |
| `dig <DOMAINE> MX` | Serveurs de messagerie |
| `dig <DOMAINE> NS` | Serveurs DNS autoritaires |
| `dig <DOMAINE> TXT` | Enregistrements texte (SPF, DKIM, vérification) |
| `dig <DOMAINE> CNAME` | Alias DNS |
| `dig <DOMAINE> SOA` | Informations sur la zone |
| `dig @1.1.1.1 <DOMAINE>` | Interroger un serveur DNS spécifique |
| `dig +trace <DOMAINE>` | Afficher le chemin complet de résolution |
| `dig -x <IP>` | Résolution inverse (PTR) |
| `dig +short <DOMAINE>` | Sortie courte (juste les valeurs) |
| `dig +noall +answer <DOMAINE>` | Afficher uniquement les réponses |

#### Lire la sortie de dig

```
$ dig exemple.com

; <<>> DiG 9.18.24 <<>> exemple.com
;; ->>HEADER<<- opcode: QUERY, status: NOERROR
;; flags: qr rd ad; QUERY: 1, ANSWER: 1

;; QUESTION SECTION:
;exemple.com.    IN    A

;; ANSWER SECTION:
exemple.com. 300    IN    A    93.184.216.34

;; Query time: 12 msec
;; SERVER: 172.23.176.1#53
```

La sortie se décompose en plusieurs sections :

- **Header** : le status `NOERROR` indique que le domaine a été résolu avec succès. Un `NXDOMAIN` signifierait que le domaine n'existe pas.
- **Question Section** : rappelle la requête envoyée (ici, un enregistrement A pour `exemple.com`).
- **Answer Section** : contient la réponse. Ici, l'adresse IPv4 `93.184.216.34` avec un TTL de 300 secondes.
- **Query time** : le temps de réponse en millisecondes.
- **Server** : le résolveur DNS qui a répondu.

### Sortie minimale avec +short

Pour du scripting ou une vérification rapide, `+short` ne renvoie que la valeur :

```bash
$ dig +short <DOMAINE_CIBLE>
104.18.20.126
104.18.21.126
```

### Résolution inverse

Pour trouver le nom de domaine associé à une adresse IP :

```bash
$ dig -x <IP_CIBLE>
```

La réponse apparaît dans un enregistrement PTR. Cette technique est utile pour vérifier si une IP pointe vers le domaine attendu ou pour identifier un hébergement mutualisé.

### Requêtes MX pour les serveurs mail

```bash
$ dig <DOMAINE_CIBLE> MX
```

Les enregistrements MX révèlent les serveurs de messagerie du domaine. La valeur numérique (priorité) indique l'ordre de préférence : plus le chiffre est bas, plus le serveur est prioritaire.

## Pièges et galères

- **Requêtes `ANY` souvent bloquées** : de nombreux serveurs DNS refusent les requêtes de type `ANY` pour des raisons de performance et de sécurité. Ne pas s'y fier pour une énumération complète.
- **Cache DNS** : les réponses peuvent être mises en cache à plusieurs niveaux. Un résultat ne reflète pas forcément l'état actuel de la configuration DNS.
- **Rate limiting** : certains serveurs DNS limitent le nombre de requêtes par IP. Espacer les requêtes si nécessaire.

## Mémo express

| Besoin | Commande |
|---|---|
| IP d'un domaine | `dig +short <domaine>` |
| Serveurs mail | `dig <domaine> MX` |
| Serveurs DNS | `dig <domaine> NS` |
| Résolution inverse | `dig -x <ip>` |
| Trace complète | `dig +trace <domaine>` |
| Serveur DNS spécifique | `dig @8.8.8.8 <domaine>` |

***
