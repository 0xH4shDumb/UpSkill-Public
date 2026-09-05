# HTTP Verb Tampering

Le protocole HTTP ne se limite pas à `GET` et `POST`. Une application mal conçue, ou un serveur mal configuré, peut laisser passer d'autres verbes sans appliquer les mêmes contrôles d'authentification ou de filtrage. Le HTTP Verb Tampering consiste justement à exploiter cette différence de traitement entre les méthodes, pour contourner une authentification ou un filtre de sécurité qui ne couvre qu'une partie des cas.

## Pourquoi

La plupart des développeurs raisonnent avec deux verbes en tête : `GET` pour récupérer une ressource, `POST` pour en envoyer une. Le reste de la spécification HTTP (`HEAD`, `PUT`, `DELETE`, `OPTIONS`, `PATCH`...) est souvent traité comme un détail d'implémentation, voire ignoré complètement. Le problème, c'est que ce sont rarement les développeurs qui décident quels verbes un client peut envoyer : c'est le serveur web qui répond, quel que soit le verbe demandé, tant que rien ne l'en empêche explicitement.

Quand une règle de sécurité, que ce soit une authentification côté serveur ou un filtre applicatif, n'est écrite que pour un sous-ensemble des verbes acceptés, il suffit de changer de verbe pour sortir du périmètre de cette règle. C'est une classe de vulnérabilité qui touche aussi bien la configuration du serveur web que le code de l'application, ce qui la rend particulièrement transverse : elle peut se cacher derrière une authentification qu'on croyait solide, ou derrière un filtre anti-injection qu'on a soigneusement testé... avec un seul verbe.

{% hint style="info" %}
On parle de "tampering" (falsification) parce que l'attaque ne modifie ni les paramètres ni le corps de la requête : elle se contente de changer le verbe HTTP utilisé pour envoyer une requête par ailleurs identique.
{% endhint %}

## Comment ça marche

### Les verbes HTTP

La spécification HTTP définit neuf méthodes. En dehors de `GET` et `POST`, il faut connaître au minimum les suivantes :

| Verbe | Description |
|---|---|
| `HEAD` | Identique à une requête `GET`, mais la réponse ne contient que les en-têtes, sans le corps |
| `PUT` | Écrit le contenu de la requête à l'emplacement indiqué |
| `DELETE` | Supprime la ressource à l'emplacement indiqué |
| `OPTIONS` | Renvoie les options acceptées par le serveur pour une ressource donnée, notamment les verbes autorisés |
| `PATCH` | Applique une modification partielle à la ressource indiquée |

{% hint style="warning" %}
Certains de ces verbes ont un impact direct et sensible : `PUT` permet potentiellement d'écrire un fichier sur le serveur, `DELETE` d'en supprimer un. Un serveur mal configuré qui accepte ces méthodes sans contrôle peut donner un accès en écriture au webroot sans passer par une quelconque application.
{% endhint %}

### Deux origines distinctes

Le HTTP Verb Tampering ne provient pas d'une seule cause. Deux mécanismes complètement différents peuvent y mener, et il vaut mieux les distinguer clairement parce qu'ils ne se corrigent pas de la même façon.

**Configuration serveur insécurisée.** C'est le cas le plus direct : une règle d'authentification au niveau du serveur web (Apache, Tomcat, IIS/ASP.NET...) ne liste qu'un ou deux verbes, et laisse tous les autres passer librement. Un exemple typique en Apache :

```xml
<Limit GET POST>
    Require valid-user
</Limit>
```

Cette directive protège les requêtes `GET` et `POST`, mais laisse `HEAD`, `OPTIONS`, `PUT` ou n'importe quel autre verbe totalement libre d'accès sur la même ressource. Le serveur ne fait pas d'exception implicite : si un verbe n'est pas listé dans le `<Limit>`, aucune règle ne s'applique à lui.

**Code applicatif insécurisé.** Ici le problème n'est pas dans la configuration du serveur, mais dans une incohérence de code. Un développeur écrit un filtre de sécurité qui ne vérifie que les paramètres `$_POST`, mais la fonction qui traite réellement la donnée utilise `$_REQUEST`, qui agrège `$_GET`, `$_POST` et les cookies :

```php
$pattern = "/^[A-Za-z\s]+$/";

if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
    // ...
}
```

Le filtre teste `$_GET["code"]`. Si on envoie notre payload malveillant en paramètre `POST`, la variable `$_GET["code"]` est vide, donc le `preg_match` réussit trivialement (une chaîne vide correspond au motif). La requête SQL, elle, utilise `$_REQUEST["code"]`, qui contient bien notre payload `POST`. Le filtre est passé sans avoir rien filtré du tout.

{% hint style="danger" %}
Cette seconde catégorie est nettement plus fréquente en conditions réelles. Une mauvaise configuration serveur se voit et se corrige facilement une fois identifiée ; une incohérence entre deux fonctions de récupération de paramètres, dispersée dans des milliers de lignes de code, peut survivre des années en production.
{% endhint %}

## En pratique

Les deux scénarios suivants reproduisent une application "File Manager" volontairement vulnérable, exécutée depuis un environnement Exegol. Elle permet de créer des fichiers par leur nom et propose un bouton de réinitialisation protégé par une authentification HTTP Basic.

### Contourner une authentification HTTP Basic

Le bouton "Reset" de l'application déclenche un appel vers `/admin/reset.php`. En cliquant dessus sans être authentifié, on reçoit une invite HTTP Basic Auth suivie, en l'absence d'identifiants, d'une erreur `401 Unauthorized`. La première étape consiste à déterminer le périmètre exact de cette protection : en visitant directement `/admin/`, on obtient la même invite, ce qui confirme que c'est tout le répertoire qui est protégé, et pas seulement la page `reset.php`.

En interceptant la requête avec un proxy, on voit qu'elle part en `GET`. On teste d'abord un changement vers `POST` pour vérifier si l'authentification couvre les deux verbes usuels : c'est le cas, la protection s'applique aussi bien à `GET` qu'à `POST`. Il faut donc chercher ailleurs.

L'étape suivante consiste à demander directement au serveur quels verbes il accepte, via une requête `OPTIONS` :

```bash
curl -i -X OPTIONS http://<IP_CIBLE>:<PORT>/
```

```http
HTTP/1.1 200 OK
Date: ...
Server: Apache/2.4.41 (Ubuntu)
Allow: POST,OPTIONS,HEAD,GET
Content-Length: 0
Content-Type: httpd/unix-directory
```

L'en-tête `Allow` révèle que le serveur accepte aussi `HEAD`, en plus de `OPTIONS`. C'est le comportement par défaut de nombreux serveurs web, souvent laissé actif sans réflexion particulière. `HEAD` se comporte exactement comme `GET` côté traitement serveur, à une différence près : la réponse ne contient jamais de corps. Si l'authentification n'a été écrite que pour `GET` et `POST`, envoyer une requête `HEAD` vers `/admin/reset.php` va déclencher l'exécution du code de réinitialisation, sans jamais demander la moindre authentification :

```bash
curl -i -X HEAD http://<IP_CIBLE>:<PORT>/admin/reset.php
```

Aucune invite de connexion, aucun `401`, juste une réponse vide comme attendu pour une requête `HEAD`. En rechargeant l'application, on constate que la fonction de réinitialisation s'est bien exécutée : tous les fichiers ont été supprimés, sans qu'aucun identifiant n'ait été fourni.

{% hint style="success" %}
Le réflexe à avoir face à toute page protégée par une authentification : tester systématiquement `OPTIONS` pour lister les verbes acceptés par le serveur, puis essayer chacun d'eux (en particulier `HEAD`) sur la ressource protégée avant de conclure que l'authentification est solide.
{% endhint %}

### Contourner un filtre de sécurité

Le même File Manager applique un filtre sur le nom des fichiers créés. Une tentative de création avec des caractères spéciaux (par exemple `test;`) déclenche un message "Malicious Request Denied!" : à première vue, l'application est correctement protégée contre l'injection de commandes.

En interceptant la requête `POST` correspondante et en la rejouant en `GET` (avec les mêmes paramètres, simplement déplacés dans l'URL), le message d'erreur disparaît et le fichier est créé sans encombre. Le filtre ne s'applique donc qu'à une partie des méthodes.

Pour confirmer qu'il s'agit bien d'un contournement exploitable, et pas d'un simple artefact, on va au bout de la démarche en injectant une vraie commande. Avec un nom de fichier de la forme `file1; touch file2;`, envoyé en `GET` :

```bash
curl -s "http://<IP_CIBLE>:<PORT>/index.php?filename=file1%3B+touch+file2%3B"
```

Les deux fichiers, `file1` et `file2`, apparaissent dans le gestionnaire. Le filtre censé bloquer les caractères spéciaux ne s'est jamais déclenché, alors que la commande shell générée en arrière-plan par l'application s'est bien exécutée avec le point-virgule intact. Le changement de verbe a suffi à faire passer un payload de Command Injection que l'application semblait pourtant bloquer efficacement en conditions normales.

{% hint style="warning" %}
Un filtre de sécurité qui semble efficace en `POST` (ou en `GET`) ne dit rien de son comportement sur l'autre méthode. Toujours rejouer un test de filtrage avec au minimum les deux verbes usuels avant de conclure qu'une entrée est correctement validée.
{% endhint %}

## Prévention

### Configuration serveur

La faille de configuration se corrige en évitant de restreindre une règle d'authentification ou d'autorisation à un verbe précis. Chaque serveur propose un mécanisme dédié pour appliquer une règle à tous les verbes sauf ceux explicitement exclus, plutôt que l'inverse.

{% tabs %}
{% tab title="Apache" %}
Configuration vulnérable, qui ne protège que `GET` :

```xml
<Directory "/var/www/html/admin">
    AuthType Basic
    AuthName "Admin Panel"
    AuthUserFile /etc/apache2/.htpasswd
    <Limit GET>
        Require valid-user
    </Limit>
</Directory>
```

Correction, avec `LimitExcept` qui couvre tous les verbes sauf ceux listés :

```xml
<Directory "/var/www/html/admin">
    AuthType Basic
    AuthName "Admin Panel"
    AuthUserFile /etc/apache2/.htpasswd
    <LimitExcept GET POST>
        Require valid-user
    </LimitExcept>
</Directory>
```

Même en listant `GET` et `POST` dans un `<Limit>`, `HEAD` et `OPTIONS` restent ouverts. `LimitExcept` inverse la logique : tout ce qui n'est pas explicitement exclu reste soumis à la règle.
{% endtab %}

{% tab title="Tomcat" %}
Configuration vulnérable dans `web.xml`, qui ne protège que `GET` :

```xml
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern>
        <http-method>GET</http-method>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>
```

Correction, avec `http-method-omission` qui applique la contrainte à tous les verbes sauf ceux listés :

```xml
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern>
        <http-method-omission>GET</http-method-omission>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>
```
{% endtab %}

{% tab title="ASP.NET" %}
Configuration vulnérable dans `web.config`, qui ne restreint que `GET` :

```xml
<system.web>
    <authorization>
        <allow verbs="GET" roles="admin">
            <deny verbs="GET" users="*">
        </deny>
        </allow>
    </authorization>
</system.web>
```

Correction, en listant explicitement chaque verbe autorisé ou refusé avec `add`/`remove` plutôt qu'un unique attribut `verbs` :

```xml
<system.web>
    <authorization>
        <deny verbs="GET,POST,PUT,DELETE" users="*" />
        <allow verbs="GET,POST,PUT,DELETE" roles="admin" />
    </authorization>
</system.web>
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
Sauf besoin fonctionnel avéré, il vaut mieux désactiver purement et simplement les requêtes `HEAD` au niveau du serveur. C'est le verbe qui revient le plus souvent dans les contournements d'authentification, précisément parce qu'il est activé par défaut et rarement pris en compte dans les règles écrites pour `GET`/`POST`.
{% endhint %}

### Code applicatif

La correction côté code est plus délicate à généraliser, parce qu'elle ne dépend pas d'une directive de configuration mais de la cohérence du code applicatif dans son ensemble. Reprenons l'exemple du File Manager, avec sa vulnérabilité de Command Injection :

```php
if (isset($_REQUEST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . $_REQUEST['filename']);
    } else {
        echo "Malicious Request Denied!";
    }
}
```

Isolé, le `preg_match` a l'air parfaitement fonctionnel : il rejette bien tout caractère hors de la liste blanche. Le problème ne se voit qu'en comparant les deux lignes suivantes : le filtre lit `$_POST['filename']`, alors que la commande exécutée utilise `$_REQUEST['filename']`. Une requête envoyée en `GET` laisse `$_POST['filename']` vide, donc filtrée avec succès (rien à filtrer), pendant que `$_REQUEST['filename']` récupère malgré tout le paramètre `GET` malveillant au moment de construire la commande.

La correction consiste à utiliser systématiquement la même source de paramètres dans le filtre et dans le traitement, idéalement en élargissant le filtre à toutes les méthodes possibles plutôt qu'en le restreignant à une seule :

| Langage | Fonction/objet à privilégier |
|---|---|
| PHP | `$_REQUEST['param']` |
| Java | `request.getParameter('param')` |
| C# | `Request['param']` |

{% hint style="info" %}
Ce n'est pas tant "utiliser `$_REQUEST`" qui compte que la cohérence : le filtre et la fonction métier doivent lire le paramètre exactement de la même façon. Le risque n'est jamais dans un choix de superglobale en particulier, mais dans l'écart entre deux fonctions censées parler du même paramètre.
{% endhint %}

## Pièges et galères

- **S'arrêter à `GET`/`POST`.** Un test de filtrage qui ne couvre que ces deux verbes passe à côté de la faille la plus fréquente. Toujours tester les deux à minima, et idéalement `HEAD`/`OPTIONS`/`PUT`/`DELETE` sur les points d'authentification sensibles.
- **Confondre les deux causes.** Une authentification contournée par changement de verbe se corrige au niveau de la configuration serveur ; un filtre applicatif contourné se corrige dans le code. Appliquer la mauvaise correction (par exemple durcir `LimitExcept` alors que le problème est une incohérence `$_GET`/`$_POST` dans le code) ne règle rien.
- **Sous-estimer `HEAD`.** Beaucoup de testeurs changent `GET` en `POST` par réflexe, et s'arrêtent là si l'authentification tient. `HEAD` reste le vecteur le plus fiable parce qu'il est activé par défaut sur la majorité des serveurs web sans que personne n'y ait jamais pensé.
- **Ne pas vérifier l'exploitation complète.** Bypasser un message d'erreur ("Malicious Request Denied!") ne prouve rien en soi : il faut aller jusqu'au bout et démontrer un impact réel (ici, la création d'un second fichier via l'injection) pour confirmer que le contournement a un effet exploitable, et pas seulement cosmétique.
- **Oublier `OPTIONS` comme outil de reconnaissance.** Une requête `OPTIONS` coûte une ligne de `curl` et révèle directement les verbes acceptés par le serveur pour une ressource donnée, ce qui évite de deviner à l'aveugle.

## Retour terrain

En audit, le HTTP Verb Tampering se cherche systématiquement dès qu'une page d'administration ou une fonctionnalité sensible est protégée par une authentification qu'on n'a pas réussi à contourner autrement. Le réflexe est de toujours passer une requête `OPTIONS` sur la ressource visée avant de conclure qu'elle est correctement protégée : c'est un test rapide, silencieux, qui ne coûte rien et qui révèle immédiatement si `HEAD` ou un autre verbe inattendu est ouvert.

Côté filtres applicatifs, cette classe de vulnérabilité se rencontre surtout sur des développements internes, moins sur des frameworks matures qui centralisent la récupération des paramètres. Dès qu'un filtre de sécurité (anti-injection, anti-XSS, contrôle de format) semble efficace, il vaut la peine de rejouer exactement la même requête avec le verbe inverse avant de le valider. C'est un test qui prend quelques secondes et qui peut faire la différence entre "filtre robuste" et "filtre contournable en changeant une seule ligne dans Burp".

Dans un rapport, ce type de faille se documente en distinguant clairement les deux causes possibles (configuration serveur vs code applicatif), parce que la remédiation proposée au client n'est pas la même : modifier un fichier de configuration Apache/Tomcat/IIS d'un côté, revoir la cohérence du code applicatif de l'autre. Un rapport qui recommande "corriger la configuration Apache" pour une faille en réalité due à une incohérence `$_GET`/`$_REQUEST` fait perdre du temps à l'équipe de développement qui devra rechercher elle-même la vraie cause.

## Mémo express

| Élément | Détail |
|---|---|
| Deux causes possibles | Configuration serveur insécurisée / code applicatif incohérent |
| Verbe le plus utile en reconnaissance | `OPTIONS` (liste les méthodes acceptées via l'en-tête `Allow`) |
| Verbe le plus fiable pour bypasser une auth | `HEAD` (identique à `GET`, activé par défaut, souvent oublié dans les règles) |
| Test de filtre | Rejouer la même requête en changeant `GET` ↔ `POST` |
| Fix Apache | `LimitExcept` au lieu de `Limit` |
| Fix Tomcat | `http-method-omission` au lieu de `http-method` |
| Fix ASP.NET | `add`/`remove` explicites au lieu d'un simple attribut `verbs` |
| Fix code | Même source de paramètres (`$_REQUEST`, `request.getParameter`, `Request[]`) dans le filtre et dans le traitement |
| Bonne pratique générale | Désactiver `HEAD` si l'application n'en a pas l'usage |

***
