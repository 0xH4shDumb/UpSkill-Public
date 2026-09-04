# Sous-domaines et virtual hosts

## Pourquoi

Les sous-domaines et les virtual hosts (vhosts) hebergent souvent des interfaces d'administration, des environnements de staging, des API internes, ou des services non securises. Un scan de la page principale ne les revele pas. Il faut les decouvrir par fuzzing DNS ou fuzzing du header `Host`.

## Comment ca marche

### Sous-domaines vs virtual hosts

| Concept | Fonctionnement | Decouverte |
|---|---|---|
| Sous-domaine | Enregistrement DNS public (`sub.domaine.com` → IP) | Fuzzing DNS (resolution par le DNS public) |
| Virtual host | Meme IP, serveur web qui route selon le header `Host` | Fuzzing du header `Host` |

Un sous-domaine a un enregistrement DNS public : n'importe qui peut le resoudre. Un vhost n'a pas forcement d'enregistrement DNS. Plusieurs sites peuvent coexister sur la meme IP, routes par le serveur web en fonction du header `Host` de la requete.

### Resolution DNS locale

En lab, les domaines ne sont pas publics. Il faut ajouter manuellement l'entree dans `/etc/hosts` :

```bash
sudo sh -c 'echo "<IP_CIBLE>  domaine.htb" >> /etc/hosts'
```

Chaque sous-domaine ou vhost decouvert doit aussi etre ajoute dans `/etc/hosts` pour etre accessible dans le navigateur.

## En pratique

### Fuzzing de sous-domaines

Fonctionne uniquement sur les domaines avec DNS public (ou en lab si le DNS interne est configure) :

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://FUZZ.domaine.com/
```

Le marqueur `FUZZ` remplace le sous-domaine. ffuf tente de resoudre chaque sous-domaine via le DNS. Les reponses valides indiquent un sous-domaine existant.

{% hint style="info" %}
Ce type de scan ne fonctionne pas sur les domaines de lab (`.htb`) qui n'ont pas d'enregistrement DNS public. Pour ceux-la, utiliser le fuzzing de vhosts.
{% endhint %}

### Fuzzing de virtual hosts

Le fuzzing de vhosts envoie les requetes a la meme IP en modifiant le header `Host`. C'est la methode adaptee pour decouvrir des sous-domaines internes et des vhosts non publics :

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://<IP_CIBLE>/ \
     -H 'Host: FUZZ.domaine.htb'
```

### Filtrer les faux positifs

Le serveur repond a tous les headers `Host`, meme invalides, avec la page par defaut. Tous les resultats ont le meme code 200 et la meme taille de reponse. Il faut filtrer cette taille par defaut pour ne garder que les vhosts qui retournent une reponse differente :

```bash
# Lancer un premier scan pour identifier la taille par defaut (ex: 900 bytes)
# Puis relancer avec le filtre
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://<IP_CIBLE>/ \
     -H 'Host: FUZZ.domaine.htb' \
     -fs 900
```

Le filtre `-fs 900` exclut toutes les reponses de 900 bytes (la page par defaut), ne gardant que les vhosts qui retournent un contenu different.

## Pieges et galeres

- Oublier d'ajouter les sous-domaines/vhosts decouverts dans `/etc/hosts` : le navigateur ne pourra pas les resoudre.
- Le fuzzing de sous-domaines DNS echoue systematiquement sur les domaines non publics. Passer directement au fuzzing de vhosts dans ce cas.
- La taille de la reponse par defaut peut varier si le serveur inclut le hostname dans la page. Verifier la taille exacte sur une requete de reference avant de configurer le filtre.
- Certains serveurs bloquent les requetes sans header `Host` valide. Ajouter un User-Agent realiste avec `-H "User-Agent: Mozilla/5.0"` si necessaire.

## Retour terrain

En engagement interne, le fuzzing de vhosts est souvent plus productif que le fuzzing DNS. Les environnements de staging, les panels d'administration et les API internes sont regulierement configures comme vhosts sur la meme IP. La technique est simple : on identifie la taille de la reponse par defaut, on filtre, et les vhosts reels emergent. Chaque vhost decouvert ouvre une nouvelle surface d'attaque a explorer.

## Memo express

| Objectif | Commande |
|---|---|
| Sous-domaines DNS | `ffuf -w list:FUZZ -u http://FUZZ.domaine.com/` |
| Virtual hosts | `ffuf -w list:FUZZ -u http://<IP>/ -H "Host: FUZZ.domaine.htb"` |
| Avec filtre de taille | Ajouter `-fs <taille_par_defaut>` |
| Ajouter au hosts | `echo "<IP> sous.domaine.htb" >> /etc/hosts` |

***
