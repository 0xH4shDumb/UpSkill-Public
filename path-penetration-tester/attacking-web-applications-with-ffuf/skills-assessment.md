# Lab - Reconnaissance complete avec ffuf

## Scenario

L'equipe de pentest a recu l'adresse IP d'une academie en ligne, sans aucune information supplementaire. L'objectif est d'enumerer methodiquement toutes les surfaces d'attaque : sous-domaines, vhosts, repertoires, fichiers, parametres, et valeurs. Chaque etape s'appuie sur les resultats de la precedente.

## Approche

1. **Sous-domaines et vhosts** : fuzzer le header `Host` pour decouvrir tous les vhosts heberges sur l'IP.
2. **Ajouter au fichier hosts** : chaque vhost decouvert doit etre ajoute dans `/etc/hosts`.
3. **Extensions** : identifier les extensions utilisees sur chaque vhost (`.php`, `.phps`, `.html`).
4. **Repertoires et fichiers** : scan recursif avec les extensions identifiees sur chaque vhost.
5. **Parametres** : sur les pages d'interet, fuzzer les parametres GET et POST.
6. **Valeurs** : une fois un parametre identifie, fuzzer sa valeur.

## Commandes

### Decouverte de vhosts

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://<IP_CIBLE>:<PORT> \
     -H 'Host: FUZZ.domaine.htb' \
     -fs <taille_par_defaut>
```

### Ajout dans /etc/hosts

```bash
sudo sh -c 'echo "<IP_CIBLE> vhost1.domaine.htb vhost2.domaine.htb" >> /etc/hosts'
```

### Scan d'extensions par vhost

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://vhost.domaine.htb:<PORT>/indexFUZZ
```

### Scan recursif

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://vhost.domaine.htb:<PORT>/FUZZ \
     -recursion -recursion-depth 1 \
     -e .php,.phps \
     -fc 403 -fs 0 \
     -v
```

### Fuzzing de parametres POST

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://vhost.domaine.htb:<PORT>/chemin/page.php \
     -X POST \
     -d 'FUZZ=test' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs <taille_par_defaut>
```

### Fuzzing de valeurs (ex: usernames)

```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt:FUZZ \
     -u http://vhost.domaine.htb:<PORT>/chemin/page.php \
     -X POST \
     -d 'parametre=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs <taille_par_defaut>
```

### Verification avec curl

```bash
curl http://vhost.domaine.htb:<PORT>/chemin/page.php \
     -X POST \
     -d 'parametre=valeur' \
     -H 'Content-Type: application/x-www-form-urlencoded'
```

## Ce qu'on en retient

- La methodologie est iterative : chaque decouverte ouvre une nouvelle piste. Les vhosts menent a des repertoires, les repertoires a des fichiers, les fichiers a des parametres, les parametres a des valeurs.
- Le filtrage est indispensable a chaque etape. Sans lui, on est submerge de faux positifs.
- Toujours verifier les resultats manuellement avec `curl` ou le navigateur avant de conclure.
- Documenter chaque commande et chaque resultat. En engagement, ces notes alimentent directement le rapport.
- Le fichier `/etc/hosts` doit etre maintenu a jour tout au long de la reconnaissance. Un vhost non ajoute est un vhost invisible dans le navigateur.

***
