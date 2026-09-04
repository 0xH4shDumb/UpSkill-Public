# Lancer SQLMap

En pentest web, une fois qu'on suspecte un point d'injection SQL, il faut savoir alimenter SQLMap avec la bonne requete. Un parametre GET dans l'URL, un formulaire POST, un cookie, un corps JSON : chaque situation demande une syntaxe differente. Mal configurer la requete initiale, c'est perdre du temps sur des faux negatifs alors que la vulnerabilite est bien la.

## Pourquoi

SQLMap ne devine pas tout seul ou chercher. Il faut lui indiquer precisement la cible : l'URL, les parametres, les headers, le corps de la requete. Si la requete est mal construite (cookie manquant, Content-Type incorrect, parametre POST oublie), l'outil ne detectera rien, meme face a une injection triviale.

La majorite des echecs avec SQLMap viennent d'une mauvaise specification de la requete, pas d'une limitation de l'outil.

## Comment ca marche

### Le systeme d'aide

SQLMap propose deux niveaux d'aide en ligne de commande.

| Option | Description |
|--------|-------------|
| `-h` | Aide basique, suffisante pour la plupart des cas |
| `-hh` | Aide avancee, liste exhaustive de toutes les options |

```bash
# Aide basique
sqlmap -h

# Aide avancee
sqlmap -hh
```

La documentation complete est aussi disponible sur le wiki du projet GitHub.

### Principe general

SQLMap a besoin d'au moins un parametre a tester. Ce parametre peut etre fourni de plusieurs facons : dans l'URL (GET), dans le corps de la requete (POST), via un fichier de requete complet, ou en marquant manuellement le point d'injection.

Le flag `--batch` permet de lancer SQLMap sans interaction : il repond automatiquement aux questions avec les valeurs par defaut. Indispensable pour les scans automatises ou quand on veut laisser tourner sans surveillance.

## En pratique

### Requete GET simple

Le cas le plus courant : un parametre dans l'URL.

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --batch
```

SQLMap va tester le parametre `id` avec ses differentes techniques d'injection. Si le SGBD est detecte (par exemple MySQL), il propose d'adapter les payloads en consequence.

{% hint style="info" %}
L'option `-u` (ou `--url`) est la facon la plus directe de specifier une cible. Le parametre a tester doit avoir une valeur (meme arbitraire) pour que SQLMap puisse travailler.
{% endhint %}

### Requete POST

Pour tester des parametres POST, on utilise `--data` :

```bash
sqlmap -u "http://<IP_CIBLE>/login.php" --data "uid=1&name=test" --batch
```

Les deux parametres `uid` et `name` seront testes. Si on sait deja lequel est vulnerable, on peut cibler avec `-p` :

```bash
sqlmap -u "http://<IP_CIBLE>/login.php" --data "uid=1&name=test" -p uid --batch
```

### Marqueur d'injection personnalise

Le caractere `*` (asterisque) permet de designer precisement le point d'injection, quel que soit l'endroit dans la requete. SQLMap ne testera que cette position.

```bash
# Injection dans un parametre POST specifique
sqlmap -u "http://<IP_CIBLE>/login.php" --data "uid=1*&name=test" --batch
```

Ce marqueur fonctionne partout : dans l'URL, le corps POST, les cookies, les headers personnalises. C'est la methode la plus fiable quand on sait ou se trouve la vulnerabilite.

### Methode cURL depuis le navigateur

La facon la plus rapide et la plus fiable de construire une commande SQLMap a partir d'une requete reelle est de passer par les DevTools du navigateur.

1. Ouvrir les outils de developpement (F12)
2. Aller dans l'onglet Reseau (Network)
3. Effectuer l'action qui genere la requete
4. Clic droit sur la requete, puis **Copier > Copier comme cURL**
5. Coller dans le terminal et remplacer `curl` par `sqlmap`

```bash
# Exemple de commande obtenue apres remplacement de curl par sqlmap
sqlmap 'http://<IP_CIBLE>/page.php?id=1' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0' \
  -H 'Accept: text/html,application/xhtml+xml' \
  -H 'Accept-Language: fr-FR,fr;q=0.9' \
  --compressed -H 'Connection: keep-alive' --batch
```

{% hint style="success" %}
Cette methode garantit que tous les headers (cookies, tokens CSRF, User-Agent) sont correctement inclus. C'est la facon recommandee de travailler sur des applications avec authentification ou des mecanismes de protection.
{% endhint %}

### Fichier de requete complet (-r)

Pour les requetes complexes (beaucoup de headers, corps POST elabore, tokens specifiques), le plus propre est de sauvegarder la requete HTTP dans un fichier et de la passer a SQLMap avec `-r`.

On peut capturer la requete depuis Burp Suite (clic droit > "Copy to file") ou depuis les DevTools du navigateur (copier les headers de la requete).

```http
GET /page.php?id=1 HTTP/1.1
Host: cible.example.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml
Accept-Language: fr-FR,fr;q=0.9
Cookie: PHPSESSID=a8d127e4f3b5c6d7e8f9
Connection: close
```

```bash
# Lancer SQLMap avec le fichier de requete
sqlmap -r req.txt --batch
```

{% hint style="info" %}
On peut aussi utiliser le marqueur `*` dans le fichier de requete pour cibler un parametre precis, par exemple `/?id=*` dans la premiere ligne.
{% endhint %}

### Options de personnalisation

SQLMap offre de nombreuses options pour affiner la requete HTTP.

| Option | Description | Exemple |
|--------|-------------|---------|
| `--cookie` | Specifier un cookie de session | `--cookie="PHPSESSID=abc123"` |
| `-H` / `--header` | Ajouter un header HTTP personnalise | `-H "X-Forwarded-For: 127.0.0.1"` |
| `--random-agent` | Utiliser un User-Agent aleatoire (navigateur reel) | Evite la detection du UA par defaut de SQLMap |
| `--mobile` | Simuler un appareil mobile via le User-Agent | Utile si le backend sert du contenu different |
| `--method` | Forcer une methode HTTP specifique | `--method PUT` |
| `-p` | Cibler un parametre specifique | `-p id` |

```bash
# Exemple avec cookie et header personnalise
sqlmap -u "http://<IP_CIBLE>/api/data" \
  --cookie="session=abc123def456" \
  -H "X-Custom-Token: mytoken" \
  --random-agent --batch
```

{% hint style="warning" %}
Le User-Agent par defaut de SQLMap (`sqlmap/x.x.x`) est detecte par la quasi-totalite des WAF et filtres applicatifs. Utiliser `--random-agent` devrait etre un reflexe systematique en conditions reelles.
{% endhint %}

Le marqueur `*` fonctionne aussi dans les valeurs des options. Par exemple, pour tester une injection dans un cookie :

```bash
sqlmap -u "http://<IP_CIBLE>/page.php" --cookie="id=1*" --batch
```

SQLMap traitera alors le cookie comme point d'injection au lieu des parametres GET/POST.

### Corps JSON et XML

SQLMap gere nativement les corps de requete au format JSON et XML. Pas besoin de configuration particuliere : il detecte automatiquement le format et propose de traiter les parametres qu'il contient.

Avec `--data` pour un JSON simple :

```bash
sqlmap -u "http://<IP_CIBLE>/api/endpoint" \
  --data '{"id":1}' \
  -H "Content-Type: application/json" \
  --batch
```

Pour un corps JSON plus complexe ou avec des structures imbriquees, passer par un fichier de requete avec `-r` :

```http
POST /api/endpoint HTTP/1.1
Host: cible.example.com
Content-Type: application/json

{
  "data": [{
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "Exemple",
      "body": "Contenu de test"
    }
  }]
}
```

```bash
sqlmap -r req.txt --batch
```

SQLMap affiche un message de confirmation quand il detecte du JSON ou du XML dans le corps : `JSON data found in HTTP body. Do you want to process it?`. Avec `--batch`, la reponse est automatiquement oui.

{% hint style="info" %}
Le support JSON/XML est souple : SQLMap n'impose pas de contraintes strictes sur la structure. Les parametres detectes dans le corps sont testes comme n'importe quel autre parametre.
{% endhint %}

## Pieges et galeres

**Oublier le cookie de session.** Sur une application avec authentification, sans le bon cookie, toutes les requetes renvoient une redirection vers la page de login. SQLMap ne detecte rien. Toujours verifier que la session est active dans la requete.

**User-Agent par defaut.** Le UA de SQLMap est un signal d'alerte pour n'importe quel WAF. Une requete bloquee en amont ne sera jamais testee. `--random-agent` doit devenir un automatisme.

**Mauvais Content-Type.** Envoyer du JSON sans le header `Content-Type: application/json` peut provoquer un rejet cote serveur. SQLMap ne le rajoute pas automatiquement quand on utilise `--data`.

**Trop de parametres testes.** Sur une requete avec 15 parametres, SQLMap va tous les tester sequentiellement. Ca prend du temps. Si on sait ou chercher, cibler avec `-p` ou le marqueur `*` accelere considerablement le scan.

**Fichier de requete mal formate.** Lors de la copie depuis Burp, verifier qu'il n'y a pas de lignes vides parasites au debut du fichier. La premiere ligne doit etre la methode HTTP (`GET /... HTTP/1.1` ou `POST /... HTTP/1.1`).

## Retour terrain

En audit, la methode la plus fiable reste le combo Burp + fichier de requete. On intercepte la requete dans Burp, on la sauvegarde avec "Copy to file", et on la passe directement a SQLMap avec `-r`. Ca garantit que tous les parametres, cookies et headers sont fidelement reproduits. La methode cURL depuis les DevTools est un bon complement quand Burp n'est pas en place.

Sur des API modernes qui utilisent du JSON, ne pas hesiter a combiner `-r` avec le marqueur `*` pour cibler precisement le champ suspect dans une structure imbriquee.

Depuis un conteneur Exegol, SQLMap est pre-installe et pret a l'emploi. Il suffit de copier le fichier de requete dans le conteneur ou de travailler directement avec les commandes cURL.

## Memo express

| Besoin | Commande |
|--------|----------|
| Test GET basique | `sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --batch` |
| Test POST | `sqlmap -u "http://<IP_CIBLE>/login.php" --data "uid=1" --batch` |
| Depuis un fichier de requete | `sqlmap -r req.txt --batch` |
| Marqueur d'injection | Ajouter `*` apres la valeur a tester |
| Cookie de session | `--cookie="PHPSESSID=abc123"` |
| User-Agent aleatoire | `--random-agent` |
| Methode HTTP specifique | `--method PUT` |
| Header personnalise | `-H "X-Custom: value"` |
| Cibler un parametre | `-p id` |
| Aide complete | `sqlmap -hh` |

***
