# Cadre juridique et reglementaire

Le test d'intrusion evolue dans un cadre juridique strict. Chaque pays dispose de lois specifiques qui encadrent les acces aux systemes informatiques, la protection des donnees et la lutte contre la cybercriminalite. Un pentester doit connaitre les grandes lignes de ce cadre pour exercer legalement et conseiller ses clients sur leur conformite.

## Pourquoi

Un test d'intrusion sans autorisation ecrite est une infraction penale dans la quasi-totalite des juridictions. Meme avec une autorisation, certaines techniques (interception de communications, acces a des donnees de sante) peuvent entrer en conflit avec des reglementations sectorielles. Connaitre le cadre juridique protege le pentester et le client.

## Comment ca marche

### Grandes categories de legislation

Les lois relatives a la cybersecurite se repartissent en plusieurs categories qui s'appliquent differemment selon le pays.

| Categorie | Objet |
|---|---|
| **Protection des donnees** | Encadrement de la collecte, du traitement et du stockage des donnees personnelles |
| **Acces non autorises** | Criminalisation de l'intrusion dans les systemes informatiques |
| **Interception des communications** | Regulation de la surveillance electronique et de l'ecoute |
| **Propriete intellectuelle numerique** | Protection des droits d'auteur sur le contenu numerique |
| **Donnees sectorielles** | Reglementations specifiques (sante, finance, defense) |
| **Cooperation internationale** | Conventions et traites pour la lutte transfrontaliere contre la cybercriminalite |

### Principales lois par juridiction

{% tabs %}
{% tab title="Europe" %}
| Loi | Objet |
|---|---|
| **RGPD** (Reglement General sur la Protection des Donnees) | Protection des donnees personnelles des citoyens europeens. S'applique a toute organisation qui traite des donnees de residents UE, quel que soit son lieu d'etablissement |
| **NIS 2** (Network and Information Systems Directive) | Securite des reseaux et systemes d'information, obligations pour les operateurs de services essentiels |
| **Convention de Budapest** | Cooperation judiciaire internationale en matiere de cybercriminalite |
| **e-Privacy Directive** (2002/58/EC) | Protection de la vie privee dans les communications electroniques |
| **Charte des droits fondamentaux de l'UE** | Droits fondamentaux incluant la protection des donnees personnelles |
{% endtab %}
{% tab title="Etats-Unis" %}
| Loi | Objet |
|---|---|
| **CFAA** (Computer Fraud and Abuse Act) | Criminalisation des acces non autorises aux systemes informatiques. Loi de reference pour les poursuites liees aux intrusions |
| **CISA** (Cybersecurity Information Sharing Act) | Partage d'informations sur les menaces entre le secteur prive et le gouvernement |
| **HIPAA** (Health Insurance Portability and Accountability Act) | Protection des donnees de sante |
| **DMCA** (Digital Millennium Copyright Act) | Protection des droits d'auteur numeriques |
| **ECPA** (Electronic Communications Privacy Act) | Encadrement de l'interception des communications electroniques |
| **COPPA** (Children's Online Privacy Protection Act) | Protection des donnees des enfants en ligne |
{% endtab %}
{% tab title="Royaume-Uni" %}
| Loi | Objet |
|---|---|
| **Computer Misuse Act 1990** | Criminalisation des acces non autorises et des modifications non autorisees de donnees |
| **Data Protection Act 2018** | Transposition du RGPD en droit britannique (post-Brexit) |
| **Investigatory Powers Act 2016** (IPA) | Encadrement des pouvoirs de surveillance electronique |
| **Human Rights Act 1998** | Protection des droits fondamentaux incluant la vie privee |
| **Police and Justice Act 2006** | Dispositions complementaires sur la cybercriminalite |
{% endtab %}
{% tab title="Autres" %}
| Pays | Loi principale | Objet |
|---|---|---|
| **Inde** | Information Technology Act 2000 | Cadre general pour la cybersecurite et les transactions electroniques |
| **Inde** | Personal Data Protection Bill 2019 | Protection des donnees personnelles (en cours d'adoption) |
| **Chine** | Cyber Security Law | Obligations de securite pour les operateurs de reseaux et protection des donnees |
| **Chine** | National Security Law | Cadre de securite nationale incluant le cyberespace |
{% endtab %}
{% endtabs %}

### Impact pour le pentester

| Situation | Risque juridique | Precaution |
|---|---|---|
| Test sans autorisation ecrite | Poursuites penales (CFAA, Computer Misuse Act, etc.) | Obtenir un contrat signe avec le perimetre exact |
| Acces a des donnees de sante | Violation de HIPAA / RGPD | Definir des regles de manipulation des donnees dans les Rules of Engagement |
| Test transfrontalier | Conflit de juridictions | Verifier la legislation de chaque pays ou les systemes sont heberges |
| Interception de trafic | Violation des lois sur l'ecoute | S'assurer que les Rules of Engagement autorisent explicitement la capture de trafic |
| Decouverte de donnees illegales | Obligation de signalement selon la juridiction | Prevoir une procedure d'escalade dans le contrat |

{% hint style="danger" %}
Le RGPD s'applique des qu'un systeme traite des donnees de residents europeens, meme si l'organisation est basee hors de l'UE. Un pentester qui exfiltre des donnees personnelles comme preuve de concept doit s'assurer que le traitement est couvert par le contrat et les Rules of Engagement.
{% endhint %}

## Pieges et galeres

- **CFAA et portee etendue** : aux Etats-Unis, le CFAA est largement interprete. Des actions qui semblent anodines (acces a une page web non protegee mais "non destinee au public") ont fait l'objet de poursuites
- **Tests multi-pays** : un test qui touche des systemes dans plusieurs juridictions peut etre legal dans un pays et illegal dans un autre. Verifier systematiquement
- **Cloud et hebergement** : les systemes heberges dans le cloud peuvent etre physiquement situes dans un pays different de celui du client. Les regles du pays d'hebergement s'appliquent aussi
- **Donnees sensibles decouvertes** : la decouverte de contenus illegaux (pedopornographie, activites terroristes) pendant un test impose des obligations de signalement qui varient selon la juridiction
- **Sous-traitance** : si le pentester sous-traite une partie du test, le sous-traitant doit etre couvert par les memes autorisations et accords de confidentialite

## Memo express

| Element | Detail |
|---|---|
| **Document indispensable** | Contrat signe avec perimetre, Rules of Engagement, NDA |
| **Europe** | RGPD (donnees personnelles), NIS 2 (services essentiels), Convention de Budapest |
| **Etats-Unis** | CFAA (acces non autorises), HIPAA (sante), CISA (partage d'informations) |
| **Royaume-Uni** | Computer Misuse Act 1990, Data Protection Act 2018 |
| **Risque principal** | Test sans autorisation = infraction penale |
| **Tests transfrontaliers** | Verifier la legislation de chaque juridiction concernee |

***
