# IMAP / POP3

IMAP (Internet Message Access Protocol) et POP3 (Post Office Protocol) sont les deux protocoles utilisés pour la réception des emails. Là ou SMTP gère l'envoi, IMAP et POP3 permettent de consulter et gérer les messages stockés sur le serveur de messagerie.

## Pourquoi

Un serveur IMAP/POP3 accessible est une cible de choix en pentest. Les boites mail contiennent régulièrement des informations sensibles : mots de passe échangés en interne, configurations de services, rapports, documents confidentiels. Un accès, même avec un compte à faibles privilèges, peut révéler des données critiques pour la suite de l'engagement.

## Comment ça marche

### Différences entre IMAP et POP3

| Caractéristique | IMAP | POP3 |
|---|---|---|
| Stockage | Messages conservés sur le serveur | Messages téléchargés en local |
| Synchronisation | Multi-client (état partagé) | Un seul client (pas de sync) |
| Gestion de dossiers | Oui (INBOX, Sent, Drafts, custom) | Non |
| Port standard | 143 (993 avec SSL/TLS) | 110 (995 avec SSL/TLS) |
| Cas d'usage | Entreprise, multi-appareils | Consultation simple, archivage local |

IMAP est largement dominant en entreprise : il permet de consulter ses mails depuis plusieurs appareils avec un état synchronisé (lu/non lu, dossiers, flags). POP3 est plus simple mais moins flexible.

{% hint style="warning" %}
Les deux protocoles transmettent les identifiants et le contenu des mails en clair par défaut. Sans SSL/TLS, un attaquant positionné sur le réseau peut intercepter les credentials et le contenu des messages.
{% endhint %}

### Implémentation serveur

**Dovecot** est l'implémentation la plus courante sous Linux. Il gère à la fois IMAP et POP3 et s'installe en quelques paquets :

```bash
apt install dovecot-imapd dovecot-pop3d
```

### Paramètres sensibles

| Paramètre | Risque |
|---|---|
| `auth_debug_passwords` | Enregistre les mots de passe soumis dans les logs |
| `auth_anonymous_username` | Définit un compte anonyme par défaut |
| `auth_verbose` | Log les tentatives d'authentification échouées (utile pour l'attaquant qui surveille les logs) |

### Commandes IMAP

| Commande | Description |
|---|---|
| `1 LOGIN user password` | Authentification |
| `1 LIST "" *` | Lister tous les dossiers |
| `1 SELECT INBOX` | Ouvrir la boite de réception |
| `1 FETCH 1 all` | Récupérer les métadonnées du message 1 |
| `1 FETCH 1 body[]` | Récupérer le contenu complet du message 1 |
| `1 LOGOUT` | Fermer la session |

### Commandes POP3

| Commande | Description |
|---|---|
| `USER username` | S'identifier |
| `PASS password` | Envoyer le mot de passe |
| `LIST` | Lister les messages |
| `RETR 1` | Récupérer le message 1 |
| `DELE 1` | Supprimer le message 1 |
| `QUIT` | Fermer la session |

## En pratique

### Scan de ports

```bash
# depuis Exegol - detection des services IMAP/POP3
sudo nmap -sV -sC -p110,143,993,995 <IP_CIBLE>
```

Le scan révèle la version du serveur (Dovecot, Cyrus, Exchange), les capacités supportées (SASL, STARTTLS, IDLE) et le certificat SSL qui contient souvent le FQDN du serveur et le nom de l'organisation.

{% hint style="info" %}
Le certificat SSL d'un serveur IMAP/POP3 est une mine d'informations passives. Le champ `Subject` contient le Common Name (FQDN du serveur), le nom de l'organisation, la ville et le pays. Ces données sont récupérables sans authentification.
{% endhint %}

### Connexion via curl

```bash
# depuis Exegol - lister les dossiers IMAP
curl -k 'imaps://<IP_CIBLE>' --user user:password
```

### Connexion via openssl

{% tabs %}
{% tab title="IMAP" %}
```bash
# depuis Exegol - session IMAP sur TLS
openssl s_client -connect <IP_CIBLE>:993
```

Une fois connecté, utiliser les commandes IMAP (`LOGIN`, `LIST`, `SELECT`, `FETCH`).
{% endtab %}
{% tab title="POP3" %}
```bash
# depuis Exegol - session POP3 sur TLS
openssl s_client -connect <IP_CIBLE>:995
```

Une fois connecté, utiliser les commandes POP3 (`USER`, `PASS`, `LIST`, `RETR`).
{% endtab %}
{% endtabs %}

### Lecture des mails

Une fois authentifié en IMAP :

```
1 LIST "" *
1 SELECT INBOX
1 FETCH 1 all
1 FETCH 1 body[]
```

Le `FETCH ... all` retourne les métadonnées (expéditeur, destinataire, date, sujet). Le `FETCH ... body[]` retourne le contenu complet du message, y compris les pièces jointes encodées en base64.

## Pièges et galères

- Les dossiers IMAP ne se limitent pas à INBOX. Des dossiers personnalisés (DEV, IMPORTANT, ADMIN) contiennent parfois des informations plus sensibles que la boite de réception classique. Toujours lister tous les dossiers avec `LIST "" *`.
- Les mots de passe faibles (username = password) sont étonnamment courants sur les serveurs de messagerie internes. C'est le premier test à faire avec un nom d'utilisateur découvert.

## Retour terrain

Les boites mail sont souvent le jackpot d'un pentest interne. Un accès avec un compte utilisateur standard suffit pour trouver des échanges contenant des mots de passe en clair ("voici les identifiants du serveur de prod"), des fichiers de configuration en pièce jointe, ou des informations sur l'infrastructure interne. Les dossiers "Envoyés" et "Brouillons" sont aussi riches que la boite de réception.

## Mémo express

| Commande | Usage |
|---|---|
| `sudo nmap -sV -sC -p110,143,993,995 <IP>` | Scan de version + NSE |
| `curl -k 'imaps://<IP>' --user user:pass` | Lister les dossiers IMAP |
| `openssl s_client -connect <IP>:993` | Session IMAP TLS |
| `openssl s_client -connect <IP>:995` | Session POP3 TLS |
| `nc -nv <IP> 110` | Bannière POP3 en clair |

***
