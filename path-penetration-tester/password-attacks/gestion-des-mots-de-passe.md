# Gestion des mots de passe

Apres avoir passe un module entier a casser, voler et reutiliser des mots de passe, il est naturel de se demander comment se proteger efficacement. Cette page couvre les bonnes pratiques de politiques de mots de passe et l'utilisation de gestionnaires de mots de passe, du point de vue d'un pentester qui sait exactement comment ces defenses sont contournees.

## Pourquoi

Comprendre les mecanismes de defense permet de mieux les evaluer lors d'un audit et de formuler des recommandations pertinentes. Un pentester qui sait comment fonctionne une politique de mots de passe peut identifier ses faiblesses. Un pentester qui connait le fonctionnement des gestionnaires peut evaluer si leur deploiement est correctement realise.

## Comment ca marche

### Politiques de mots de passe

Une politique de mots de passe definit les regles de creation, de gestion et de renouvellement des mots de passe dans une organisation. Elle couvre deux aspects : la **definition** des regles et leur **application technique** (enforcement).

#### Standards de reference

Plusieurs standards fournissent des recommandations sur les politiques de mots de passe.

| Standard | Organisme | Particularite |
|---|---|---|
| **NIST SP 800-63B** | NIST | Recommande de supprimer l'expiration periodique |
| **CIS Password Policy Guide** | CIS | Guide pratique detaille |
| **PCI DSS** | PCI SSC | Obligatoire pour le traitement de cartes bancaires |

{% hint style="info" %}
Le NIST a revise sa position sur l'expiration des mots de passe. Forcer un changement tous les 90 jours pousse les utilisateurs a adopter des patterns previsibles (`Company2025!` devient `Company2026!`). La recommandation actuelle est de ne forcer le changement qu'en cas de compromission confirmee.
{% endhint %}

#### Anatomie d'une politique type

Une politique classique impose :

- Minimum 8 caracteres
- Au moins une majuscule et une minuscule
- Au moins un chiffre
- Au moins un caractere special
- Ne peut pas etre le nom d'utilisateur

Le probleme, c'est que `Company2025!` respecte toutes ces regles tout en etant trivial a deviner par password spraying. C'est pourquoi les politiques robustes ajoutent une **liste noire de mots** : nom de l'entreprise, mois, saisons, variations de "welcome" et "password".

#### Enforcement en Active Directory

En environnement AD, la politique se configure via une GPO (Group Policy Object) qui applique les regles a l'ensemble du domaine ou a des groupes specifiques. Des outils tiers comme des filtres de mots de passe personnalises peuvent renforcer les controles au-dela de ce que la GPO native propose.

### Gestionnaires de mots de passe

Un gestionnaire de mots de passe est une application qui stocke les identifiants dans une base chiffree protegee par un mot de passe maitre. L'utilisateur n'a besoin de retenir qu'un seul mot de passe pour acceder a tous les autres.

#### Fonctionnement

Le principe repose sur la derivation de cles cryptographiques a partir du mot de passe maitre :

| Etape | Description |
|---|---|
| **Cle maitre** | Derivee du mot de passe maitre via une KDF (PBKDF2, Argon2) |
| **Hash d'authentification** | Utilise pour s'authentifier aupres du service cloud |
| **Cle de dechiffrement** | Generee a partir de la cle maitre pour dechiffrer le coffre |

Les gestionnaires cloud implementent le **Zero-Knowledge Encryption** : le fournisseur ne peut jamais acceder aux donnees stockees, car le dechiffrement se fait uniquement cote client.

{% tabs %}
{% tab title="Cloud" %}
Synchronisation multi-appareils, extensions navigateur, partage securise.

| Gestionnaire | Particularite |
|---|---|
| **Bitwarden** | Open source, auto-hebergeable |
| **1Password** | Secret Key en plus du mot de passe maitre |
| **Dashlane** | VPN integre |
| **KeeperSecurity** | Zero-knowledge certifie |
| **LastPass** | Historiquement populaire (incidents de securite en 2022) |
{% endtab %}
{% tab title="Local" %}
Base de donnees stockee localement, l'utilisateur gere la securite et les sauvegardes.

| Gestionnaire | Particularite |
|---|---|
| **KeePass** | Reference open source, fichier `.kdbx` |
| **KWalletManager** | Integre a KDE |
| **Password Safe** | Cree par Bruce Schneier |
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Du point de vue d'un pentester, les fichiers `.kdbx` (KeePass) trouves sur un systeme compromis sont des cibles de choix. L'outil `keepass2john` extrait le hash du mot de passe maitre, qui peut ensuite etre soumis a une attaque par dictionnaire.
{% endhint %}

### Alternatives aux mots de passe

| Methode | Principe |
|---|---|
| **MFA** | Combinaison de facteurs (mot de passe + OTP, biometrie...) |
| **FIDO2 / WebAuthn** | Authentification sans mot de passe via cle physique (YubiKey) |
| **OTP / TOTP** | Codes a usage unique generes par une application (Authy, Google Authenticator) |
| **Passwordless** | Suppression totale du mot de passe au profit de la biometrie ou d'un appareil physique |

{% hint style="success" %}
La tendance actuelle est au passwordless. Microsoft, Okta et Auth0 proposent des solutions qui eliminent completement les mots de passe, en s'appuyant sur des facteurs de possession (cle physique) ou d'inherence (biometrie). C'est la meilleure protection contre le phishing et le credential stuffing.
{% endhint %}

## En pratique

### Creer un mot de passe robuste

Deux approches valides :

```
# - Mot de passe aleatoire (genere par un gestionnaire)
CjDC2x[U9#kL

# - Passphrase (facile a retenir, difficile a casser)
()Le nom de mon chat est Felix!
```

Une passphrase de 5+ mots avec des caracteres speciaux resiste a des milliards d'annees de brute force tout en etant memorisable.

### Recommandations post-audit

Lors de la redaction du rapport, les recommandations les plus courantes :

| Constat | Recommandation |
|---|---|
| Reutilisation du mot de passe admin local | Deployer **LAPS** (Local Administrator Password Solution) |
| Mots de passe saisonniers (`Company2025!`) | Ajouter une liste noire dans la politique AD |
| Pas de MFA | Deployer le MFA sur tous les acces critiques |
| Fichiers `.kdbx` sur des partages | Stocker les bases KeePass hors des partages reseau |
| Credentials dans des scripts | Utiliser un coffre-fort de secrets (HashiCorp Vault, Azure Key Vault) |

## Pieges et galeres

- **Politique trop stricte** : forcer des rotations frequentes et des regles complexes pousse les utilisateurs vers des mots de passe previsibles ou des post-it sur l'ecran
- **Gestionnaire cloud compromis** : les incidents de securite chez les fournisseurs (LastPass en 2022) rappellent qu'aucune solution n'est infaillible. Le zero-knowledge limite l'impact, mais le mot de passe maitre reste le maillon faible
- **MFA bypass** : le MFA n'est pas une solution magique. Les attaques par phishing en temps reel (Evilginx2), le SIM swapping et le fatigue MFA permettent de le contourner
- **FIDO2 et adoption** : les cles physiques sont la meilleure protection, mais leur deploiement a grande echelle reste un defi logistique et financier

## Memo express

| Element | Bonnes pratiques |
|---|---|
| **Longueur** | 12+ caracteres minimum (16+ recommande) |
| **Complexite** | Preferer les passphrases aux mots de passe courts complexes |
| **Expiration** | Ne forcer le changement qu'en cas de compromission |
| **Liste noire** | Bloquer le nom de l'entreprise, les mois, les patterns courants |
| **Stockage** | Gestionnaire de mots de passe (cloud ou local) |
| **MFA** | Activer sur tous les acces critiques |
| **Admin local** | Deployer LAPS pour eviter la reutilisation |
| **Secrets applicatifs** | Coffre-fort de secrets, jamais en dur dans le code |

***
