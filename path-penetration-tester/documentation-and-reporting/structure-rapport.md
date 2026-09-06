# Structure et types de rapports

Tous les rapports de pentest ne se ressemblent pas. Un rapport d'evaluation de vulnerabilites n'a pas la meme structure qu'un rapport de test d'intrusion interne avec compromission du domaine Active Directory. Cette page couvre les differents types de rapports, leurs composants et la maniere de structurer un livrable professionnel.

## Pourquoi

Produire un rapport dont la structure ne correspond pas au type d'evaluation realisee desoriente le lecteur et diminue l'impact des conclusions. Connaitre les differents formats et leurs composants permet d'adapter le livrable au contexte et de repondre aux attentes du client.

## Comment ca marche

### Types d'evaluations et leurs rapports

| Type d'evaluation | Perspective | Exploitation | Specificite du rapport |
|---|---|---|---|
| **Evaluation de vulnerabilites** | Interne ou externe | Non (validation seulement) | Focus sur les themes et la severite des vulnerabilites |
| **Test d'intrusion externe** | Attaquant anonyme depuis internet | Oui | Inclut les donnees OSINT, la surface d'attaque publique |
| **Test d'intrusion interne** | Utilisateur sur le reseau interne | Oui | Chaine d'attaque, compromission AD, mouvement lateral |
| **Red Team** | Simulation d'attaquant avance | Oui, evasif | Focus sur la detection et la reponse de la Blue Team |
| **Purple Team** | Collaboration Red/Blue Team | Oui, cooperative | Evaluation conjointe de la detection et des regles d'alerte |
| **Test applicatif** | Utilisateur de l'application | Oui | Vulnerabilites OWASP, logique metier, infrastructure sous-jacente |

{% hint style="info" %}
Un meme test peut combiner plusieurs perspectives. Un test d'intrusion externe qui aboutit a une compromission interne contiendra des elements des deux types de rapports (surface externe + chaine d'attaque interne).
{% endhint %}

### Cycle de vie du rapport

Le rapport passe par plusieurs etapes avant la livraison finale.

1. **Brouillon (draft)** : premiere version soumise au client pour commentaires
2. **Revue client** : le client propose des modifications (langage, contexte, reponses de la direction)
3. **Rapport final** : version definitive integrant les retours du client
4. **Rapport post-remediation** : re-test cible des vulnerabilites corrigees (si prevu au contrat)

{% hint style="warning" %}
Le rapport post-remediation se limite aux findings du rapport initial sur les hotes initialement affectes. Ce n'est pas un nouveau test complet. Etendre le perimetre risque de decouvrir de nouvelles vulnerabilites et de creer une boucle sans fin.
{% endhint %}

### Composants d'un rapport de pentest

#### 1. Page de garde

Nom du client, type d'evaluation, dates de test, nom du prestataire, classification du document (confidentiel).

#### 2. Synthese pour la direction (Executive Summary)

Resume non technique destine aux decideurs. Il repond a trois questions : quel est l'etat global de la securite, quels sont les risques majeurs, quelles actions prioritaires sont recommandees.

| A inclure | A eviter |
|---|---|
| Niveau de risque global | Jargon technique |
| Nombre de findings par severite | Sorties de commandes |
| Recommandations strategiques | Details d'exploitation |
| Comparaison avec les standards du secteur | Acronymes non expliques |

#### 3. Perimetre et methodologie

Description du perimetre teste (plages IP, applications, exclusions), type de test (boite noire/grise/blanche), dates, adresses IP source, outils utilises, referentiels suivis (OWASP, PTES, OSSTMM).

#### 4. Chaine d'attaque (Attack Chain)

Section narrative qui illustre le chemin complet de la compromission. Elle relie les findings individuels pour montrer comment des vulnerabilites de severite moyenne, combinees, permettent une compromission totale.

La chaine d'attaque doit inclure :

- Un resume en quelques phrases du chemin global
- Chaque etape numerotee avec la technique utilisee
- Les captures d'ecran et sorties de commandes associees
- L'impact de chaque etape sur la progression

{% hint style="success" %}
La chaine d'attaque est l'element le plus impactant du rapport. Elle transforme une liste de vulnerabilites en un recit concret qui parle aussi bien a la direction qu'aux equipes techniques. Les preuves utilisees ici peuvent etre reutilisees dans les findings individuels.
{% endhint %}

#### 5. Findings

Section detaillee des vulnerabilites. Chaque finding est documente individuellement avec une structure standardisee (voir la page suivante).

#### 6. Annexes

Informations complementaires qui alourdiraient le corps du rapport : resultats de scans complets, listes de comptes compromis, configurations detaillees, glossaire.

### Classification des severites

| Severite | Critere |
|---|---|
| **Critique** | Exploitation immediate possible, impact majeur (RCE, compromission du domaine) |
| **Haute** | Exploitation probable, impact significatif (acces non autorise, exfiltration de donnees) |
| **Moyenne** | Exploitation conditionnelle ou impact modere (information disclosure, misconfiguration) |
| **Basse** | Impact faible, exploitation difficile (headers manquants, versions obsoletes sans exploit connu) |
| **Informationnelle** | Pas d'exploitation directe, observation utile pour le durcissement |

## En pratique

### Structure type d'un rapport de test d'intrusion interne

```
1. Page de garde
2. Table des matieres
3. Synthese pour la direction
4. Perimetre et methodologie
   4.1 Perimetre
   4.2 Dates et duree
   4.3 Methodologie et referentiels
   4.4 Adresses IP source
5. Chaine d'attaque
6. Findings
   6.1 Finding critique : [nom]
   6.2 Finding haute : [nom]
   ...
7. Annexes
   A. Resultats de scans
   B. Comptes compromis
   C. Glossaire
```

### Exemple de synthese pour la direction

> Durant la periode de test (du 7 au 18 janvier 2025), l'equipe a identifie 14 vulnerabilites sur le perimetre interne de l'organisation. Parmi celles-ci, 3 sont classees critiques et ont permis la compromission complete du domaine Active Directory en moins de 4 heures depuis un poste utilisateur standard. Les principales faiblesses identifiees concernent la gestion des mots de passe des comptes de service et l'absence de segmentation reseau entre les postes utilisateurs et les serveurs critiques.

## Pieges et galeres

- **Rapport sans executive summary** : un rapport purement technique sans synthese pour la direction perd la moitie de son audience. Les decideurs ne liront pas les details techniques
- **Chaine d'attaque absente** : une liste de findings sans chaine d'attaque ne montre pas l'impact reel. Le client ne comprend pas comment des failles "moyennes" menent a une compromission totale
- **Rapport post-remediation sans cadrage temporel** : si le re-test est realise plusieurs mois apres l'evaluation initiale, l'environnement a change et la comparaison devient impossible. Cadrer les delais dans le contrat
- **Severity inflation** : gonfler les severites pour impressionner le client nuit a la credibilite. Etre honnete et precis sur l'impact reel de chaque vulnerabilite
- **Copier-coller entre clients** : reutiliser un rapport sans le nettoyer risque de laisser le nom d'un autre client dans le document. Toujours partir d'un template vierge

## Memo express

| Element | Detail |
|---|---|
| **Types de rapports** | Vuln assessment, pentest (interne/externe), red team, purple team, applicatif |
| **Cycle** | Draft, revue client, rapport final, post-remediation |
| **Executive summary** | Non technique, risques majeurs, recommandations strategiques |
| **Chaine d'attaque** | Recit de la compromission etape par etape avec preuves |
| **Severites** | Critique, haute, moyenne, basse, informationnelle |
| **Template** | Toujours partir d'un template vierge, jamais d'un ancien rapport |

***
