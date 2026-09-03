# Well-Known URIs

## Pourquoi

Le standard `.well-known` (RFC 8615) définit un emplacement conventionnel sur les serveurs web pour centraliser les métadonnées critiques d'un site. Accessible via le chemin `/.well-known/`, ce répertoire contient des fichiers de configuration standardisés qui décrivent les services, protocoles et mécanismes de sécurité du site. En reconnaissance web, ces fichiers sont une source d'information structurée, souvent négligée, qui peut révéler des endpoints, des configurations et des détails d'infrastructure.

## Comment ça marche

L'IANA (Internet Assigned Numbers Authority) maintient un registre de tous les suffixes `.well-known` reconnus. Chaque URI a un rôle précis, défini par une spécification ou un standard.

### URIs notables

| URI | Rôle | Standard |
|---|---|---|
| `security.txt` | Coordonnées de contact pour signaler des vulnérabilités | RFC 9116 |
| `change-password` | URL standard pour la page de changement de mot de passe | W3C |
| `openid-configuration` | Configuration OpenID Connect (endpoints OAuth2, clés, scopes) | OpenID Connect Discovery |
| `assetlinks.json` | Vérification de propriété d'actifs numériques (apps mobiles) | Google Digital Asset Links |
| `mta-sts.txt` | Politique MTA Strict Transport Security pour sécuriser les emails | RFC 8461 |

Ce n'est qu'un échantillon. Le registre IANA complet contient des dizaines d'entrées, chacune potentiellement intéressante selon le contexte.

## En pratique

### security.txt

Le fichier `security.txt` est le plus courant. Il fournit les coordonnées pour signaler des vulnérabilités de manière responsable :

```bash
curl -s https://<DOMAINE_CIBLE>/.well-known/security.txt
```

Il peut contenir des emails de contact, des clés PGP, des politiques de divulgation et des URLs de programmes de bug bounty.

### openid-configuration

Ce fichier est particulièrement riche en informations. Quand un site utilise OpenID Connect pour l'authentification, ce endpoint expose la configuration complète du fournisseur d'identité :

```bash
curl -s https://<DOMAINE_CIBLE>/.well-known/openid-configuration | jq .
```

Exemple de réponse :

```json
{
  "issuer": "https://exemple.com",
  "authorization_endpoint": "https://exemple.com/oauth2/authorize",
  "token_endpoint": "https://exemple.com/oauth2/token",
  "userinfo_endpoint": "https://exemple.com/oauth2/userinfo",
  "jwks_uri": "https://exemple.com/oauth2/jwks",
  "response_types_supported": ["code", "token", "id_token"],
  "scopes_supported": ["openid", "profile", "email"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

Ce fichier révèle :

- **Les endpoints d'authentification** : URLs d'autorisation, de tokens, d'informations utilisateur.
- **Le JWKS URI** : emplacement des clés cryptographiques utilisées pour signer les tokens JWT.
- **Les scopes et types de réponses** : quelles données sont accessibles et comment.
- **Les algorithmes de signature** : indique les mesures cryptographiques en place.

{% hint style="success" %}
L'`openid-configuration` est une cible de choix pour la reconnaissance. Il cartographie l'infrastructure d'authentification et peut révéler des endpoints à tester pour des failles d'autorisation.
{% endhint %}

### Explorer systématiquement

Le registre IANA est accessible en ligne. En pentest, on peut tester les URIs les plus courants :

```bash
# - Tester plusieurs URIs well-known
for uri in security.txt openid-configuration assetlinks.json mta-sts.txt robots.txt; do
  echo "--- $uri ---"
  curl -s -o /dev/null -w "%{http_code}" "https://<DOMAINE_CIBLE>/.well-known/$uri"
  echo
done
```

## Pièges et galères

- **URIs absents** : la plupart des sites n'implémentent qu'une poignée d'URIs `.well-known`. Un code 404 est la réponse la plus fréquente.
- **Informations partielles** : les fichiers présents ne contiennent pas toujours toutes les informations attendues par la spécification.
- **Fausse piste** : un `security.txt` peut être générique ou obsolète. Toujours vérifier la cohérence des informations trouvées avec d'autres sources.

## Mémo express

| Besoin | Commande |
|---|---|
| security.txt | `curl -s https://<cible>/.well-known/security.txt` |
| Config OpenID | `curl -s https://<cible>/.well-known/openid-configuration \| jq .` |
| Liens d'assets Android | `curl -s https://<cible>/.well-known/assetlinks.json \| jq .` |

***
