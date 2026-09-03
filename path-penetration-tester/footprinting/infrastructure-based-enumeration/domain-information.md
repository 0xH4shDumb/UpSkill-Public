# Informations de domaine

La collecte d'informations de domaine est la première étape concrète du footprinting. Avant de lancer le moindre scan, on exploite les données publiques pour cartographier la présence Internet de la cible. L'avantage : tout se fait de manière passive, sans générer de trafic vers l'infrastructure visée.

## Pourquoi

Un nom de domaine n'est que la partie visible d'une infrastructure. Derrière, il y a des sous-domaines, des serveurs mail, des intégrations tierces, des environnements de développement, des API. Tout ce qui est enregistré dans le DNS ou référencé dans un certificat SSL est, par nature, de l'information publique. Et cette information dessine un plan très précis de l'architecture technique de la cible.

Collecter ces données avant de scanner quoi que ce soit permet de cibler l'énumération active sur les bons périmètres, et d'identifier des pistes que les scans réseau seuls ne révéleraient pas.

## Comment ça marche

### Certificate Transparency

Les certificats SSL/TLS sont enregistrés dans des journaux publics de transparence (Certificate Transparency Logs). Chaque certificat émis pour un domaine ou un sous-domaine y est référencé, ce qui permet de reconstituer une liste de sous-domaines sans envoyer la moindre requête à la cible.

```bash
# - extraire les sous-domaines depuis crt.sh
curl -s "https://crt.sh/?q=example.com&output=json" | jq -r '.[].name_value' | sort -u
```

Les résultats incluent souvent des sous-domaines internes qui n'étaient pas censés être publics : `dev.example.com`, `staging.example.com`, `vpn.example.com`. C'est exactement ce qu'on cherche.

### Résolution et identification des hôtes

Une fois la liste de sous-domaines en main, on résout chacun d'eux pour identifier les adresses IP derrière et déterminer ce qui est hébergé en interne (plage privée ou AS de l'entreprise) et ce qui est externalisé (CDN, cloud, fournisseur tiers).

```bash
# - résoudre chaque sous-domaine et filtrer ceux hébergés par la cible
for sub in $(cat subdomains.txt); do
  host "$sub" | grep "has address"
done
```

```bash
# - générer une liste d'IPs uniques pour un scan Shodan
cut -d" " -f4 resolved.txt | sort -u > ips.txt
```

### Informations Shodan

Shodan indexe les services exposés sur Internet. En interrogeant les IPs identifiées, on obtient la liste des ports ouverts, les bannières de services, les versions logicielles et parfois les technologies utilisées, le tout sans scanner la cible directement.

```bash
# - interroger Shodan pour chaque IP identifiée
for ip in $(cat ips.txt); do shodan host "$ip"; done
```

### Enregistrements DNS

Les enregistrements DNS sont une source d'information souvent sous-exploitée. Chaque type d'enregistrement raconte quelque chose sur l'infrastructure.

| Type | Ce que ça révèle |
|---|---|
| A / AAAA | Adresses IP des sous-domaines |
| MX | Infrastructure mail (interne ou externalisée) |
| NS | Serveurs de noms (hébergement DNS) |
| TXT | Intégrations tierces, vérifications, politique SPF |
| SOA | Contact administratif, informations de zone |

```bash
# depuis Exegol - récupérer tous les enregistrements publics
dig any example.com
```

{% hint style="info" %}
Les enregistrements TXT sont particulièrement intéressants. On y trouve des tokens de vérification Google, Atlassian, Microsoft, des configurations SPF qui listent les fournisseurs d'envoi d'email, et parfois des artefacts de configuration qui n'auraient pas dû rester en production.
{% endhint %}

## En pratique

La démarche complète pour une reconnaissance de domaine :

1. **Certificate Transparency** : extraire tous les sous-domaines connus via crt.sh
2. **Résolution DNS** : résoudre chaque sous-domaine et séparer les hébergements internes des externes
3. **Shodan / Censys** : interroger les IPs internes pour identifier les services exposés
4. **Enregistrements DNS** : récupérer A, MX, NS, TXT, SOA pour comprendre l'architecture
5. **Recoupement** : croiser les résultats pour construire une vue d'ensemble cohérente

```bash
# depuis Exegol - workflow condensé
curl -s "https://crt.sh/?q=example.com&output=json" | jq -r '.[].name_value' | sort -u > subs.txt
for sub in $(cat subs.txt); do host "$sub" 2>/dev/null; done | grep "has address" | tee resolved.txt
dig any example.com
```

## Pièges et galères

{% hint style="warning" %}
Ne pas confondre reconnaissance passive et énumération active. Interroger crt.sh ou Shodan, c'est passif. Lancer un `dig` contre le serveur DNS de la cible, c'est déjà de l'énumération active. La distinction est importante pour respecter le périmètre d'un engagement.
{% endhint %}

- Les intégrations tierces découvertes dans les TXT records (Atlassian, LogMeIn, Google Workspace) sont des vecteurs d'attaque potentiels. Un mot de passe réutilisé sur une interface Jira ou Confluence peut compromettre toute l'infrastructure.
- Les sous-domaines découverts par Certificate Transparency incluent souvent des certificats wildcard (`*.example.com`). Ces entrées ne pointent pas vers un hôte réel, il faut les filtrer.

## Retour terrain

La reconnaissance de domaine est la phase qu'on néglige le plus par empressement. Pourtant, c'est souvent celle qui pose les bases des découvertes les plus critiques. Un enregistrement TXT qui mentionne un fournisseur VPN, un sous-domaine `staging` avec un certificat expiré, une IP Shodan qui expose un service d'administration non protégé : ces pistes n'apparaissent jamais dans un scan Nmap classique.

## Mémo express

| Action | Commande |
|---|---|
| Sous-domaines via CT | `curl -s "https://crt.sh/?q=example.com&output=json" \| jq -r '.[].name_value' \| sort -u` |
| Résolution DNS | `host sub.example.com` |
| Tous les records DNS | `dig any example.com` |
| Interrogation Shodan | `shodan host <IP>` |

***
