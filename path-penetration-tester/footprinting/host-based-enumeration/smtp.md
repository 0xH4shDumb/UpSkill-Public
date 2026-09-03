# SMTP

Le Simple Mail Transfer Protocol (SMTP) est le protocole standard pour l'envoi d'emails sur les réseaux IP. En test d'intrusion, un serveur SMTP mal configuré permet d'énumérer des utilisateurs, d'envoyer des mails d'hameçonnage, ou de relayer du spam. Son analyse fait partie de la routine d'énumération de tout réseau d'entreprise.

## Pourquoi

La messagerie est au coeur de la communication d'une entreprise. Le serveur SMTP gère l'envoi des mails, et sa configuration expose souvent des informations utiles : noms d'utilisateurs valides, domaine interne, intégrations tierces, voire la possibilité d'envoyer des mails au nom de n'importe qui (open relay). Identifier ces faiblesses sans même avoir besoin de credentials est un avantage considérable.

## Comment ça marche

### Architecture

Le workflow mail suit un parcours en plusieurs étapes :

| Composant | Rôle |
|---|---|
| MUA (Mail User Agent) | Client de messagerie (Thunderbird, Outlook) |
| MSA (Mail Submission Agent) | Reçoit le mail du MUA, authentifie et valide |
| MTA (Mail Transfer Agent) | Relaye le mail entre serveurs SMTP |
| MDA (Mail Delivery Agent) | Dépose le mail dans la boite du destinataire |

Le flux complet : MUA envoie via MSA, qui transmet au MTA, qui route vers le MTA du destinataire, qui passe au MDA pour livraison dans la boite mail.

### Ports

| Port | Usage |
|---|---|
| 25 | SMTP standard (relai entre serveurs) |
| 587 | SMTP avec authentification (STARTTLS) |
| 465 | SMTP sur SSL/TLS (ancien standard) |

### Sécurité

SMTP transmet les données en clair par défaut. Les mécanismes de protection existent mais ne sont pas toujours activés :

- **STARTTLS** : chiffrement de la session après la connexion initiale
- **SSL/TLS** : chiffrement dès l'établissement de la connexion (port 465)
- **SPF, DKIM, DMARC** : protection contre l'usurpation d'adresse d'expéditeur

{% hint style="info" %}
L'absence de SPF/DKIM/DMARC sur un domaine permet d'envoyer des mails en usurpant l'identité de n'importe quel employé. C'est un vecteur d'ingénierie sociale redoutable, souvent mentionné dans les rapports d'audit même quand il n'est pas exploité.
{% endhint %}

### Commandes SMTP

| Commande | Description |
|---|---|
| `HELO` / `EHLO` | Initie la session (EHLO retourne les extensions supportées) |
| `MAIL FROM` | Définit l'expéditeur |
| `RCPT TO` | Définit le destinataire |
| `DATA` | Démarre la transmission du corps du message |
| `VRFY` | Vérifie l'existence d'un utilisateur |
| `EXPN` | Liste les membres d'une liste de distribution |
| `QUIT` | Termine la session |

### Configuration (Postfix)

Exemple de configuration Postfix (`/etc/postfix/main.cf`) :

```ini
smtpd_banner = ESMTP Server
myhostname = mail.example.com
mydestination = $myhostname, localhost
mynetworks = 127.0.0.0/8 10.10.0.0/16
smtp_bind_address = 0.0.0.0
```

### Paramètres dangereux

| Paramètre | Risque |
|---|---|
| `mynetworks = 0.0.0.0/0` | Toute IP peut utiliser le serveur comme relai |
| `VRFY` activé | Permet d'énumérer les utilisateurs valides |
| STARTTLS non requis | Authentification en clair |
| AUTH sans chiffrement | Credentials interceptables |

## En pratique

### Identification du service

```bash
# depuis Exegol - scan de version et scripts NSE
sudo nmap -sC -sV -p25 <IP_CIBLE>
```

### Détection de relai ouvert

```bash
# depuis Exegol - test d'open relay
sudo nmap -p25 --script smtp-open-relay -v <IP_CIBLE>
```

Ce script effectue 16 tests pour déterminer si le serveur accepte l'envoi de mails sans authentification depuis des sources non autorisées.

### Enumération d'utilisateurs

```bash
# depuis Exegol - brute-force des comptes via VRFY
smtp-user-enum -M VRFY -U userlist.txt -t <IP_CIBLE> -p 25
```

{% hint style="warning" %}
Si `VRFY` est désactivé, les commandes `RCPT TO` et `EXPN` peuvent parfois servir d'alternatives pour énumérer les utilisateurs. smtp-user-enum supporte les trois méthodes via l'option `-M`.
{% endhint %}

### Interaction manuelle

```bash
# depuis Exegol - session telnet pour tester le comportement du serveur
telnet <IP_CIBLE> 25
```

```
EHLO test.local
MAIL FROM:<test@test.local>
RCPT TO:<admin@example.com>
DATA
Subject: Test
Ceci est un test.
.
QUIT
```

L'interaction manuelle permet de tester le comportement du serveur face à des requêtes spécifiques : accepte-t-il des expéditeurs arbitraires ? Rejette-t-il les domaines inconnus ? Requiert-il une authentification ?

### Récupération de la bannière

```bash
# depuis Exegol - lecture directe de la bannière
nc -nv <IP_CIBLE> 25
```

## Retour terrain

SMTP est un service qui donne beaucoup d'informations pour peu d'efforts. La bannière révèle souvent la version du serveur et le nom d'hôte interne. L'énumération VRFY fournit une liste de comptes valides exploitable pour du password spraying. Et quand le serveur est configuré en open relay, c'est un outil d'ingénierie sociale prêt à l'emploi.

En interne, les serveurs SMTP sont rarement durcis. Ils font partie de l'infrastructure de base et sont considérés comme "de confiance", ce qui conduit à des configurations permissives que personne ne révise.

## Mémo express

| Commande | Usage |
|---|---|
| `nc -nv <IP> 25` | Récupérer la bannière |
| `sudo nmap -sC -sV -p25 <IP>` | Scan de version + NSE |
| `sudo nmap -p25 --script smtp-open-relay <IP>` | Test d'open relay |
| `smtp-user-enum -M VRFY -U users.txt -t <IP>` | Enumération d'utilisateurs |
| `telnet <IP> 25` | Interaction manuelle |

***
