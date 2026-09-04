# Fuzzing de parametres

## Pourquoi

Les pages web acceptent souvent des parametres non documentes, que ce soit en GET (dans l'URL) ou en POST (dans le body). Ces parametres caches peuvent donner acces a des fonctionnalites d'administration, reveler des donnees sensibles, ou exposer des vulnerabilites. Les decouvrir par fuzzing est une etape essentielle du pentest web.

## Comment ca marche

Le principe est le meme que pour le fuzzing de repertoires : on place le marqueur `FUZZ` a la position du nom de parametre ou de sa valeur, et on teste chaque entree de la wordlist. La difference de taille entre la reponse par defaut et une reponse valide permet d'identifier les parametres acceptes.

## En pratique

### Fuzzing de parametres GET

Le parametre GET est passe dans l'URL apres le `?` :

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://<IP_CIBLE>/admin/admin.php?FUZZ=test \
     -fs <taille_par_defaut>
```

Le filtre `-fs` exclut les reponses de la taille par defaut (quand le parametre n'est pas reconnu). Les reponses avec une taille differente indiquent un parametre accepte.

### Fuzzing de parametres POST

Les parametres POST sont envoyes dans le body de la requete. Il faut specifier la methode et le Content-Type :

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://<IP_CIBLE>/admin/admin.php \
     -X POST \
     -d 'FUZZ=test' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs <taille_par_defaut>
```

{% hint style="info" %}
En PHP, le Content-Type `application/x-www-form-urlencoded` est obligatoire pour les requetes POST. Sans lui, les parametres ne seront pas interpretes par le serveur.
{% endhint %}

### Fuzzing de valeurs

Une fois un parametre identifie (par exemple `id`), fuzzer sa valeur pour trouver les entrees valides :

```bash
# Generer une wordlist de 1 a 1000
for i in $(seq 1 1000); do echo $i >> ids.txt; done

# Fuzzer la valeur du parametre id
ffuf -w ids.txt:FUZZ \
     -u http://<IP_CIBLE>/admin/admin.php \
     -X POST \
     -d 'id=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs <taille_par_defaut>
```

Une fois la valeur trouvee, verifier avec `curl` :

```bash
curl http://<IP_CIBLE>/admin/admin.php \
     -X POST \
     -d 'id=<VALEUR>' \
     -H 'Content-Type: application/x-www-form-urlencoded'
```

### Fuzzing multi-positions

ffuf supporte plusieurs marqueurs pour fuzzer plusieurs elements simultanement. Par exemple, fuzzer un nom d'utilisateur et un mot de passe en meme temps :

```bash
ffuf -w users.txt:USER -w passwords.txt:PASS \
     -u http://<IP_CIBLE>/login \
     -X POST \
     -d 'username=USER&password=PASS' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs <taille_par_defaut>
```

Par defaut, ffuf teste toutes les combinaisons (mode clusterbomb). Pour un mode pitchfork (ligne par ligne), ajouter `-mode pitchfork`.

## Pieges et galeres

- Les parametres peuvent etre sensibles a la casse. Tester avec une wordlist qui contient les deux formes si necessaire.
- Un parametre qui retourne `Invalid value!` au lieu de la page par defaut est un succes : le parametre est reconnu, il faut maintenant trouver la bonne valeur.
- Les wordlists de parametres comme `burp-parameter-names.txt` contiennent environ 6500 entrees. Le scan est rapide.
- Pour les valeurs numeriques, generer la wordlist avec un range adapte. Si rien n'est trouve dans 1-1000, etendre a 1-10000.

## Retour terrain

Le fuzzing de parametres est la technique qui suit naturellement la decouverte de pages. Une page d'administration avec un message "Access Denied" peut devenir accessible avec le bon parametre. En engagement, toujours tester les parametres GET et POST sur chaque page d'interet. Les parametres deprecies ou non documentes sont souvent les moins securises et les plus interessants.

## Memo express

| Objectif | Commande |
|---|---|
| Parametres GET | `ffuf -w params:FUZZ -u http://<IP>/page?FUZZ=val -fs <taille>` |
| Parametres POST | `ffuf -w params:FUZZ -u http://<IP>/page -X POST -d "FUZZ=val" -H "Content-Type: ..." -fs <taille>` |
| Valeurs d'un parametre | `ffuf -w vals:FUZZ -u http://<IP>/page -X POST -d "param=FUZZ" -fs <taille>` |
| Generer une liste d'IDs | `for i in $(seq 1 1000); do echo $i >> ids.txt; done` |
| Verifier avec curl | `curl http://<IP>/page -X POST -d "param=val" -H "Content-Type: ..."` |

***
