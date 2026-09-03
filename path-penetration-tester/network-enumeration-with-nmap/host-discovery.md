# Host Discovery

La découverte d'hôtes est la première étape de toute reconnaissance réseau. Avant d'énumérer des ports ou de chercher des vulnérabilités, il faut savoir **quelles machines sont actives** sur le périmètre ciblé. C'est ce qui va conditionner toute la suite de l'approche.

## Pourquoi

En test d'intrusion interne, on se retrouve souvent face à des plages IP entières sans savoir ce qui tourne dessus. Lancer un scan de ports sur chaque adresse d'un /24 sans tri préalable, c'est à la fois lent et bruyant : des milliers de paquets envoyés vers des adresses vides, pour un résultat noyé dans le bruit.

L'objectif de la découverte d'hôtes est de **réduire la surface de scan** aux machines réellement actives. On gagne du temps, on limite le trafic généré, et on produit un inventaire exploitable pour les étapes suivantes.

## Comment ça marche

Nmap propose plusieurs mécanismes de découverte. Chacun a ses avantages et ses limites, et le choix dépend du contexte réseau.

{% tabs %}
{% tab title="ARP" %}
**Réseau local uniquement**

Sur un même segment réseau, Nmap envoie par défaut des requêtes ARP. C'est le mécanisme le plus fiable : l'ARP est indispensable au fonctionnement de la couche réseau (résolution IP → MAC), il n'est donc quasiment jamais filtré. Si la machine est allumée et connectée au segment, elle répondra.

{% hint style="info" %}
L'ARP ne traverse pas les routeurs. Pour un scan à travers un pivot ou un tunnel, Nmap basculera automatiquement sur ICMP ou TCP.
{% endhint %}
{% endtab %}

{% tab title="ICMP" %}
**ICMP Echo Request / Echo Reply**

Nmap envoie un ICMP Echo Request (type 8) et attend un Echo Reply (type 0). Mécanisme simple et rapide, mais souvent filtré en entreprise, ce qui conduit à des faux négatifs : l'hôte est bien présent, mais il ne répond tout simplement pas au ping.

{% hint style="warning" %}
Le pare-feu Windows bloque les ICMP Echo par défaut dans les profils « Domaine » et « Public ». En environnement Active Directory, s'appuyer uniquement sur l'ICMP donne un inventaire incomplet.
{% endhint %}
{% endtab %}

{% tab title="TCP SYN/ACK" %}
**Découverte par paquets TCP**

Nmap envoie un paquet TCP SYN (port 443 par défaut) et un ACK (port 80). Si l'hôte répond par un SYN-ACK ou un RST, il est considéré comme actif. Cette méthode passe là où l'ICMP est bloqué, parce que le trafic web est rarement restreint.

Les ports sont configurables via `-PS` (SYN) et `-PA` (ACK) :

```bash
# Découverte par SYN sur les ports 22, 80 et 445
sudo nmap <IP_CIBLE> -sn -PS22,80,445
```

{% hint style="success" %}
En interne Windows, `-PS445` est souvent le plus efficace. Le port SMB est rarement filtré entre machines du même domaine.
{% endhint %}
{% endtab %}
{% endtabs %}

### Identification d'OS par le TTL

Le champ TTL (Time To Live) dans les réponses IP donne un indice sur le système d'exploitation distant. Chaque OS initialise le TTL à une valeur caractéristique :

| TTL initial | OS probable |
|---|---|
| 64 | Linux / macOS / FreeBSD |
| 128 | Windows |
| 255 | Équipement réseau (routeur, switch L3) |

{% hint style="info" %}
Le TTL est décrémenté de 1 à chaque routeur traversé. Un TTL de 62 reçu sur un réseau à 2 sauts correspond probablement à un Linux (64 − 2 = 62). Ce n'est pas un diagnostic définitif, mais c'est un indicateur rapide qu'on peut exploiter avant même un scan de version.
{% endhint %}

## En pratique

### Scanner un sous-réseau complet

```bash
# Depuis Exegol - ping sweep sur un /24
sudo nmap 10.10.10.0/24 -sn -oA discovery
```

L'option `-sn` désactive le scan de ports : Nmap se contente de vérifier quels hôtes répondent. Les résultats sont sauvegardés dans les trois formats (normal, grepable, XML) grâce à `-oA`.

Pour extraire les adresses actives :

```bash
# Filtrer les hôtes vivants
grep "for" discovery.nmap | cut -d" " -f5
```

### Scanner depuis une liste d'IP

Quand on dispose déjà d'une liste de cibles (export DHCP, extraction Active Directory, résultat d'un scan précédent) :

```bash
# Depuis Exegol - scan depuis un fichier de cibles
sudo nmap -sn -oA targeted -iL targets.txt
```

Le fichier `targets.txt` contient une adresse par ligne. Ça permet de limiter le scan aux hôtes pertinents sans balayer tout le sous-réseau.

### Scanner plusieurs IPs sans fichier

Pour un nombre limité de cibles, pas besoin de créer un fichier. Nmap accepte les adresses en ligne ou la syntaxe de plage :

```bash
# Depuis Exegol - liste explicite
sudo nmap -sn -oA multi 10.10.10.5 10.10.10.12 10.10.10.30
```

```bash
# Depuis Exegol - syntaxe de plage (adresses .18 à .25)
sudo nmap -sn -oA range 10.10.10.18-25
```

La syntaxe de plage fonctionne sur n'importe quel octet et évite de lister chaque adresse manuellement.

### Vérifier qu'un hôte est actif

Avant de lancer un scan de ports complet, vérifier que la cible est en ligne évite des attentes inutiles :

```bash
# Depuis Exegol - vérification rapide
sudo nmap <IP_CIBLE> -sn -oA host_check
```

### Forcer l'utilisation d'ICMP

Sur un réseau local, Nmap privilégie l'ARP par défaut. Pour forcer l'ICMP (utile pour tester si les paquets ICMP traversent les filtres) :

```bash
# Depuis Exegol - ICMP forcé, ARP désactivé
sudo nmap <IP_CIBLE> -sn -PE --disable-arp-ping --packet-trace
```

L'option `--packet-trace` affiche chaque paquet envoyé et reçu. C'est indispensable pour analyser ce qui se passe réellement sur le réseau quand les résultats ne correspondent pas à ce qu'on attend.

### Analyser la raison de détection

```bash
# Depuis Exegol - afficher le mécanisme de détection utilisé
sudo nmap <IP_CIBLE> -sn -PE --reason --disable-arp-ping
```

Le flag `--reason` montre sur quel mécanisme Nmap s'est appuyé pour conclure (réponse ARP, ICMP Echo Reply, RST TCP…). Quand un hôte connu apparaît comme « down », c'est la première option à ajouter pour comprendre ce qui bloque.

### Désactiver la découverte ARP

Dans certains contextes (scan à travers un tunnel, test de contournement de pare-feu), on veut s'assurer que seuls les mécanismes qui traversent les routeurs sont utilisés :

```bash
# Depuis Exegol - ICMP uniquement, trace complète
sudo nmap <IP_CIBLE> -sn -PE --packet-trace --disable-arp-ping
```

## Pièges & galères

{% hint style="danger" %}
**ICMP bloqué = inventaire incomplet.** En environnement durci, de nombreuses machines ne répondent pas au ping. Un `-sn` basé uniquement sur l'ICMP produira un inventaire lacunaire. Combiner systématiquement avec une découverte TCP sur des ports courants (`-PS22,80,443,445`).
{% endhint %}

{% hint style="warning" %}
**Portée limitée de l'ARP.** Les requêtes ARP ne fonctionnent que sur le même broadcast domain. En cas de scan à travers un pivot, penser à vérifier avec `--packet-trace` que Nmap utilise bien ICMP ou TCP.
{% endhint %}

{% hint style="warning" %}
**Détectabilité du ping sweep.** Même sans scan de ports, un balayage `-sn` sur un /24 génère 254 requêtes quasi simultanées. Les IDS/IPS détectent facilement ce schéma. Sur un réseau supervisé, réduire la cadence avec `-T2` ou découper en plages plus petites.
{% endhint %}

{% hint style="warning" %}
**Faux positifs ARP.** Certains équipements (load balancers, proxies ARP) répondent pour des adresses IP qui ne correspondent à aucune machine réelle. En cas de doute, confirmer avec un scan TCP ciblé.
{% endhint %}

## Retour terrain

{% hint style="success" %}
**Approche en entonnoir** : un `-sn` rapide pour la vue d'ensemble, puis des scans ciblés sur les hôtes identifiés. C'est ce qui donne le meilleur ratio temps/couverture.
{% endhint %}

{% hint style="success" %}
**Sauvegarde systématique** : `-oA` avec un nommage explicite (ex: `discovery-subnet42-20250903`) doit devenir un réflexe. Les fichiers de sortie permettent de comparer les scans dans le temps et d'alimenter les annexes du rapport.
{% endhint %}

{% hint style="success" %}
**Port 445 en interne** : quand l'ICMP est bloqué, `-PS445` détecte la majorité des machines d'un domaine Active Directory. C'est souvent le point d'entrée le plus fiable en reconnaissance interne.
{% endhint %}

- `--reason` est le premier réflexe de diagnostic quand un scan donne des résultats inattendus : la raison identifie immédiatement si le problème vient d'un filtrage, d'un timeout ou d'une absence de réponse ARP.
- Sur les périmètres étendus (/16 ou plus), découper en sous-réseaux et paralléliser est plus efficace qu'un balayage monolithique.
- Croiser les résultats Nmap avec d'autres sources (table ARP du routeur, logs DHCP, DNS inversé) donne un inventaire plus complet. La découverte d'hôtes ne voit que les machines qui répondent au moment du scan, pas celles qui étaient éteintes ou temporairement injoignables.

## Mémo express

| Option | Rôle |
|---|---|
| `-sn` | Ping scan uniquement, pas de scan de ports |
| `-iL fichier` | Charger les cibles depuis un fichier |
| `-PE` | Forcer l'envoi d'ICMP Echo Request |
| `--disable-arp-ping` | Désactiver la découverte ARP |
| `-PS<ports>` | Découverte par TCP SYN sur les ports spécifiés |
| `-PA<ports>` | Découverte par TCP ACK sur les ports spécifiés |
| `--packet-trace` | Afficher tous les paquets envoyés/reçus |
| `--reason` | Afficher le mécanisme de détection utilisé |
| `-oA nom` | Sauvegarder dans les 3 formats (nmap, gnmap, xml) |
| `-T<0-5>` | Ajuster la cadence du scan (0=furtif, 5=rapide) |

<details>
<summary>Ressources complémentaires</summary>

- [Nmap Host Discovery Strategies](https://nmap.org/book/host-discovery-strategies.html) : documentation officielle sur les méthodes de découverte
- [Nmap Reference Guide](https://nmap.org/book/man-host-discovery.html) : référence complète des options de host discovery

</details>

***
