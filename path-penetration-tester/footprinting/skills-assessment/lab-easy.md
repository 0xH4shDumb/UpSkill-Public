# Lab - Enumeration d'un serveur DNS interne

## Scenario

Tu arrives sur un audit interne. Le client t'a fourni l'adresse d'un premier serveur qui fait office de DNS pour le reseau. L'objectif est clair : sans aucune exploitation agressive, tu dois collecter un maximum d'informations a partir des services exposes et demontrer ce qu'un attaquant pourrait obtenir en partant de rien (ou presque). On t'a communique un couple d'identifiants recuperes lors d'un precedent exercice de phishing interne, et des rumeurs internes mentionnent que certains employes utilisent des cles SSH pour se connecter a leurs machines.

## Approche

Le point de depart classique : un scan de ports complet pour cartographier les services. Sur un serveur DNS interne, on s'attend a trouver le port 53 (DNS), mais il faut regarder plus large. FTP, SSH, et d'autres services peuvent tourner sur des ports non standards.

La strategie se decompose en trois temps :

1. **Enumerer les services** pour identifier les points d'entree possibles
2. **Exploiter les identifiants connus** sur les services qui acceptent une authentification
3. **Pivoter** a partir des informations recuperees (fichiers de configuration, cles SSH, donnees sensibles)

{% hint style="info" %}
Un serveur peut exposer plusieurs instances d'un meme service sur des ports differents. Un FTP sur le port 21 et un autre sur le port 2121 n'ont pas forcement la meme configuration ni les memes utilisateurs.
{% endhint %}

## Commandes

### Scan initial

Depuis Exegol, on commence par un scan complet pour ne rien rater :

```bash
sudo nmap -sV -sC -p- <IP_CIBLE> -oN footprint_easy_tcp
```

| Option | Role |
|---|---|
| `-sV` | Detection de version des services |
| `-sC` | Execution des scripts par defaut |
| `-p-` | Scan de tous les ports TCP (1-65535) |
| `-oN` | Sauvegarde en format texte |

Dans la sortie, on repere les services et leurs bannieres. Un FTP qui affiche un nom personnalise (par exemple `ProFTPD Server (Ceil's FTP)`) indique souvent un service configure pour un utilisateur specifique.

### Connexion FTP avec identifiants

Si un service FTP ecoute sur un port non standard et que la banniere correspond a l'utilisateur dont on possede les identifiants :

```bash
ftp <UTILISATEUR>@<IP_CIBLE> <PORT>
```

Une fois connecte, on explore le repertoire personnel :

```bash
ftp> ls -al
ftp> cd .ssh
ftp> ls -al
ftp> get id_rsa
```

{% hint style="warning" %}
Toujours verifier les fichiers caches (`ls -al`). Le repertoire `.ssh` contient potentiellement des cles privees qui ouvrent un acces direct au serveur sans mot de passe.
{% endhint %}

### Utilisation de la cle privee SSH

Apres avoir recupere la cle privee, on lui attribue les bons droits puis on se connecte :

```bash
chmod 600 id_rsa
ssh -i id_rsa <UTILISATEUR>@<IP_CIBLE>
```

{% hint style="danger" %}
Sans `chmod 600`, SSH refusera d'utiliser la cle. C'est une protection contre les cles accessibles a d'autres utilisateurs du systeme.
{% endhint %}

### Exploration du systeme

Une fois connecte, on cherche les fichiers sensibles :

```bash
# - recherche de fichiers specifiques sur le systeme
find / -type f -name "flag.txt" 2>/dev/null

# - exploration des repertoires home
ls /home/
```

## Ce qu'on en retient

- Un scan de ports complet (`-p-`) est indispensable. Un service sur un port non standard passe completement inapercu avec un scan par defaut (top 1000 ports).
- Les bannieres de services sont des mines d'information. Un FTP qui annonce le nom d'un utilisateur dans sa banniere te dit exactement ou essayer tes identifiants.
- Le repertoire `.ssh` est une cible prioritaire des qu'on accede au home d'un utilisateur. Une cle privee permet de pivoter vers SSH sans connaitre de mot de passe supplementaire.
- La combinaison FTP + cle SSH est un chemin d'acces frequent en environnement interne : l'utilisateur stocke sa cle dans son home, et le FTP expose ce home sur le reseau.

***
