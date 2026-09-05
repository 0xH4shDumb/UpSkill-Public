# Attaquer osTicket et GitLab

osTicket et GitLab sont deux applications frequemment exposees en entreprise. osTicket est un systeme de ticketing open-source utilise pour le support client. GitLab est une plateforme de gestion de code source et de CI/CD. Ces deux applications offrent des surfaces d'attaque distinctes : osTicket permet la collecte d'informations sensibles et le SSRF, GitLab expose potentiellement du code source interne et des vulnerabilites RCE.

## Pourquoi

osTicket est interessant non pas pour son exploitation directe, mais pour les informations qu'il revele. Le systeme de tickets par email permet d'obtenir une adresse email valide de l'entreprise, et les tickets ouverts par les utilisateurs contiennent parfois des identifiants, des cles ou des informations confidentielles.

GitLab, en revanche, est une cible directe. Un acces en lecture aux depots internes peut reveler des secrets (cles API, mots de passe en dur dans le code), et plusieurs CVE critiques permettent un RCE authentifie ou pre-authentifie selon la version.

## Comment ca marche

### osTicket

#### Identification

osTicket est identifiable par :
- Le cookie `OSTSESSID` dans les reponses HTTP
- La page de soumission de ticket (`/open.php`)
- Le panneau agent (`/scp/login.php`)

#### Collecte d'adresses email

Quand un utilisateur cree un ticket, osTicket genere une adresse email de suivi (ex: `940288@support.entreprise.com`). Cette adresse email est liee au domaine de l'entreprise, ce qui permet de l'utiliser pour :

- S'inscrire sur d'autres services internes qui valident le domaine email
- Recevoir des emails envoyes au support (confirmation de compte, reinitialisation de mot de passe)

{% hint style="info" %}
Le numero de ticket est predictible (incremental). Si le dernier ticket est le #940288, les precedents sont #940287, #940286, etc. Chaque numero correspond a une adresse email valide qui recoit les reponses au ticket.
{% endhint %}

#### Exposition de donnees sensibles

Les tickets de support contiennent frequemment des informations sensibles :
- Mots de passe envoyes "temporairement" en clair
- Captures d'ecran contenant des identifiants
- Fichiers de configuration joints pour le debug
- Discussions internes avec des informations d'infrastructure

Si on obtient un acces agent (panneau d'administration), tous les tickets sont lisibles.

#### SSRF (CVE-2020-24881)

Sur les versions vulnerables, un SSRF permet d'interroger des services internes depuis le serveur osTicket :

```bash
# - SSRF via l'endpoint vulnerable
curl -s "http://<IP_CIBLE>/osticket/ajax.php/osticket/rss" \
    -d "url=http://127.0.0.1:8080/"
```

### GitLab

#### Identification

GitLab est identifiable par :
- La page de login avec le logo GitLab
- Le lien `/help` qui affiche la version exacte
- Le repertoire `/explore` pour les projets publics

```bash
# - Identifier la version GitLab
curl -s http://<IP_CIBLE>/help | grep "GitLab"
```

#### Enumeration

{% tabs %}
{% tab title="Inscription ouverte" %}
Si l'inscription est ouverte (configuration par defaut), creer un compte donne acces aux projets internes marques "Internal". Ces projets sont visibles uniquement par les utilisateurs authentifies mais pas par les visiteurs anonymes.

```bash
# - Verifier si l'inscription est ouverte
curl -s http://<IP_CIBLE>/users/sign_up | grep "Register"
```
{% endtab %}
{% tab title="Enumeration d'utilisateurs" %}
GitLab permet l'enumeration d'utilisateurs via la page d'inscription : si un nom d'utilisateur est deja pris, le message d'erreur le revele.

```bash
# - Script d'enumeration
for user in admin root gitlab-runner deploy; do
    curl -s "http://<IP_CIBLE>/users/$user" -o /dev/null -w "$user: %{http_code}\n"
done
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Les projets internes contiennent souvent des fichiers `.env`, des `docker-compose.yml` avec des mots de passe, des scripts de deploiement avec des identifiants en dur, ou des historiques de commits avec des secrets supprimes mais toujours accessibles dans le diff.
{% endhint %}

#### RCE authentifie (GitLab CE 13.10.2)

Cette vulnerabilite exploite le traitement des metadonnees ExifTool sur les images uploadees. Un fichier image specialement concu permet d'executer du code sur le serveur.

```bash
# - Exploitation avec le script dedie
python3 gitlab_13_10_2_rce.py \
    -u <USERNAME> -p <PASSWORD> \
    -c "id" -t http://<IP_CIBLE>
```

L'exploit necessite un compte utilisateur valide (meme un compte auto-enregistre suffit si l'inscription est ouverte).

## En pratique

```bash
# osTicket
# 1 - Identifier osTicket (cookie OSTSESSID, /open.php)
# 2 - Creer un ticket pour obtenir une adresse email valide
# 3 - Utiliser l'adresse pour s'inscrire sur d'autres services internes
# 4 - Si acces agent obtenu, chercher les identifiants dans les tickets

# GitLab
# 1 - Identifier la version via /help
curl -s http://<IP_CIBLE>/help | grep "GitLab"
# 2 - Verifier l'inscription ouverte et creer un compte
# 3 - Explorer les projets internes (/explore)
# 4 - Chercher des secrets dans le code et les commits
# 5 - Si version vulnerable, exploiter le RCE
```

## Pieges et galeres

{% tabs %}
{% tab title="osTicket" %}
- **Pas d'acces agent** : sans identifiants agent, on ne peut pas lire les tickets existants. Le vecteur principal reste la collecte d'adresses email via la creation de tickets
- **Rate limiting** : certaines configurations limitent la creation de tickets. Ne pas spam, un seul ticket suffit pour obtenir une adresse email
- **SSRF filtre** : le SSRF CVE-2020-24881 peut etre filtre par un WAF. Tester avec des variations d'adresse (decimal IP, IPv6)
{% endtab %}
{% tab title="GitLab" %}
- **Inscription desactivee** : si l'inscription est fermee, il faut trouver des identifiants par d'autres moyens (brute force, credential stuffing, OSINT)
- **Projets vides** : les projets internes sont parfois des repos vides ou de test. Prioriser ceux avec un historique de commits recent
- **Version non vulnerable** : le RCE ExifTool concerne une version specifique. Verifier la version dans `/help` avant de tenter l'exploit
- **2FA** : GitLab supporte le 2FA. Meme avec des identifiants valides, le 2FA bloque l'acces
{% endtab %}
{% endtabs %}

## Retour terrain

osTicket est rarement la cible principale, mais c'est un outil de collecte d'informations precieux. L'adresse email obtenue via un ticket peut debloquer l'acces a d'autres services (portails internes, VPN, inscription GitLab). Les tickets eux-memes sont une mine d'informations quand on obtient un acces agent. On y trouve regulierement des mots de passe envoyes en clair par les utilisateurs qui ont oublie le leur.

GitLab est une cible directe et souvent lucrative. L'inscription ouverte par defaut est un classique : les administrateurs deploient GitLab, importent leurs projets, et oublient de fermer l'inscription. Un simple compte auto-enregistre donne acces a tous les projets "Internal", qui contiennent souvent des secrets exploitables.

## Memo express

| Technique | Application | Vecteur | Prerequis |
|---|---|---|---|
| Email harvesting | osTicket | Creation de ticket | Acces au formulaire |
| Data exposure | osTicket | Lecture de tickets | Acces agent |
| SSRF | osTicket | CVE-2020-24881 | Version vulnerable |
| User enum | GitLab | Page d'inscription | Inscription ouverte |
| Code secrets | GitLab | Projets internes | Compte utilisateur |
| RCE | GitLab | ExifTool (CE 13.10.2) | Compte utilisateur |

***
