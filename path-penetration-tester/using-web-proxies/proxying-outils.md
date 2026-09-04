# Proxying d'outils CLI

## Pourquoi

Les proxys web ne servent pas uniquement au navigateur. En pentest, de nombreux outils en ligne de commande envoient des requetes HTTP qu'on a besoin d'inspecter ou de modifier. Router le trafic de ces outils a travers Burp ou ZAP permet de voir exactement ce qu'ils envoient et de debugger les cas ou un module ou un script ne fonctionne pas comme prevu.

## Comment ca marche

Chaque outil a sa propre methode pour configurer un proxy. La plupart acceptent une variable d'environnement ou une option de ligne de commande. Pour les outils qui ne supportent pas nativement la configuration de proxy, `proxychains` permet de forcer le routage du trafic.

## En pratique

### Proxychains

Proxychains redirige le trafic de n'importe quel outil CLI a travers un proxy, sans modification de l'outil.

**Configuration :**

Editer `/etc/proxychains.conf` et modifier la derniere ligne :

```bash
# Commenter la ligne SOCKS par defaut
#socks4         127.0.0.1 9050

# Ajouter le proxy HTTP
http 127.0.0.1 8080
```

**Utilisation :**

```bash
# -q pour le mode silencieux (pas de logs proxychains)
proxychains -q curl http://<IP_CIBLE>

proxychains -q nmap -sT -Pn <IP_CIBLE>

proxychains -q sqlmap -u "http://<IP_CIBLE>/page?id=1"
```

La requete apparait dans l'historique du proxy et peut etre inspectee ou rejouee.

{% hint style="warning" %}
Proxychains ralentit les outils. Ne l'utiliser que pour debugger ou inspecter des requetes specifiques, pas pour une utilisation courante.
{% endhint %}

### Metasploit

Metasploit supporte nativement la configuration d'un proxy via l'option `PROXIES` :

```bash
msf6 > use auxiliary/scanner/http/robots_txt
msf6 auxiliary(scanner/http/robots_txt) > set PROXIES HTTP:127.0.0.1:8080
msf6 auxiliary(scanner/http/robots_txt) > set RHOSTS <IP_CIBLE>
msf6 auxiliary(scanner/http/robots_txt) > set RPORT 80
msf6 auxiliary(scanner/http/robots_txt) > run
```

Les requetes envoyees par le module Metasploit sont visibles dans l'historique du proxy. Fonctionne avec tous les modules (exploits, auxiliary, scanners).

### Autres outils courants

| Outil | Configuration proxy |
|---|---|
| `curl` | `curl -x http://127.0.0.1:8080 <URL>` |
| `wget` | `wget -e http_proxy=127.0.0.1:8080 <URL>` |
| `ffuf` | `ffuf -x http://127.0.0.1:8080 ...` |
| `gobuster` | `gobuster dir --proxy http://127.0.0.1:8080 ...` |
| `sqlmap` | `sqlmap --proxy http://127.0.0.1:8080 ...` |
| `nikto` | `nikto -useproxy http://127.0.0.1:8080 ...` |

{% hint style="info" %}
Pour les outils Python, la variable d'environnement `HTTP_PROXY=http://127.0.0.1:8080` et `HTTPS_PROXY=http://127.0.0.1:8080` fonctionne souvent avec les scripts utilisant la librairie `requests`.
{% endhint %}

## Pieges et galeres

- Les outils qui font du HTTPS a travers le proxy vont echouer si le certificat CA du proxy n'est pas installe au niveau systeme (pas seulement dans Firefox).
- Certains outils ne respectent pas les variables d'environnement proxy. Verifier la documentation de chaque outil.
- Proxychains ne fonctionne pas avec les scans UDP de Nmap (`-sU`). Uniquement TCP connect (`-sT`).

## Retour terrain

Router un outil a travers le proxy est un reflexe de debug essentiel. Quand un module Metasploit echoue ou qu'un script Python ne retourne pas le resultat attendu, voir la requete et la reponse reelles dans Burp permet souvent d'identifier le probleme en quelques secondes. C'est aussi utile pour capturer les payloads exacts envoyes par un outil automatise et les adapter manuellement dans le Repeater.

## Memo express

| Methode | Commande |
|---|---|
| Proxychains | `proxychains -q <commande>` |
| Metasploit | `set PROXIES HTTP:127.0.0.1:8080` |
| curl | `-x http://127.0.0.1:8080` |
| Variable env | `HTTP_PROXY=http://127.0.0.1:8080` |
| Config | `/etc/proxychains.conf` |

***
