# Cloture de l'engagement

La fin d'un test d'intrusion ne se limite pas a la decouverte de la derniere vulnerabilite. La phase de cloture englobe le nettoyage de l'environnement, la redaction du rapport, la livraison au client et le debriefing. C'est cette phase qui transforme un exercice technique en livrable professionnel.

## Pourquoi

Un pentest sans rapport exploitable n'a aucune valeur pour le client. La cloture est le moment ou le travail technique est traduit en recommandations actionnables, ou les artefacts de test sont supprimes et ou la relation de confiance avec le client se concretise par un echange constructif. Un nettoyage incomplet peut laisser des portes ouvertes dans l'environnement du client.

## Comment ca marche

### Nettoyage de l'environnement

Chaque modification apportee a l'environnement pendant le test doit etre annulee. Le journal de bord tenu pendant l'engagement sert de checklist.

| Element | Action de nettoyage |
|---|---|
| **Comptes crees** | Supprimer les comptes locaux et de domaine crees |
| **Cles SSH ajoutees** | Retirer les cles publiques de `authorized_keys` |
| **Fichiers deposes** | Supprimer les outils, scripts et web shells uploades |
| **SPN modifies** | Retirer les faux SPN ajoutes pour le Kerberoasting cible |
| **Mots de passe changes** | Restaurer les mots de passe originaux ou notifier le client |
| **Regles de firewall** | Supprimer les regles ajoutees pour le pivoting |
| **Taches planifiees** | Retirer les mecanismes de persistence |
| **Configurations modifiees** | Restaurer les fichiers de configuration originaux |

{% hint style="danger" %}
Les modifications qui ne peuvent pas etre annulees proprement (comme un mot de passe change dont l'original est inconnu) doivent etre signalees immediatement au client avec les details necessaires pour qu'il puisse intervenir.
{% endhint %}

### Communication pendant l'engagement

La communication ne commence pas avec le rapport final. Certains evenements necessitent une notification immediate.

| Evenement | Action |
|---|---|
| **Vulnerabilite critique** | Notification immediate par email ou telephone |
| **Compromission de donnees sensibles** | Alerte au contact designe dans les ROE |
| **Interruption de service involontaire** | Notification immediate + arret des tests sur le composant |
| **Fin de journee de test** | Statut quotidien (optionnel, selon le contrat) |
| **Blocage ou question de scope** | Contact du responsable designe |

{% hint style="warning" %}
Ne jamais attendre le rapport final pour signaler une vulnerabilite critique. Un RCE non authentifie sur un service expose, une base de donnees sans authentification accessible publiquement, ou des credentials admin par defaut sur un systeme de production doivent etre communiques dans l'heure.
{% endhint %}

### Redaction du rapport

Le rapport est le livrable principal de l'engagement. Il s'adresse a deux audiences : la direction (executive summary) et l'equipe technique (findings detailles).

#### Structure type du rapport

{% tabs %}
{% tab title="Executive Summary" %}
Redige pour un public non technique, generalement la direction ou le RSSI.

- **Contexte** : type de test, perimetre, dates, approche
- **Synthese des resultats** : nombre de vulnerabilites par severite, risque global
- **Scenarios d'impact** : description en termes metier de ce qu'un attaquant pourrait accomplir
- **Recommandations strategiques** : priorites a haut niveau, investissements necessaires
- **Classification du risque** : evaluation globale (critique/eleve/modere/faible)
{% endtab %}
{% tab title="Findings techniques" %}
Chaque vulnerabilite est documentee individuellement.

- **Titre** : nom clair et descriptif
- **Severite** : critique/elevee/moyenne/faible/informationnelle (CVSS si applicable)
- **Systemes affectes** : hotes, services, URLs concernes
- **Description** : explication de la vulnerabilite et de son impact
- **Preuves** : captures d'ecran annotees, commandes executees, reponses obtenues
- **Remediation** : recommandation specifique et actionnable
- **References** : CVE, CWE, OWASP, documentation editeur
{% endtab %}
{% tab title="Annexes" %}
Informations complementaires qui alourdiraient le corps du rapport.

- **Journal des modifications** : liste des changements apportes a l'environnement
- **Resultats de scans complets** : sortie Nmap, Nessus, BloodHound
- **Liste des credentials decouverts** (chiffree ou protegee)
- **Methodologie detaillee** : outils utilises, techniques employees
{% endtab %}
{% endtabs %}

#### Classification des severites

| Severite | Critere | Exemple |
|---|---|---|
| **Critique** | Compromission totale, acces sans authentification | RCE non authentifie, DCSync, credentials admin par defaut |
| **Elevee** | Acces significatif avec exploitation simple | SQLi avec extraction de donnees, escalade de privileges locale |
| **Moyenne** | Acces limite ou exploitation conditionnelle | XSS stocke, CSRF sur une fonction sensible |
| **Faible** | Impact minimal, exploitation difficile | Information disclosure mineure, headers de securite manquants |
| **Informationnelle** | Bonne pratique non respectee, sans impact direct | Version de logiciel dans les headers, absence de HSTS |

### Livraison et debriefing

#### Livraison du rapport

```
1. Rapport draft envoye au client pour relecture
2. Corrections et ajustements apres retour client
3. Rapport final signe et date
4. Transmission securisee (email chiffre, plateforme securisee)
5. Confirmation de reception par le client
```

{% hint style="info" %}
Le rapport contient des informations extremement sensibles (vulnerabilites actives, credentials, chemins d'acces). La transmission doit utiliser un canal securise : email chiffre (PGP/GPG), plateforme de partage avec authentification, ou remise en main propre sur support chiffre.
{% endhint %}

#### Debriefing

Le debriefing est une reunion de restitution avec le client. Il permet de presenter les resultats, de repondre aux questions et de discuter des priorites de remediation.

| Element du debriefing | Detail |
|---|---|
| **Participants** | Equipe technique du client, management, RSSI |
| **Presentation** | Walkthrough des findings les plus critiques |
| **Demonstration** | Rejeu en direct de certaines attaques (si possible et autorise) |
| **Discussion** | Prioritisation des remediations, faisabilite, planning |
| **Suivi** | Planification d'un retest apres remediation (optionnel) |

### Apres l'engagement

| Action | Objectif |
|---|---|
| **Archivage securise** | Conserver les notes et preuves de maniere chiffree |
| **Suppression des donnees client** | Selon la politique de retention et le contrat |
| **Retest** | Verifier les corrections apres remediation (engagement separe) |
| **Retour d'experience interne** | Identifier les points d'amelioration pour les prochains engagements |

## En pratique

### Checklist de fin d'engagement

```
[ ] Nettoyage : tous les artefacts supprimes de l'environnement
[ ] Journal : modifications documentees et communiquees
[ ] Captures : toutes les preuves sauvegardees localement
[ ] Rapport draft : soumis au client pour relecture
[ ] Rapport final : corrections integrees, rapport signe
[ ] Transmission : canal securise, confirmation de reception
[ ] Debriefing : presentation planifiee ou effectuee
[ ] Archivage : notes et preuves chiffrees localement
[ ] Donnees client : supprimees selon la politique de retention
```

## Pieges et galeres

- **Nettoyage incomplet** : un web shell oublie sur un serveur de production est une vulnerabilite que le pentester introduit lui-meme. Utiliser le journal de bord pour ne rien oublier
- **Rapport trop technique** : l'executive summary doit etre lisible par un non-technicien. Eviter les acronymes non expliques et les descriptions purement techniques
- **Preuves insuffisantes** : une capture d'ecran sans contexte ne prouve rien. Chaque preuve doit montrer clairement le systeme cible, la commande executee et le resultat obtenu
- **Pas de recommandation** : un finding sans remediation actionnable est frustrant pour l'equipe technique du client. Chaque finding doit proposer une solution concrete
- **Oublier le retest** : proposer un retest apres remediation est une bonne pratique qui renforce la relation client et garantit que les corrections sont effectives
- **Transmission non securisee** : envoyer un rapport de pentest en piece jointe d'un email non chiffre est une faute professionnelle. Toujours utiliser un canal securise

## Memo express

| Phase | Action | Detail |
|---|---|---|
| **Nettoyage** | Supprimer tous les artefacts | Comptes, cles, fichiers, SPN, persistence |
| **Communication** | Alertes immediates si critique | Ne pas attendre le rapport final |
| **Rapport** | Executive summary + findings detailles | Deux audiences, deux niveaux de detail |
| **Severites** | Critique > Elevee > Moyenne > Faible > Info | Basees sur l'impact et la facilite d'exploitation |
| **Livraison** | Canal securise, confirmation | Email chiffre ou plateforme dediee |
| **Debriefing** | Presentation et discussion | Walkthrough, prioritisation, planning retest |
| **Archivage** | Donnees chiffrees, retention limitee | Selon le contrat et la politique interne |

***
