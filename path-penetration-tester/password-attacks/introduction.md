# Introduction aux attaques de mots de passe

Le mot de passe reste le mecanisme d'authentification le plus repandu. Malgre l'adoption croissante du MFA et de la biometrie, la grande majorite des systemes reposent encore sur un couple identifiant/mot de passe. Comprendre comment les mots de passe sont stockes, attaques et contournes est une competence fondamentale en pentest.

## Pourquoi

Les statistiques sont eloquentes : en 2025, `123456` reste le mot de passe le plus utilise dans les fuites de donnees. Plus de la moitie des utilisateurs reutilisent le meme mot de passe sur plusieurs comptes, et beaucoup ne le changent pas meme apres une compromission. Ce module couvre l'ensemble du spectre : attaques distantes, cassage de hashs, extraction de credentials sur Windows et Linux, mouvement lateral, et bonnes pratiques de gestion.

## Comment ca marche

### Facteurs d'authentification

L'authentification repose sur la verification d'un ou plusieurs facteurs.

| Facteur | Description | Exemple |
|---|---|---|
| **Connaissance** | Quelque chose que l'utilisateur connait | Mot de passe, PIN, passphrase |
| **Possession** | Quelque chose que l'utilisateur possede | Token materiel, cle de securite, smartphone (MFA) |
| **Inherence** | Quelque chose que l'utilisateur est | Empreinte digitale, reconnaissance faciale |
| **Localisation** | Ou se trouve l'utilisateur | Geolocalisation, adresse IP |

Un systeme peut combiner plusieurs facteurs (MFA) pour augmenter la securite. Un mot de passe seul est considere comme un facteur unique, donc faible.

### Stockage des mots de passe

Les systemes ne stockent pas les mots de passe en clair (en theorie). Ils utilisent des fonctions de hachage pour transformer le mot de passe en une empreinte de taille fixe.

```bash
# - Hash MD5 d'un mot de passe
echo -n "MonMotDePasse" | md5sum
# 5a105e8b9d40e1329780d62ea2265d8a

# - Hash SHA-256
echo -n "MonMotDePasse" | sha256sum
# a5b9d...
```

{% hint style="info" %}
Le hachage est une fonction unidirectionnelle : il ne devrait pas etre possible de retrouver le mot de passe a partir du hash. Les attaques de cassage tentent de trouver un mot de passe qui produit le meme hash, soit par brute force, soit par dictionnaire.
{% endhint %}

### Salage

Pour contrer les rainbow tables (tables de correspondances precomputees hash/mot de passe), les systemes ajoutent un **salt** (valeur aleatoire) au mot de passe avant le hachage. Le salt est stocke en clair a cote du hash. Il rend les rainbow tables inutilisables car chaque mot de passe produit un hash different meme si le mot de passe est identique.

### Panorama des attaques

| Type d'attaque | Cible | Technique |
|---|---|---|
| **Brute force distante** | Services reseau (SSH, RDP, SMB, WinRM) | Hydra, NetExec, Metasploit |
| **Spraying** | Comptes multiples | Un mot de passe teste sur N comptes |
| **Credential stuffing** | Reutilisation de credentials | Paires user:pass issues de fuites |
| **Cassage offline** | Hashs extraits | Hashcat, John the Ripper |
| **Extraction locale** | SAM, LSASS, NTDS.dit, /etc/shadow | secretsdump, pypykatz, mimikatz |
| **Mouvement lateral** | Sessions authentifiees | Pass-the-Hash, Pass-the-Ticket |

## En pratique

### Verification de compromission

Avant meme de lancer des attaques, il est utile de verifier si des credentials de l'organisation cible ont deja fuite.

- **HaveIBeenPwned** : verification d'emails dans les fuites publiques
- **DeHashed** : recherche de credentials dans les bases de donnees compromises
- **IntelX** : moteur de recherche de fuites

### Outils principaux

| Outil | Usage |
|---|---|
| **Hydra** | Brute force multi-protocole (SSH, RDP, FTP, HTTP, etc.) |
| **NetExec** | Brute force et enumeration Windows (SMB, WinRM, RDP, LDAP) |
| **Hashcat** | Cassage de hashs GPU (MD5, NTLM, SHA, DCC2, etc.) |
| **John the Ripper** | Cassage de hashs CPU + conversion de formats (ssh2john, zip2john) |
| **Mimikatz** | Extraction de credentials en memoire (Windows) |
| **secretsdump** | Dump distant de SAM, LSA secrets, NTDS.dit |
| **CeWL** | Generation de wordlists depuis un site web |

## Retour terrain

Ce module est l'un des plus transversaux du parcours CPTS. Les attaques de mots de passe interviennent a chaque phase d'un pentest : depuis le brute force initial sur un service expose jusqu'au mouvement lateral apres compromission d'un domaine AD. La cle est de savoir choisir la bonne technique au bon moment. Un brute force massif sur SSH generera des alertes, alors qu'un spray discret sur AD avec un mot de passe saisonnalement probable (`Company2025!`) passera souvent inapercu.

***
