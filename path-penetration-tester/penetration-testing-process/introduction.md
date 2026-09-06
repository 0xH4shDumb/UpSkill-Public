# Processus de test d'intrusion

Un test d'intrusion est une evaluation methodique de la securite d'un systeme, d'un reseau ou d'une application. L'objectif est de decouvrir l'ensemble des vulnerabilites exploitables et de demontrer leur impact reel, pour permettre a l'organisation de renforcer sa posture de securite.

## Pourquoi

Sans processus structure, un pentest devient un exercice desordonne ou l'on risque de passer a cote de vecteurs d'attaque entiers. Les phases definies ci-dessous garantissent une couverture complete, de la negociation du perimetre a la remise du rapport final. Chaque phase produit des livrables qui alimentent la suivante.

## Comment ca marche

### Types de tests

La quantite d'informations fournies au pentester conditionne l'approche et le realisme de l'evaluation.

| Type | Informations fournies | Realisme |
|---|---|---|
| **Boite noire** | Minimum (adresse IP ou nom de domaine) | Simule un attaquant externe sans connaissance prealable |
| **Boite grise** | Intermediaire (sous-reseaux, URLs, comptes utilisateur) | Simule un attaquant avec un acces initial ou des informations partielles |
| **Boite blanche** | Complet (code source, architecture, credentials admin) | Permet une couverture exhaustive des vulnerabilites |
| **Red Teaming** | Variable, souvent minimal | Simulation d'attaque realiste incluant social engineering et tests physiques |
| **Purple Teaming** | Collaboration Red/Blue Team | Amelioration conjointe des capacites de detection et de reponse |

{% hint style="info" %}
Le choix du type de test depend des objectifs du client. Une boite noire teste la resilience face a un attaquant externe. Une boite blanche maximise la decouverte de vulnerabilites. Le Red Teaming evalue la capacite de detection globale de l'organisation.
{% endhint %}

### Les 8 phases du processus

#### 1. Pre-engagement

La phase de pre-engagement pose le cadre legal et operationnel de l'intervention. Elle inclut la signature des documents contractuels et la definition precise du perimetre.

| Document | Moment de creation | Contenu |
|---|---|---|
| **NDA** (accord de confidentialite) | Apres le premier contact | Protection des informations echangees (unilateral, bilateral ou multilateral) |
| **Questionnaire de cadrage** | Avant la reunion de cadrage | Inventaire des actifs, contraintes, horaires, contacts d'urgence |
| **Scoping Document / SoW** | Pendant la reunion de cadrage | Perimetre detaille, exclusions, methodologie, livrables attendus |
| **Rules of Engagement** | Avant le demarrage | Regles d'engagement : techniques autorisees, limites, procedures d'escalade |
| **Rapport final** | Pendant et apres le test | Vulnerabilites, preuves, recommandations, synthese pour la direction |

{% hint style="warning" %}
Ne jamais commencer un test sans un document de perimetre signe et des regles d'engagement claires. Un test d'intrusion sans autorisation ecrite est une intrusion illegale, quel que soit l'objectif.
{% endhint %}

#### 2. Collecte d'informations

L'enumeration initiale vise a cartographier la surface d'attaque. Elle combine des techniques passives (OSINT) et actives (scans reseau).

- Recherche OSINT (sources ouvertes, reseaux sociaux, bases de donnees publiques)
- Enumeration reseau (infrastructure, services exposes, hotes)
- Identification des technologies (frameworks, versions, pile applicative)
- Comprehension du fonctionnement des applications cibles

#### 3. Evaluation des vulnerabilites

L'evaluation transforme les donnees collectees en une liste de vulnerabilites classees par criticite. Quatre types d'analyse interviennent.

| Type d'analyse | Objectif |
|---|---|
| **Descriptive** | Decrire les caracteristiques des systemes observes |
| **Diagnostic** | Identifier les causes et consequences des problemes trouves |
| **Predictive** | Anticiper les risques a partir des donnees historiques et des tendances |
| **Prescriptive** | Proposer des actions correctives ou preventives |

#### 4. Exploitation

L'exploitation consiste a valider les vulnerabilites identifiees en les exploitant effectivement. Les vecteurs sont priorises selon leur taux de succes estime, leur impact potentiel et leur difficulte.

- Execution de code a distance ou locale
- Escalade de privileges
- Exfiltration de donnees de demonstration

#### 5. Post-exploitation

Apres la compromission initiale, la post-exploitation determine l'impact reel de l'acces obtenu.

1. Evaluation de la furtivite (est-ce que l'acces a ete detecte ?)
2. Collecte d'informations internes (credentials, configurations, donnees sensibles)
3. Identification de nouvelles vulnerabilites depuis la position compromise
4. Maintien de la persistance (si autorise par les regles d'engagement)
5. Exfiltration de donnees de demonstration

#### 6. Mouvement lateral

Si le perimetre inclut plusieurs systemes, le mouvement lateral permet de demontrer l'impact reseau global d'une compromission initiale.

- Reconnaissance interne (reseaux accessibles, services, partages)
- Reutilisation de credentials extraits
- Exploitation de relations de confiance entre systemes
- Acces a des ressources critiques (controleurs de domaine, bases de donnees, sauvegardes)

#### 7. Preuve de concept

Chaque vulnerabilite exploitee doit etre documentee avec une preuve de concept reproductible. La PoC doit etre suffisamment claire pour que l'equipe de remediation puisse comprendre le vecteur, reproduire le probleme et verifier la correction.

#### 8. Post-engagement

La phase finale inclut la production et la livraison des livrables.

1. Analyse des donnees collectees et correlation des resultats
2. Redaction du rapport (technique + synthese pour la direction)
3. Evaluation des risques et classification des vulnerabilites
4. Debriefing avec le client
5. Recommandations de remediation priorisees
6. Re-test des vulnerabilites corrigees (si prevu au contrat)
7. Documentation complete et archivage securise
8. Suivi post-mission

## Pieges et galeres

- **Perimetre flou** : sans document de cadrage precis, le pentester risque de tester des systemes hors perimetre. Toujours obtenir une liste d'actifs signee
- **Regles d'engagement insuffisantes** : des regles vagues ("testez tout") peuvent mener a des incidents. Definir les techniques interdites, les plages horaires et les contacts d'urgence
- **Documentation pendant le test** : prendre des notes et des captures au fil de l'eau. Reconstituer les etapes apres coup est chronophage et source d'erreurs
- **Rapport sans contexte business** : un rapport purement technique sans synthese pour la direction perd une partie de son impact. Toujours inclure un resume executif
- **Re-test oublie** : verifier que le contrat prevoit un re-test pour valider les corrections. Sans re-test, le client ne sait pas si ses corrections sont efficaces

## Memo express

| Phase | Livrable principal |
|---|---|
| Pre-engagement | NDA, Scoping Document, Rules of Engagement |
| Collecte d'informations | Cartographie de la surface d'attaque |
| Evaluation des vulnerabilites | Liste de vulnerabilites classees |
| Exploitation | Preuves d'exploitation (screenshots, shells) |
| Post-exploitation | Donnees sensibles, chemins de compromission |
| Mouvement lateral | Cartographie des systemes compromis |
| Preuve de concept | PoC reproductibles pour chaque vulnerabilite |
| Post-engagement | Rapport final, debriefing, re-test |

***
