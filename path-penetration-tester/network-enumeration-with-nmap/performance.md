# Optimisation des performances

## Pourquoi

Un scan Nmap sur un réseau entier peut durer des dizaines de minutes si tu laisses les réglages par défaut. En pentest, le temps est compté : tu dois trouver le bon équilibre entre vitesse et fiabilité. Aller trop vite, c'est rater des hôtes ou des ports. Aller trop lentement, c'est perdre un temps précieux — et parfois se faire repérer par un IDS qui voit un flux anormalement régulier.

Nmap propose plusieurs leviers pour ajuster la cadence d'un scan : les délais de réponse (RTT), le nombre de tentatives, le débit minimal et les templates de timing.

## Comment ça marche

### RTT — Round-Trip Time

Le RTT, c'est le temps qu'un paquet met à faire l'aller-retour entre ta machine et la cible. Nmap démarre avec un RTT initial de 100 ms et l'ajuste dynamiquement en fonction des réponses reçues. Deux options permettent de forcer des valeurs :

| Option | Rôle |
|---|---|
| `--initial-rtt-timeout` | Définit le délai d'attente initial pour chaque paquet |
| `--max-rtt-timeout` | Plafonne le délai maximal que Nmap s'autorise |

En réduisant ces valeurs, tu forces Nmap à abandonner plus vite les hôtes lents — le scan accélère, mais au prix de faux négatifs potentiels.

### Retries — tentatives de renvoi

Quand un port ne répond pas, Nmap renvoie le paquet. Par défaut, il retente jusqu'à 10 fois (`--max-retries 10`). Sur un réseau fiable, 0 ou 1 retry suffit largement. Sur un lien instable (VPN, Wi-Fi), garde au moins 2-3 tentatives.

### Min-rate — débit plancher

L'option `--min-rate` impose un nombre minimal de paquets envoyés par seconde. Nmap ne descendra jamais en dessous de ce seuil, même si le réseau est lent. Un `--min-rate 300` divise facilement le temps de scan par 3 ou 4.

### Timing templates — les profils `-T`

Nmap embarque 6 profils prédéfinis qui ajustent simultanément le RTT, les retries, le parallélisme et les délais inter-paquets :

| Profil | Nom | Usage typique |
|---|---|---|
| `-T0` | paranoid | Évasion IDS, un paquet toutes les 5 minutes |
| `-T1` | sneaky | Évasion IDS, plus rapide que T0 |
| `-T2` | polite | Réduction de la charge réseau |
| `-T3` | normal | Profil par défaut |
| `-T4` | aggressive | Réseau fiable, pentest classique |
| `-T5` | insane | Réseau local rapide, précision réduite |

## En pratique

### Comparer l'impact du RTT

```bash
# Depuis Exegol — scan par défaut
sudo nmap 10.10.10.0/24 -F
# Durée : ~40s, 10 hôtes détectés
```

```bash
# Depuis Exegol — RTT réduit
sudo nmap 10.10.10.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms
# Durée : ~12s, 8 hôtes détectés (2 hôtes lents ratés)
```

### Réduire les retries

```bash
# Depuis Exegol — sans retry
sudo nmap 10.10.10.0/24 -F --max-retries 0
# Plus rapide, mais quelques ports silencieux disparaissent du résultat
```

### Forcer un débit minimal

```bash
# Depuis Exegol — débit plancher à 300 paquets/s
sudo nmap 10.10.10.0/24 -F --min-rate 300
# Durée divisée par 3-4, résultats quasi identiques sur un réseau stable
```

### Utiliser un timing template agressif

```bash
# Depuis Exegol — profil T4 (agressif)
sudo nmap <IP_CIBLE> -p- -T4 -oN scan-rapide.nmap
```

```bash
# Depuis Exegol — profil T5 (insane) sur réseau local
sudo nmap 10.10.10.0/24 -F -T5
# Très rapide, mais risque de faux négatifs sur les ports lents
```

## Pièges & galères

- **Faux négatifs** : réduire le RTT ou les retries fait rater les hôtes qui répondent lentement (serveurs chargés, liens saturés, pare-feux qui droppent puis laissent passer). En cas de doute, relance un scan ciblé sur les hôtes suspects avec les paramètres par défaut.
- **Détection IDS** : un `-T5` ou un `--min-rate` élevé génère un pic de trafic évident. Sur un réseau surveillé, un IDS type Suricata va lever une alerte en quelques secondes. Privilégie `-T2` ou `-T3` en phase de discrétion.
- **Fausse économie** : accélérer un scan de découverte n'a de sens que si tu repasses ensuite en mode précis sur les cibles identifiées. Un `-T5 -p-` seul ne remplace pas un `-sV -sC` ciblé.

## Retour terrain

En pratique, la combinaison qui fonctionne le mieux sur un réseau d'entreprise classique :

1. **Découverte rapide** : `nmap -sn -T4 --min-rate 300 10.10.10.0/24` pour lister les hôtes actifs
2. **Scan de ports ciblé** : `nmap -p- -T4 --max-retries 2 <IP_CIBLE>` sur chaque hôte identifié
3. **Énumération fine** : `nmap -sV -sC -p <PORTS> <IP_CIBLE>` avec les paramètres par défaut

Sur un VPN vers un lab distant, le RTT est souvent autour de 30-80 ms. Un `--initial-rtt-timeout 80ms --max-rtt-timeout 200ms` donne de bons résultats sans trop de pertes. En revanche, sur un réseau local (LAN), tu peux pousser à `--min-rate 1000` sans problème.

## Mémo express

| Option | Effet | Risque |
|---|---|---|
| `--initial-rtt-timeout <ms>` | Réduit le délai initial d'attente | Faux négatifs sur hôtes lents |
| `--max-rtt-timeout <ms>` | Plafonne le délai max | Idem |
| `--max-retries <n>` | Limite les renvois de paquets | Ports silencieux ratés |
| `--min-rate <n>` | Impose un débit plancher (paquets/s) | Détection IDS, saturation réseau |
| `-T0` à `-T5` | Profils de timing prédéfinis | T4-T5 : bruyant, T0-T1 : très lent |
| `--host-timeout <s>` | Abandonne un hôte après N secondes | Hôtes complexes ignorés |
| `--scan-delay <ms>` | Pause entre chaque paquet | Ralentit considérablement le scan |

***
