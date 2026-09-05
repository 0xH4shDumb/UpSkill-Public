# Attaquer DNS

Le Domain Name System (DNS) traduit les noms de domaine en adresses IP. Il fonctionne principalement sur UDP/53, avec un fallback TCP/53 pour les réponses volumineuses et les transferts de zone. Les attaques DNS vont de l'énumération passive (zone transfer, brute force de sous-domaines) aux attaques actives (subdomain takeover, DNS spoofing/cache poisoning).

## Pourquoi

Le DNS est la colonne vertébrale de la résolution de noms dans tout réseau. Un transfert de zone mal protégé expose l'intégralité de l'infrastructure interne d'une organisation. Les sous-domaines oubliés pointant vers des services tiers désactivés permettent des prises de contrôle (subdomain takeover). En pentest, l'énumération DNS est souvent le premier pas pour cartographier la surface d'attaque d'une cible.

## Comment ça marche

### Énumération

```bash
# - Scan du service DNS
nmap -p53 -Pn -sCV <IP_CIBLE>
```

### Transfert de zone (AXFR)

Un transfert de zone DNS permet de copier l'intégralité des enregistrements d'une zone depuis un serveur DNS. C'est un mécanisme légitime de réplication entre serveurs DNS, mais s'il n'est pas restreint par une liste blanche d'adresses IP, n'importe qui peut demander une copie complète de la zone.

```bash
# - Tentative de transfert de zone
dig AXFR @<IP_DNS> domaine.htb
```

Un transfert réussi expose tous les enregistrements : sous-domaines, adresses IP internes, enregistrements MX, TXT (parfois avec des tokens ou des clés), SRV, etc.

On peut aussi utiliser Fierce pour automatiser le test sur tous les serveurs DNS d'un domaine :

```bash
# - Énumération DNS automatisée avec Fierce
fierce --domain domaine.com
```

### Brute force de sous-domaines

Si le transfert de zone échoue, le brute force de sous-domaines reste une option. Plusieurs outils existent :

{% tabs %}
{% tab title="Subfinder" %}
```bash
# - Énumération passive de sous-domaines (sources OSINT)
subfinder -d domaine.com -v
```

Subfinder interroge des sources passives (DNSdumpster, Censys, etc.) sans envoyer de requêtes directes au serveur DNS cible.
{% endtab %}
{% tab title="Subbrute" %}
```bash
# - Brute force DNS avec résolveurs personnalisés
echo "ns1.domaine.htb" > resolvers.txt
subbrute.py domaine.htb -s noms.txt -r resolvers.txt
```

Subbrute permet de spécifier ses propres résolveurs DNS, ce qui est utile en pentest interne quand les machines n'ont pas accès à Internet.
{% endtab %}
{% endtabs %}

### Subdomain Takeover

Le subdomain takeover exploite des enregistrements DNS (souvent des CNAME) pointant vers des services tiers qui ne sont plus actifs.

Le scénario type :
1. L'entreprise crée un CNAME `support.domaine.com` pointant vers `domaine.s3.amazonaws.com`
2. Le bucket S3 est supprimé, mais l'enregistrement DNS reste en place
3. Un attaquant crée un bucket S3 avec le même nom et prend le contrôle du sous-domaine

```bash
# - Vérifier si un sous-domaine est vulnérable au takeover
host -t CNAME support.domaine.com

# Si la réponse pointe vers un service qui retourne une erreur
# (NoSuchBucket, 404 GitHub Pages, etc.), le takeover est possible
```

{% hint style="danger" %}
Le subdomain takeover permet de servir du contenu malveillant sous un sous-domaine légitime de l'entreprise. Les impacts possibles : phishing crédible, vol de cookies (si le domaine parent a des cookies sur `*.domaine.com`), contournement de CSP, abus de CORS.
{% endhint %}

Le repository [can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz) référence les services tiers vulnérables au takeover et les signatures permettant de les identifier.

### DNS Spoofing / Cache Poisoning

Le DNS spoofing consiste à injecter de faux enregistrements dans le cache DNS d'une victime pour rediriger son trafic. Deux approches principales :

**Via MITM (réseau local) :**

Des outils comme Ettercap ou Bettercap permettent d'empoisonner le cache DNS d'une cible en interceptant et en répondant aux requêtes DNS avant le serveur légitime.

```bash
# - Configuration Ettercap pour DNS spoofing
# Éditer /etc/ettercap/etter.dns :
# domaine.com      A   <IP_ATTAQUANT>
# *.domaine.com    A   <IP_ATTAQUANT>
```

Ensuite, lancer Ettercap avec le plugin `dns_spoof` activé, en ciblant la machine victime (Target1) et la passerelle (Target2).

**Via exploitation du serveur DNS :**

Si on compromet le serveur DNS lui-même, on peut modifier directement les enregistrements. Plus rare mais plus impactant, car cela affecte tous les clients de ce serveur.

{% hint style="info" %}
Le DNS spoofing local (via MITM) est surtout utile pour démontrer un risque en pentest interne. Il nécessite un positionnement sur le même segment réseau que la victime. Les protections comme DNSSEC rendent le cache poisoning à distance beaucoup plus difficile.
{% endhint %}

## En pratique

```bash
# 1 - Identifier le serveur DNS et sa version
nmap -p53 -Pn -sCV <IP_CIBLE>

# 2 - Récupérer les enregistrements publics
dig any domaine.htb @<IP_DNS>

# 3 - Tenter un transfert de zone
dig AXFR @<IP_DNS> domaine.htb

# 4 - Brute force de sous-domaines (si pas de transfert)
subfinder -d domaine.com
# ou en interne :
subbrute.py domaine.htb -s wordlist.txt -r resolvers.txt

# 5 - Vérifier les CNAME pour le subdomain takeover
for sub in $(cat sous-domaines.txt); do
    host -t CNAME "$sub"
done

# 6 - DNS spoofing local (nécessite positionnement MITM)
# Configurer etter.dns puis lancer Ettercap avec dns_spoof
```

## Pièges et galères

{% tabs %}
{% tab title="Zone transfer" %}
- **TCP requis** : les transferts de zone utilisent TCP/53, pas UDP. S'assurer que le port TCP est ouvert dans le scan
- **Sous-zones** : un transfert de zone sur le domaine principal peut ne rien révéler d'intéressant, mais un transfert sur un sous-domaine découvert par brute force peut exposer une zone interne complète
- **Restriction par IP** : la majorité des serveurs DNS publics restreignent les transferts de zone. C'est plus courant de trouver des transferts ouverts sur des serveurs DNS internes
{% endtab %}
{% tab title="Subdomain takeover" %}
- **Faux positifs** : un CNAME pointant vers un service tiers ne signifie pas automatiquement que le takeover est possible. Il faut vérifier que le service est effectivement non revendiqué
- **Rapidité** : en bug bounty, les subdomain takeovers sont très compétitifs. D'autres chercheurs peuvent revendiquer le sous-domaine avant vous
- **Impact limité** : certains programmes de bug bounty ne considèrent pas le subdomain takeover comme une vulnérabilité s'il n'y a pas de démonstration d'impact concret
{% endtab %}
{% tab title="DNS spoofing" %}
- **DNSSEC** : si le domaine cible utilise DNSSEC, le cache poisoning à distance est quasi impossible. Le spoofing local (MITM) reste possible car il intervient avant la vérification DNSSEC
- **DoH / DoT** : DNS over HTTPS et DNS over TLS chiffrent les requêtes DNS, rendant l'interception et le spoofing beaucoup plus difficiles
{% endtab %}
{% endtabs %}

## Retour terrain

En pentest interne, le transfert de zone reste étonnamment fréquent sur les serveurs DNS Active Directory. C'est l'un des premiers tests à effectuer car il donne une vue complète du réseau en quelques secondes.

Le brute force de sous-domaines via subfinder (passif) suivi de résolutions manuelles est la méthode la plus efficace en pentest externe. Les subdomain takeovers sont plus pertinents en bug bounty qu'en pentest classique, mais il faut toujours vérifier les CNAME orphelins pendant la reconnaissance.

Le DNS spoofing local est rarement démontré en pentest (risque de perturbation du réseau), mais il fait partie des recommandations de sécurisation à inclure dans le rapport si le réseau n'est pas segmenté.

## Mémo express

| Technique | Outil / Commande | Prérequis |
|---|---|---|
| Zone transfer | `dig AXFR @<NS> domaine` | Transfert non restreint |
| Enum auto | `fierce --domain domaine` | Accès Internet ou résolveur |
| Brute force passif | `subfinder -d domaine` | Sources OSINT accessibles |
| Brute force actif | `subbrute.py domaine -s wordlist` | Résolveur DNS accessible |
| Subdomain takeover | `host -t CNAME sous.domaine` | CNAME vers service abandonné |
| DNS spoofing | Ettercap + `dns_spoof` | Positionnement MITM |

***
