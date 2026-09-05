# Skills Assessment

Trois scenarios d'evaluation pratique pour valider les competences acquises sur l'ensemble du module. Chaque scenario simule un contexte de pentest realiste avec des applications differentes a identifier, enumerer et exploiter.

## Scenario 1 : enumeration et exploitation

### Contexte

On dispose d'une adresse IP cible. L'objectif est d'identifier les applications deployees, de trouver une application vulnerable, et d'obtenir une execution de code a distance pour recuperer un flag.

### Approche

```bash
# 1 - Enumeration des ports et services
nmap -sV --open -p- <IP_CIBLE>

# 2 - Capture d'ecran des services web identifies
eyewitness --web -x scan.xml -d rapport

# 3 - Identifier les applications et leurs versions
# 4 - Tester les identifiants par defaut
# 5 - Chercher les CVE pour chaque version identifiee
# 6 - Exploiter la vulnerabilite trouvee pour obtenir un RCE
```

### Ce qu'on en retient

- L'enumeration complete est indispensable avant de se lancer dans l'exploitation
- Les identifiants par defaut sont le vecteur le plus frequent
- Un seul service vulnerable suffit pour compromettre le serveur

## Scenario 2 : chaine d'attaque multi-applications

### Contexte

Plusieurs applications sont deployees sur la cible : un WordPress, un GitLab, et au moins un troisieme service sur un vhost a decouvrir. L'objectif est d'enchainer les compromissions pour obtenir un acces administrateur et un reverse shell.

### Approche

```bash
# 1 - Enumerer les vhosts
gobuster vhost -u http://<IP_CIBLE> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# 2 - Enumerer WordPress
wpscan --url http://wordpress.<IP_CIBLE>/ --enumerate ap,at,u

# 3 - Explorer les projets GitLab
# S'inscrire si l'inscription est ouverte
# Chercher des secrets dans le code source

# 4 - Pivoter entre les applications
# Les identifiants trouves dans GitLab peuvent fonctionner sur WordPress
# Le mot de passe admin WordPress donne acces a l'editeur de theme

# 5 - RCE via WordPress
# Modifier un fichier de theme pour deposer un webshell
```

### Ce qu'on en retient

- Le pivotage entre applications est une technique cle : un identifiant trouve dans GitLab peut deverrouiller WordPress
- La reutilisation de mots de passe entre applications est extremement courante
- Les vhosts cachent souvent des applications supplementaires

## Scenario 3 : client lourd sur Windows

### Contexte

Un hote Windows est accessible. L'objectif est d'analyser un client lourd pour trouver un mot de passe de base de donnees MSSQL stocke en dur dans le code.

### Approche

```bash
# 1 - Enumerer les services
nmap -sV --open -p- <IP_CIBLE>

# 2 - Identifier le client lourd (application .NET)
# Recuperer le binaire (partage SMB, service web, etc.)

# 3 - Decompiler avec dnSpy ou ILSpy
# Chercher les chaines de connexion MSSQL

# 4 - Deobfusquer si necessaire
de4dot MultimasterAPI.dll

# 5 - Extraire le mot de passe MSSQL
# Chercher "ConnectionString", "SqlConnection", "Password"
strings MultimasterAPI.dll | grep -i password
```

### Ce qu'on en retient

- Les clients lourds et les DLL d'API contiennent souvent des identifiants en dur
- La deobfuscation avec `de4dot` est une etape standard pour les binaires .NET
- Les chaines de connexion MSSQL suivent un format standard facilement reconnaissable : `Server=...;Database=...;User Id=...;Password=...`

***
