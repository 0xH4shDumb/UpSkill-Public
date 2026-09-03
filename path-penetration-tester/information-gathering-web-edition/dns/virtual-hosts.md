# Virtual Hosts

## Pourquoi

Les Virtual Hosts (VHosts) permettent à un seul serveur web d'héberger plusieurs sites ou applications sur une même adresse IP. Apache, Nginx, IIS : tous les serveurs web majeurs supportent cette fonctionnalité. Pour un pentester, les VHosts sont importants parce qu'un serveur peut héberger des applications cachées qui ne sont pas référencées dans le DNS public. On peut y trouver des portails d'administration, des environnements de test, ou des applications internes accessibles uniquement si on connaît le bon nom d'hôte.

## Comment ça marche

### Le rôle de l'en-tête Host

Quand un navigateur envoie une requête HTTP, il inclut un en-tête `Host` contenant le nom de domaine demandé. Le serveur web utilise cette valeur pour déterminer quel site servir :

1. Le navigateur envoie une requête avec `Host: app.exemple.com`.
2. Le serveur web consulte sa configuration de Virtual Hosts.
3. Il associe la requête au bon `DocumentRoot` (répertoire de fichiers).
4. Il renvoie le contenu correspondant.

### Sous-domaines vs Virtual Hosts

La distinction est importante :

- **Sous-domaines** : extensions du domaine principal (`blog.exemple.com`) avec leur propre enregistrement DNS. Découvrables par énumération DNS.
- **Virtual Hosts** : configurations internes du serveur web, pas forcément référencées dans le DNS. Deux VHosts peuvent pointer vers la même IP mais servir des contenus totalement différents.

{% hint style="warning" %}
Un VHost peut exister sans enregistrement DNS public. C'est pour ça que l'énumération DNS seule ne suffit pas : il faut aussi fuzzer les VHosts directement.
{% endhint %}

### Types de Virtual Hosting

| Type | Description |
|---|---|
| Name-based | Le plus courant. Utilise l'en-tête `Host` pour différencier les sites. Une seule IP suffit. |
| IP-based | Chaque site utilise une adresse IP distincte. Offre une meilleure isolation. |
| Port-based | Chaque site répond sur un port différent (`:80`, `:8080`). Peu fréquent en production. |

### Exemple de configuration Apache

```apacheconf
<VirtualHost *:80>
    ServerName www.site-a.com
    DocumentRoot /var/www/site-a
</VirtualHost>

<VirtualHost *:80>
    ServerName admin.site-a.com
    DocumentRoot /var/www/admin
</VirtualHost>
```

Deux sites totalement différents, même IP, même port. Seul l'en-tête `Host` les distingue.

## En pratique

### Découverte de VHosts avec gobuster

`gobuster` en mode `vhost` teste systématiquement des noms d'hôtes en les injectant dans l'en-tête `Host` et en comparant les réponses :

```bash
gobuster vhost -u http://<IP_CIBLE>:<PORT> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  --append-domain \
  -t 50
```

| Option | Rôle |
|---|---|
| `-u` | URL cible (IP ou domaine) |
| `-w` | Wordlist de noms à tester |
| `--append-domain` | Ajoute le domaine principal aux mots de la wordlist |
| `-t` | Nombre de threads (accélère le scan) |
| `-k` | Ignore les erreurs de certificat SSL |
| `-o` | Sauvegarde la sortie dans un fichier |

#### Exemple de sortie

```
Found: admin.cible.htb Status: 200 [Size: 100]
Found: blog.cible.htb Status: 200 [Size: 98]
Found: forum.cible.htb Status: 200 [Size: 100]
Found: support.cible.htb Status: 200 [Size: 104]
```

### Autres outils

| Outil | Particularité |
|---|---|
| `ffuf` | Fuzzer web très flexible, supporte le fuzzing de l'en-tête `Host` |
| `feroxbuster` | Fuzzer rapide écrit en Rust |
| `wfuzz` | Outil de fuzzing polyvalent |

### Accéder à un VHost découvert

Une fois un VHost identifié, il faut l'ajouter au fichier `/etc/hosts` pour pouvoir y accéder :

```bash
echo '<IP_CIBLE>    admin.cible.htb' >> /etc/hosts
```

## Pièges et galères

- **Filtrage par taille de réponse** : beaucoup de serveurs renvoient une page par défaut pour les VHosts inexistants. Il faut filtrer les résultats par taille de réponse (`--exclude-length` avec gobuster, ou `-fs` avec ffuf) pour ne garder que les réponses différentes.
- **Volume de trafic** : le fuzzing de VHosts génère un grand nombre de requêtes. Les IDS/WAF peuvent détecter et bloquer ce comportement.
- **HTTPS et SNI** : en HTTPS, l'en-tête `Host` est chiffré mais le SNI (Server Name Indication) dans le handshake TLS peut révéler des VHosts. Penser à tester aussi en HTTPS.

## Mémo express

| Besoin | Commande |
|---|---|
| Fuzzing VHosts (gobuster) | `gobuster vhost -u http://<ip> -w <wordlist> --append-domain` |
| Fuzzing VHosts (ffuf) | `ffuf -u http://<ip> -H "Host: FUZZ.<domaine>" -w <wordlist>` |
| Ajouter un VHost au hosts | `echo '<ip> <vhost>' >> /etc/hosts` |

***
