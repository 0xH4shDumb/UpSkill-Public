# Énumération de sous-domaines

## Pourquoi

Sous la surface d'un domaine principal se cache souvent tout un réseau de sous-domaines. Chacun peut héberger un service différent : `blog.exemple.com` pour le blog, `api.exemple.com` pour l'API, `staging.exemple.com` pour l'environnement de test. Pour un pentester, ces sous-domaines représentent autant de points d'entrée potentiels, parfois bien moins protégés que le site principal.

### Ce qu'on peut y trouver

- **Environnements de développement et de staging** : souvent configurés avec des mesures de sécurité relâchées, voire avec des credentials par défaut.
- **Portails d'administration cachés** : interfaces d'admin non référencées publiquement mais accessibles si on connaît l'URL.
- **Applications legacy** : d'anciennes applications oubliées sur des sous-domaines, tournant sur des versions logicielles vulnérables.
- **Informations sensibles** : documents internes, fichiers de configuration, données exposées par inadvertance.

## Comment ça marche

L'énumération de sous-domaines consiste à identifier systématiquement tous les sous-domaines associés à un domaine cible. Du point de vue DNS, les sous-domaines sont généralement représentés par des enregistrements `A` (ou `AAAA`) qui associent le nom à une adresse IP, ou par des enregistrements `CNAME` qui créent des alias.

Deux approches complémentaires existent :

### Énumération active

On interagit directement avec les serveurs DNS de la cible. La technique la plus courante est le **brute-force** : on teste systématiquement une liste de noms potentiels en les préfixant au domaine cible.

**Méthodologie en 4 étapes :**

1. **Sélection de la wordlist** : générique (`admin`, `dev`, `mail`, `test`), ciblée par industrie, ou personnalisée à partir d'informations collectées en amont.
2. **Génération des noms** : chaque mot de la liste est préfixé au domaine (`dev.cible.com`, `staging.cible.com`).
3. **Résolution DNS** : chaque nom généré est soumis à une requête DNS pour vérifier s'il résout vers une IP.
4. **Filtrage et validation** : les sous-domaines qui résolvent sont conservés pour une analyse plus poussée (scan de ports, navigation, etc.).

### Énumération passive

On s'appuie sur des sources externes sans jamais interroger directement les serveurs de la cible :

- **Certificate Transparency Logs** : les certificats SSL/TLS publiés dans les CT logs listent souvent les sous-domaines dans le champ SAN (Subject Alternative Name).
- **Moteurs de recherche** : les opérateurs comme `site:` permettent de filtrer les résultats pour ne voir que les sous-domaines indexés.
- **Bases de données en ligne** : des services comme crt.sh, SecurityTrails ou VirusTotal agrègent des données DNS provenant de multiples sources.

{% hint style="info" %}
L'approche active offre un meilleur contrôle et une découverte potentiellement plus complète, mais elle est plus détectable. L'approche passive est plus discrète mais ne révèle pas tous les sous-domaines existants. En pratique, on combine les deux.
{% endhint %}

## En pratique

### Brute-force avec dnsenum

`dnsenum` est un outil Perl complet qui combine plusieurs techniques : récupération des enregistrements DNS, tentatives de transfert de zone, brute-force par wordlist et résolution inverse.

```bash
dnsenum --enum <DOMAINE_CIBLE> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
```

| Option | Rôle |
|---|---|
| `--enum` | Active les fonctions d'énumération automatique |
| `-f` | Spécifie la wordlist à utiliser |
| `-r` | Active la récursivité (si `dev.cible.com` est trouvé, tente `sub.dev.cible.com`) |

#### Exemple de sortie

```
[...] Brute forcing with subdomains-top1million-20000.txt:

www.cible.com.       A   10.10.10.1
support.cible.com.   A   10.10.10.1
mail.cible.com.      A   10.10.10.1
```

### Autres outils de brute-force

| Outil | Particularité |
|---|---|
| `fierce` | Bonne détection des wildcards DNS, interface intuitive |
| `dnsrecon` | Multi-techniques, export JSON/CSV |
| `amass` | Puissant pour les campagnes à grande échelle, intègre des sources OSINT |
| `assetfinder` | Rapide, orienté reconnaissance passive |
| `puredns` | Très rapide, utile pour nettoyer les faux positifs |

## Pièges et galères

- **Wildcard DNS** : certains domaines répondent positivement à n'importe quel sous-domaine (wildcard `*.cible.com`). Sans filtrage, le brute-force renvoie des milliers de faux positifs. Utiliser `puredns` ou les options de filtrage des outils pour détecter et exclure les wildcards.
- **Autorisation préalable** : le brute-force DNS génère un volume de requêtes important. Ne jamais lancer ce type de scan sans autorisation explicite.
- **Wordlists inadaptées** : une wordlist générique ne trouvera pas les sous-domaines spécifiques à une organisation. Compléter avec des mots-clés issus de la reconnaissance initiale (noms de produits, technologies utilisées, conventions de nommage observées).

## Mémo express

| Besoin | Commande |
|---|---|
| Brute-force basique | `dnsenum --enum <domaine> -f <wordlist>` |
| Brute-force récursif | `dnsenum --enum <domaine> -f <wordlist> -r` |
| Découverte passive (crt.sh) | `curl -s "https://crt.sh/?q=<domaine>&output=json" \| jq -r '.[].name_value' \| sort -u` |

***
