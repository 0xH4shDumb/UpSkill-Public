# Ressources cloud

La plupart des entreprises s'appuient sur des services cloud (AWS, Azure, GCP) pour héberger une partie de leur infrastructure. Les fournisseurs cloud sécurisent leur propre plateforme, mais la configuration reste la responsabilité du client. Et c'est là que les erreurs s'accumulent.

## Pourquoi

Les stockages cloud mal configurés sont parmi les vecteurs de fuite de données les plus courants. Un bucket S3 public, un blob Azure accessible sans authentification, un dépôt GCS en lecture ouverte : ces erreurs exposent directement des fichiers qui n'auraient jamais dû être accessibles. Sauvegardes de bases de données, fichiers de configuration, clés SSH, tokens d'API, documents internes.

Identifier ces ressources pendant la phase de reconnaissance permet de mettre en évidence des failles critiques sans même avoir besoin de scanner l'infrastructure réseau.

## Comment ça marche

### Détection via la résolution DNS

En résolvant les sous-domaines identifiés lors de la collecte de domaine, certaines entrées pointent directement vers des services cloud. Un CNAME vers `s3-website-us-west-2.amazonaws.com` ou un enregistrement A vers une IP Azure trahit immédiatement la présence de stockage cloud.

```bash
# depuis Exegol - identifier les sous-domaines liés au cloud
for sub in $(cat subdomains.txt); do
  host "$sub" | grep -E "amazonaws|blob\.core|storage\.googleapis"
done
```

### Google Dorks

Les moteurs de recherche indexent parfois le contenu des stockages cloud publics. Des requêtes ciblées permettent de trouver des fichiers exposés sans interagir avec l'infrastructure de la cible.

{% tabs %}
{% tab title="AWS S3" %}
```
intext:"nom-de-l-entreprise" inurl:amazonaws.com
```
{% endtab %}
{% tab title="Azure Blob" %}
```
intext:"nom-de-l-entreprise" inurl:blob.core.windows.net
```
{% endtab %}
{% tab title="GCP Storage" %}
```
intext:"nom-de-l-entreprise" inurl:storage.googleapis.com
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Le code source HTML des sites de la cible est aussi une piste. Les fichiers JavaScript, CSS et les images sont souvent servis depuis un bucket cloud. Inspecter les attributs `src` et `href` dans le DOM peut révéler des stockages cloud non documentés.
{% endhint %}

### Outils spécialisés

| Outil | Usage |
|---|---|
| GrayHatWarfare | Recherche de buckets S3, blobs Azure et stockages GCP publics. Filtrage par type de fichier. |
| domain.glass | Informations de domaine, services de sécurité détectés (Cloudflare, etc.), liens vers les réseaux sociaux. |
| cloud\_enum | Enumération automatisée de ressources cloud à partir d'un nom d'entreprise. |

## En pratique

La recherche de ressources cloud s'intègre naturellement dans la collecte passive :

1. **Résolution des sous-domaines** : repérer les CNAME et IPs qui pointent vers des services cloud
2. **Google Dorks** : rechercher des fichiers indexés sur les domaines cloud
3. **Code source** : inspecter les URLs dans le HTML/JS des sites de la cible
4. **Outils spécialisés** : lancer GrayHatWarfare ou cloud\_enum pour une recherche automatisée

```bash
# depuis Exegol - enumération cloud avec cloud_enum
python3 cloud_enum.py -k "nom-entreprise" -k "nom-domaine"
```

## Pièges et galères

{% hint style="warning" %}
Un bucket S3 public ne signifie pas forcément qu'il appartient à la cible. Les noms de buckets sont globaux sur AWS, et n'importe qui peut en créer un avec un nom ressemblant. Toujours vérifier que le contenu est bien lié à l'entreprise avant de l'inclure dans le rapport.
{% endhint %}

- Les fichiers les plus sensibles dans un bucket ouvert ne sont pas toujours les plus évidents. Un `config.json`, un `.env`, un `id_rsa` ou un `db_backup.sql` valent bien plus qu'un PDF marketing.
- Certains stockages cloud sont configurés en "list denied, read allowed" : on ne peut pas lister le contenu, mais si on connait le nom d'un fichier, on peut le télécharger. Les wordlists de noms de fichiers courants sont utiles dans ce cas.

## Retour terrain

Les fuites via stockage cloud sont un classique des audits de sécurité. On trouve régulièrement des sauvegardes de bases de données, des exports de credentials, ou des fichiers de configuration avec des tokens d'API en clair. Le plus surprenant, c'est que ces expositions persistent souvent pendant des mois avant d'être détectées, parce que personne ne surveille les permissions des buckets.

Les développeurs sont aussi une source indirecte : des dépôts GitHub ou des pastebins peuvent contenir des URLs de stockage cloud avec des tokens d'accès temporaires ou des clés d'API intégrées dans le code.

***
