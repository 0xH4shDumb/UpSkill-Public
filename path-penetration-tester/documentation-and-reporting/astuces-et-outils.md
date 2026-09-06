# Astuces et outils de reporting

Un bon processus de reporting repose sur des templates solides, de l'automatisation et une base de donnees de findings. Cette page couvre les outils de reporting, les techniques de productivite et les bonnes pratiques de communication client.

## Pourquoi

Rediger un rapport de zero a chaque engagement est un gaspillage de temps et une source d'incoherences. Des templates bien concus, une base de findings reutilisables et un processus de QA structure reduisent le temps de reporting et ameliorent la qualite du livrable.

## Comment ca marche

### Templates de rapport

Chaque type d'evaluation devrait avoir son propre template vierge. Le template inclut la mise en page, les sections recurrentes, les placeholders pour les informations specifiques au client et les styles predefinis.

| Element | Detail |
|---|---|
| **Un template par type d'evaluation** | Pentest interne, externe, applicatif, vuln assessment |
| **Toujours partir du template vierge** | Ne jamais modifier un ancien rapport client (risque de laisser des donnees) |
| **Placeholders** | Nom du client, dates, scope, IP source, nom du testeur |
| **Styles predefinis** | Titres, code, tableaux, captions, findings |

{% hint style="danger" %}
Reutiliser un rapport d'un precedent client en remplacant les noms est la source d'erreur la plus embarrassante en consulting. Un "ACME Corp" oublie dans un rapport destine a un autre client peut detruire la relation commerciale. Toujours partir d'un template vierge.
{% endhint %}

### Bases de donnees de findings

Apres plusieurs engagements, les memes vulnerabilites reviennent regulierement (LLMNR poisoning, Kerberoasting, default credentials, etc.). Maintenir une base de findings pre-rediges economise des heures de travail.

| Approche | Description |
|---|---|
| **Document dedie** | Fichier avec des findings pre-rediges, copies et adaptes pour chaque rapport |
| **Outil dedie** | Plateforme specialisee avec recherche, versioning et export |

#### Plateformes de reporting

{% tabs %}
{% tab title="Gratuites / Open-source" %}
| Outil | Description |
|---|---|
| **Ghostwriter** | Plateforme complete de gestion d'engagements et de reporting |
| **Dradis** | Framework collaboratif de reporting (version Community) |
| **VECTR** | Outil de suivi des exercises de securite (Security Risk Advisors) |
| **WriteHat** | Outil de reporting web avec templates integres |
| **PwnDoc** | Generateur de rapports de pentest avec base de findings |
{% endtab %}
{% tab title="Payantes" %}
| Outil | Description |
|---|---|
| **AttackForge** | Plateforme SaaS de gestion de pentest et de reporting |
| **PlexTrac** | Plateforme de reporting et de gestion de la securite offensive |
| **Rootshell Prism** | Outil de gestion des vulnerabilites et de reporting |
{% endtab %}
{% endtabs %}

### Automatisation

L'automatisation reduit les taches repetitives et limite les erreurs humaines.

| Tache automatisable | Methode |
|---|---|
| Insertion du nom du client, des dates, du scope | Macros Word avec placeholders |
| Numerotation des findings | Numerotation automatique Word |
| Table des matieres et table des figures | Champs automatiques Word |
| Suppression de sections non pertinentes | Macros avec bookmarks (par exemple, supprimer la section OSINT pour un test interne) |
| Verification orthographique ciblée | Dictionnaire personnalise pour ignorer les termes techniques |

{% hint style="info" %}
Pour les macros Word, sauvegarder les templates en `.dotm` (macro-enabled template). La creation de macros avancees necessite l'editeur VB (Visual Basic), disponible uniquement sur Word pour Windows.
{% endhint %}

### Processus de QA (Quality Assurance)

Chaque rapport devrait passer par au minimum un cycle de relecture, idealement deux relecteurs differents du redacteur.

| Etape | Verificateur | Points a verifier |
|---|---|---|
| **Auto-relecture** | Le redacteur | Coherence, orthographe, captures lisibles |
| **QA technique** | Un pair technique | Exactitude des findings, reproductibilite, severites |
| **QA editoriale** | Un non-technique si possible | Clarte de l'executive summary, langage accessible |

{% hint style="success" %}
Laisser reposer le rapport une nuit avant la relecture. Apres des heures de redaction, le cerveau ne voit plus les erreurs. Une relecture a froid le lendemain revele souvent des incoherences invisibles la veille.
{% endhint %}

### Communication client

La communication avec le client s'etend du cadrage a la livraison du rapport final.

#### Notifications pendant l'engagement

| Notification | Moment | Contenu |
|---|---|---|
| **Notification de debut** | Premier jour du test | Nom du testeur, type de test, IP source, dates |
| **Alerte finding critique** | Des la decouverte | Description de la vulnerabilite, impact immediat, recommandation d'urgence |
| **Notification de fin** | Dernier jour du test | Confirmation de la fin des activites, delai de livraison du rapport |
| **Livraison du rapport** | Selon le contrat | Rapport draft, puis final apres retours client |

{% hint style="danger" %}
Les findings critiques (RCE, acces non authentifie a des donnees sensibles) doivent etre communiques au client immediatement, pas dans le rapport final. Un email ou un appel telephonique le jour de la decouverte est la norme.
{% endhint %}

#### Debriefing

Le debriefing est la reunion ou le pentester presente les resultats au client. C'est le moment de contextualiser les findings, repondre aux questions et discuter des priorites de remediation.

| Bonne pratique | Detail |
|---|---|
| Adapter le discours au public | Technique pour l'equipe IT, strategique pour la direction |
| Commencer par les points positifs | Ce que le client fait bien avant de lister les problemes |
| Proposer un plan de remediation priorise | Les corrections les plus impactantes en premier |
| Rester ouvert aux retours | Le client peut avoir du contexte supplementaire qui change la severite |

## Pieges et galeres

- **Template perime** : un template qui n'est pas mis a jour avec les nouvelles sections, les nouveaux styles ou les retours des precedents QA perd son efficacite. Reviser les templates apres chaque engagement
- **Base de findings non maintenue** : des findings rediges il y a deux ans avec des recommandations obsoletes font mauvaise impression. Mettre a jour la base regulierement
- **QA par le redacteur** : relire son propre travail ne suffit jamais. Le cerveau complete automatiquement les phrases manquantes et ignore les erreurs familierees
- **Finding critique non signale immediatement** : attendre le rapport final pour communiquer une RCE expose le client pendant des jours ou des semaines
- **Sorties d'outils non nettoyees** : des termes comme "Pwn3d!" dans une sortie CrackMapExec ou des mots grossiers dans une wordlist Hashcat n'ont pas leur place dans un document professionnel. Nettoyer les sorties avant de les inclure

## Memo express

| Element | Detail |
|---|---|
| **Templates** | Un par type d'evaluation, toujours vierge, avec placeholders |
| **Base de findings** | Findings pre-rediges, adaptes a chaque client |
| **Outils gratuits** | Ghostwriter, Dradis, WriteHat, PwnDoc |
| **QA** | Minimum 1 relecture par un pair, idealement 2 |
| **Finding critique** | Communiquer immediatement au client, pas dans le rapport final |
| **Debriefing** | Adapter au public, commencer par les points positifs |

***
