# WHOIS

## Pourquoi

WHOIS est un protocole de requête-réponse qui donne accès aux informations d'enregistrement des ressources Internet : noms de domaine, plages d'adresses IP, systèmes autonomes. C'est souvent le tout premier réflexe lors d'une phase de reconnaissance, et pour cause : en une seule commande, on peut obtenir le nom du propriétaire d'un domaine, ses contacts, ses serveurs DNS et les dates clés du cycle de vie du domaine.

## Comment ça marche

Quand on exécute une requête WHOIS sur un domaine, le client interroge un serveur WHOIS (port 43/TCP) qui renvoie les informations stockées par le registrar. Ces données sont structurées autour de plusieurs champs clés :

| Champ | Ce qu'il révèle |
|---|---|
| Domain Name | Le nom de domaine interrogé |
| Registrar | Le bureau d'enregistrement (OVH, GoDaddy, Amazon Registrar, etc.) |
| Registrant Contact | La personne ou l'organisation qui détient le domaine |
| Admin / Tech Contact | Les contacts administratifs et techniques |
| Creation / Expiry Date | Les dates de création et d'expiration |
| Name Servers | Les serveurs DNS autoritaires pour ce domaine |
| Domain Status | Les protections actives (transferProhibited, deleteProhibited, etc.) |

{% hint style="info" %}
Un domaine récent, enregistré via un service de confidentialité (privacy proxy), avec des serveurs DNS inhabituels, peut être un indicateur de domaine malveillant. A l'inverse, un domaine ancien avec des protections multiples indique généralement une organisation établie.
{% endhint %}

## En pratique

### Installation et utilisation de base

```bash
# - Installation sur Debian/Ubuntu
sudo apt update && sudo apt install whois -y

# - Requête WHOIS sur un domaine
whois <DOMAINE_CIBLE>
```

La sortie brute contient beaucoup d'informations. Ce qu'on cherche en priorité lors d'un pentest :

- **Les contacts** : noms, emails, numéros de téléphone. Exploitables pour du phishing ou de l'ingénierie sociale.
- **L'infrastructure DNS** : les name servers révèlent si le DNS est géré en interne ou externalisé.
- **Les dates** : un domaine qui expire bientôt ou qui vient d'être créé raconte une histoire différente.
- **Le registrar** : certains registrars sont connus pour leur tolérance aux abus, d'autres pour leur rigueur.

### Cas d'usage concrets

**Investigation de phishing** : un email suspect contient un lien vers un domaine inconnu. Une requête WHOIS révèle une date de création très récente, un propriétaire masqué derrière un proxy de confidentialité et des serveurs DNS associés à un hébergeur peu regardant. Suffisant pour bloquer le domaine et alerter les utilisateurs.

**Analyse de malware** : un binaire communique avec un serveur C2 dont le domaine, passé au WHOIS, montre un email jetable, une adresse postale dans un pays à forte activité cybercriminelle et un registrar réputé pour sa tolérance aux abus. Ces informations alimentent les indicateurs de compromission (IOC).

**Threat Intelligence** : en corrélant les enregistrements WHOIS de plusieurs domaines liés à un même groupe d'attaquants, on peut identifier des patterns : créations groupées dans le temps, alias récurrents, serveurs DNS partagés entre campagnes. Ce type d'analyse permet de tracer les TTPs d'un groupe APT.

### Aller plus loin : WHOIS historique

Des services comme WhoisFreaks ou DomainTools conservent l'historique des enregistrements WHOIS. En comparant les versions successives d'un même domaine, on peut détecter des changements d'hébergeur, des modifications de contacts ou des évolutions techniques qui racontent l'histoire d'une infrastructure.

## Pièges et galères

- **Privacy proxy** : de plus en plus de domaines masquent leurs informations derrière un service de confidentialité. Les données WHOIS ne montrent alors que le proxy, pas le véritable propriétaire.
- **Données obsolètes** : les informations WHOIS ne sont pas toujours à jour. Le propriétaire réel peut avoir changé sans que l'enregistrement le reflète.
- **RGPD et restrictions** : depuis l'entrée en vigueur du RGPD, de nombreux registrars européens masquent les données personnelles par défaut.

## Mémo express

| Action | Commande |
|---|---|
| Requête WHOIS basique | `whois <domaine>` |
| Filtrer un champ spécifique | `whois <domaine> \| grep -i "registrar"` |
| WHOIS sur une IP | `whois <adresse_ip>` |

***
