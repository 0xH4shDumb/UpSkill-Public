# Fondamentaux du DNS

## Pourquoi

Le DNS (Domain Name System) est le système qui traduit les noms de domaine lisibles par les humains en adresses IP exploitables par les machines. Sans DNS, il faudrait retenir des adresses comme `142.251.47.142` au lieu de taper `google.com`. Pour un pentester, le DNS est bien plus qu'un service de résolution : c'est une source d'information considérable sur l'infrastructure d'une cible.

## Comment ça marche

### Architecture hiérarchique

Le DNS fonctionne comme un annuaire distribué, organisé en arborescence :

1. **Serveurs racine** (Root Servers) : le point de départ de toute résolution DNS. Il en existe 13 groupes, identifiés de A à M.
2. **Serveurs TLD** (Top-Level Domain) : gèrent les domaines de premier niveau (.com, .org, .fr, etc.).
3. **Serveurs autoritaires** : détiennent les enregistrements DNS réels pour un domaine donné.
4. **Résolveurs récursifs** : les intermédiaires qui reçoivent les requêtes des clients et parcourent la hiérarchie pour obtenir la réponse.

### Processus de résolution

Quand on tape `exemple.com` dans un navigateur :

1. Le système vérifie d'abord le cache local et le fichier `/etc/hosts`.
2. Si rien n'est trouvé, la requête part vers le résolveur récursif (souvent celui du FAI ou un public comme `1.1.1.1`).
3. Le résolveur interroge successivement les serveurs racine, puis le TLD, puis le serveur autoritaire.
4. La réponse finale (l'adresse IP) remonte la chaîne et est mise en cache à chaque niveau pour accélérer les requêtes suivantes.

### Types d'enregistrements DNS

Chaque domaine peut avoir plusieurs types d'enregistrements, chacun avec un rôle précis :

| Type | Rôle | Exemple |
|---|---|---|
| `A` | Associe un nom à une adresse IPv4 | `exemple.com → 93.184.216.34` |
| `AAAA` | Associe un nom à une adresse IPv6 | `exemple.com → 2606:2800:220:1:...` |
| `MX` | Désigne les serveurs de messagerie | `exemple.com → mail.exemple.com` |
| `NS` | Indique les serveurs DNS autoritaires | `exemple.com → ns1.exemple.com` |
| `TXT` | Contient du texte libre (SPF, DKIM, vérification) | `v=spf1 include:_spf.google.com ~all` |
| `CNAME` | Alias pointant vers un autre nom | `www.exemple.com → exemple.com` |
| `SOA` | Informations sur la zone (serveur primaire, email admin, serial) | Start of Authority de la zone |
| `PTR` | Résolution inverse (IP vers nom) | `34.216.184.93 → exemple.com` |
| `SRV` | Localise un service spécifique (port, protocole) | `_sip._tcp.exemple.com` |

{% hint style="success" %}
En pentest, les enregistrements `MX` révèlent les serveurs mail (cibles potentielles), les `TXT` peuvent contenir des tokens de vérification exploitables, et les `NS` indiquent si le DNS est géré en interne ou externalisé.
{% endhint %}

### Fichiers de zone

Un fichier de zone DNS est le fichier de configuration qui contient tous les enregistrements d'un domaine sur un serveur autoritaire. Sa structure suit un format standardisé :

```
$TTL 86400
@   IN  SOA   ns1.exemple.com. admin.exemple.com. (
            2024010101  ; Serial
            3600        ; Refresh
            900         ; Retry
            1209600     ; Expire
            86400       ; Minimum TTL
)
@       IN  NS    ns1.exemple.com.
@       IN  NS    ns2.exemple.com.
@       IN  A     93.184.216.34
www     IN  CNAME exemple.com.
mail    IN  A     93.184.216.35
@       IN  MX 10 mail.exemple.com.
```

Le TTL (Time To Live) détermine combien de temps un enregistrement reste en cache. Un TTL court signifie des mises à jour fréquentes, un TTL long favorise les performances.

## Retour terrain

Le DNS est souvent le premier service qu'on interroge lors d'une reconnaissance. Un simple `dig` sur un domaine peut révéler des serveurs internes, des sous-domaines oubliés, des services de messagerie, et parfois des informations de configuration qui n'auraient jamais du être publiques. La suite de ce module explore les outils et techniques pour exploiter pleinement cette mine d'information.

***
