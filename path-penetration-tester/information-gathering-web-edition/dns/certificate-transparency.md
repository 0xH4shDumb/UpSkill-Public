# Certificate Transparency

## Pourquoi

Les Certificate Transparency Logs (CT Logs) sont des registres publics dans lesquels chaque certificat SSL/TLS émis par une autorité de certification (CA) doit être enregistré. A l'origine, ce mécanisme a été conçu pour détecter les certificats frauduleux et responsabiliser les CA. En reconnaissance web, les CT Logs sont une source passive de découverte de sous-domaines souvent plus efficace que le brute-force, parce qu'ils fournissent des données réelles et officielles.

## Comment ça marche

### Le principe des CT Logs

Quand une CA émet un certificat SSL/TLS, elle doit l'enregistrer dans un ou plusieurs registres publics (les CT Logs). Ces registres sont des bases de données en mode **append-only** : on ne peut qu'ajouter des entrées, jamais en supprimer ni en modifier. Ils sont maintenus par des organisations indépendantes et consultables librement.

### Pourquoi c'est utile en reconnaissance

Les certificats contiennent souvent le champ **SAN (Subject Alternative Name)** qui liste tous les noms de domaine couverts par le certificat. Un seul certificat peut couvrir `exemple.com`, `www.exemple.com`, `api.exemple.com`, `staging.exemple.com`, etc. Les CT Logs permettent donc de :

- **Découvrir des sous-domaines** sans interagir avec la cible (reconnaissance passive pure).
- **Retrouver des sous-domaines historiques** : même les certificats expirés restent dans les logs.
- **Identifier des erreurs de configuration** : un certificat couvrant un sous-domaine de développement révèle une surface d'attaque potentielle.

{% hint style="success" %}
Contrairement au brute-force qui repose sur des suppositions (tester des mots d'une wordlist), les CT Logs fournissent une vue réelle de l'infrastructure, basée sur les certificats effectivement émis.
{% endhint %}

## En pratique

### crt.sh : recherche via le terminal

`crt.sh` est une interface web gratuite qui permet de chercher dans les CT Logs par nom de domaine. Elle expose aussi une API JSON exploitable en ligne de commande :

```bash
# - Extraire tous les sous-domaines d'un domaine via crt.sh
curl -s "https://crt.sh/?q=<DOMAINE_CIBLE>&output=json" \
  | jq -r '.[].name_value' \
  | sort -u
```

Pour filtrer sur un mot-clé spécifique (par exemple, les sous-domaines contenant "dev") :

```bash
curl -s "https://crt.sh/?q=<DOMAINE_CIBLE>&output=json" \
  | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' \
  | sort -u
```

**Explication de la commande :**

- `curl -s` : interroge l'API JSON de crt.sh silencieusement.
- `jq -r` : filtre et extrait les valeurs du champ `name_value`.
- `select(... | contains("dev"))` : ne garde que les entrées contenant "dev".
- `sort -u` : trie et déduplique les résultats.

### Censys : recherche avancée

Censys est un moteur de recherche plus puissant qui permet de filtrer par domaine, IP ou certificat. Il offre des capacités de recherche avancées et une API, mais nécessite une inscription.

### Outils de CT Logs

| Outil | Points forts | Limites |
|---|---|---|
| `crt.sh` | Gratuit, sans inscription, API JSON, simple | Peu de filtres avancés |
| `Censys` | Recherches puissantes, filtres par IP/certificat, API | Inscription requise |

## Pièges et galères

- **Volume de résultats** : les domaines populaires peuvent avoir des milliers d'entrées dans les CT Logs. Le tri et la déduplication sont indispensables.
- **Wildcards** : les certificats avec `*.exemple.com` apparaissent dans les résultats mais ne révèlent pas les sous-domaines spécifiques.
- **Données historiques** : les CT Logs contiennent aussi des certificats expirés. Un sous-domaine trouvé dans un ancien certificat peut ne plus exister.

## Retour terrain

Les CT Logs sont un complément idéal au brute-force DNS. En combinant les deux approches, on maximise la couverture : les CT Logs révèlent les sous-domaines réels pour lesquels un certificat a été émis, tandis que le brute-force peut trouver des sous-domaines qui n'ont jamais eu de certificat. C'est une technique 100% passive qui ne génère aucun trafic vers la cible.

## Mémo express

| Besoin | Commande |
|---|---|
| Lister les sous-domaines (crt.sh) | `curl -s "https://crt.sh/?q=<domaine>&output=json" \| jq -r '.[].name_value' \| sort -u` |
| Filtrer par mot-clé | Ajouter `\| select(.name_value \| contains("<mot>"))` dans le pipeline jq |

***
