# Introduction au brute forcing

## Pourquoi

L'authentification par mot de passe reste le mecanisme le plus repandu pour proteger les acces. Si le mot de passe est faible, previsible ou reutilise, un attaquant peut le deviner en testant methodiquement toutes les combinaisons possibles. C'est le principe du brute forcing : essayer jusqu'a trouver la bonne cle.

En pentest, le brute forcing intervient quand les autres vecteurs (vulnerabilites connues, social engineering, exposition de credentials) n'ont rien donne. C'est aussi un moyen direct d'evaluer la robustesse des politiques de mots de passe d'une organisation.

## Comment ca marche

Le processus suit une boucle simple :

1. Generer une combinaison (mot de passe candidat)
2. L'envoyer au service cible (formulaire web, SSH, FTP, etc.)
3. Analyser la reponse : succes ou echec
4. Si echec, passer a la combinaison suivante
5. Si succes, l'acces est obtenu

La difficulte reside dans le nombre de combinaisons possibles, qui croit de maniere exponentielle avec la longueur et la complexite du mot de passe.

## Types d'attaques par brute forcing

| Methode | Principe | Cas d'usage |
|---|---|---|
| Brute force simple | Tester toutes les combinaisons d'un jeu de caracteres donne | Aucune information sur le mot de passe, ressources de calcul importantes |
| Attaque par dictionnaire | Utiliser une liste precompilee de mots de passe courants | Mot de passe probablement faible ou base sur des patterns communs |
| Attaque hybride | Combiner dictionnaire et mutations (ajout de chiffres, caracteres speciaux) | Mot de passe base sur un mot commun avec des modifications |
| Credential stuffing | Reutiliser des credentials fuites d'un service sur d'autres services | Base de credentials fuitees disponible, reutilisation probable |
| Password spraying | Tester quelques mots de passe courants sur beaucoup de comptes | Politique de verrouillage active, on repartit les tentatives |
| Rainbow table | Tables precalculees de hashes pour retrouver les mots de passe en clair | Grand nombre de hashes a craquer, espace de stockage disponible |
| Brute force inverse | Tester un seul mot de passe contre plusieurs comptes | Suspicion forte qu'un mot de passe est reutilise sur plusieurs comptes |

## Securite des mots de passe

### Recommandations NIST (SP 800-63B)

Le NIST recommande des mots de passe d'au moins 8 caracteres, avec une longueur maximale d'au moins 64 caracteres. Les regles de complexite rigides (majuscule + chiffre + symbole obligatoires) sont desormais decouragees au profit de mots de passe longs et uniques. Les passphrases (suite de mots) sont encouragees.

### Faiblesses courantes

Les mots de passe les plus repandus restent `123456`, `password`, `qwerty` et leurs variantes. Les utilisateurs appliquent des patterns previsibles :

- Majuscule en premiere position : `Password1`
- Chiffres a la fin : `summer2024`
- Substitutions classiques : `p@ssw0rd`
- Dates de naissance, prenoms, noms d'animaux

### Credentials par defaut

Beaucoup d'equipements et d'applications sont livres avec des credentials par defaut que les administrateurs ne changent pas toujours :

| Equipement / Application | Login | Mot de passe |
|---|---|---|
| Routeurs Cisco | admin | admin |
| Apache Tomcat | tomcat | s3cret |
| WordPress | admin | admin |
| MySQL | root | (vide) |
| phpMyAdmin | root | (vide) |
| Jenkins | admin | admin |

{% hint style="warning" %}
Toujours tester les credentials par defaut avant de lancer un brute force. C'est la premiere verification a faire et elle ne genere qu'une seule tentative de connexion.
{% endhint %}

### Impact de la complexite sur le temps de cassage

| Mot de passe | Longueur | Jeu de caracteres | Combinaisons possibles |
|---|---|---|---|
| Simple | 6 | Minuscules (a-z) | ~309 millions |
| Moyen | 8 | Minuscules (a-z) | ~209 milliards |
| Complexe | 8 | Minuscules + majuscules | ~53 billions |
| Tres complexe | 12 | Tous les caracteres ASCII | ~476 quintillions |

Meme un supercalculateur testant 1 billion de mots de passe par seconde mettrait environ 15 000 ans a craquer un mot de passe de 12 caracteres utilisant tous les caracteres ASCII.

## Pieges et galeres

- Le brute force n'est pas toujours la bonne approche. Si le service a une politique de verrouillage apres N tentatives, on se fait bloquer avant d'avoir teste quoi que ce soit d'interessant.
- Les CAPTCHAs, le rate limiting et les WAF peuvent rendre le brute force impraticable. Identifier ces protections avant de lancer l'attaque.
- Le brute force genere beaucoup de bruit dans les logs. En engagement, coordonner avec l'equipe bleue si necessaire.
- Ne pas confondre brute force (tester des combinaisons) et password spraying (tester peu de mots de passe sur beaucoup de comptes). Le spraying est souvent plus discret et plus efficace.

## Retour terrain

En engagement, le brute force est rarement la premiere option. On commence par chercher des credentials par defaut, des credentials fuitees (bases de donnees, pastes, repositories publics), ou des vulnerabilites d'authentification (bypass, injection). Le brute force intervient quand ces pistes sont epuisees ou qu'on cible un service specifique avec une surface d'attaque limitee.

La qualite de la wordlist fait toute la difference. Une wordlist generique comme rockyou.txt peut fonctionner, mais une wordlist construite a partir d'informations sur la cible (OSINT, politique de mots de passe, noms d'employes) sera bien plus efficace.

## Memo express

| Concept | Detail |
|---|---|
| Brute force simple | Toutes les combinaisons d'un charset |
| Dictionnaire | Liste precompilee (rockyou.txt, SecLists) |
| Hybride | Dictionnaire + mutations |
| Credential stuffing | Credentials fuitees reutilisees |
| Password spraying | Peu de mots de passe, beaucoup de comptes |
| Premiere verification | Credentials par defaut |
| Facteurs de succes | Qualite de la wordlist, absence de protections (lockout, CAPTCHA) |

***
