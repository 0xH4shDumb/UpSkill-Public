# Pentest en entreprise

Un test d'intrusion en entreprise combine toutes les competences acquises au fil du parcours : reconnaissance, exploitation web, escalade de privileges, mouvement lateral et compromission Active Directory. Ce module aborde le pentest comme un engagement complet, de la reunion de cadrage a la livraison du rapport.

## Pourquoi

Les modules precedents traitent chaque technique de maniere isolee. En situation reelle, un pentester doit enchainer ces techniques de facon fluide, adapter sa strategie en fonction des resultats, gerer les impasses et documenter chaque etape. L'objectif est de simuler un engagement professionnel complet pour developper cette capacite d'adaptation.

## Comment ca marche

### Les phases d'un engagement complet

Un test d'intrusion en entreprise suit les huit phases du processus standard, mais les enchaine dans un scenario realiste.

| Phase | Objectif | Livrable |
|---|---|---|
| **Pre-engagement** | Cadrage, signatures, scope, ROE | Documents contractuels signes |
| **Reconnaissance externe** | Cartographier la surface d'attaque publique | Sous-domaines, services, technologies |
| **Exploitation externe** | Obtenir un point d'entree dans le reseau interne | Acces initial (shell, credentials) |
| **Pivoting** | Acceder au reseau interne depuis la DMZ | Tunnel SSH, proxy SOCKS |
| **Reconnaissance interne** | Enumerer le reseau interne et l'Active Directory | Hotes, services, utilisateurs, ACL |
| **Mouvement lateral** | Progresser vers des cibles de valeur | Acces a de nouveaux systemes |
| **Compromission AD** | Obtenir un acces Domain Admin | DCSync, Golden Ticket |
| **Post-engagement** | Nettoyage, rapport, debriefing | Rapport final |

### Pre-engagement : ce qui doit etre pret

Avant de lancer le moindre scan, verifier que tous les documents sont signes et que le perimetre est clairement defini.

| Document | Contenu |
|---|---|
| **Scope of Work (SoW)** | Perimetre, methodologie, planning, livrables |
| **Rules of Engagement (ROE)** | Techniques autorisees, plages horaires, contacts d'urgence, exclusions |
| **NDA** | Confidentialite des donnees decouvertes |
| **Scope detaille** | Plages IP, domaines, sous-domaines, credentials fournis |

{% hint style="danger" %}
Ne jamais commencer les tests sans un document de scope signe avec les exclusions explicites. Les elements generalement exclus sont le social engineering (sauf accord specifique), les attaques physiques, le deni de service et toute modification destructive de l'environnement.
{% endhint %}

### Notification de debut

Le premier jour de test, envoyer un email de notification aux contacts definis dans le SoW. Cet email doit contenir le nom du testeur, le type de test, l'adresse IP source utilisee, les dates de test et les coordonnees du contact de secours.

### Approche typique

{% tabs %}
{% tab title="Boite noire" %}
Le pentester n'a aucune information prealable en dehors du nom de domaine ou des plages IP. Il doit decouvrir la surface d'attaque par ses propres moyens (OSINT, enumeration DNS, scans). C'est l'approche la plus realiste pour simuler un attaquant externe.
{% endtab %}
{% tab title="Boite grise" %}
Le pentester recoit des informations partielles : plages IP, sous-domaines connus, eventuellement des credentials utilisateur standard. C'est l'approche la plus courante en entreprise, car elle optimise le temps de test.
{% endtab %}
{% tab title="Boite blanche" %}
Le pentester recoit un acces complet : code source, architecture, credentials admin. L'objectif est une couverture exhaustive des vulnerabilites plutot qu'une simulation realiste d'attaque.
{% endtab %}
{% endtabs %}

### Gestion du temps

Un engagement typique dure entre une et quatre semaines. La repartition du temps varie, mais un schema courant est le suivant.

| Phase | Proportion du temps |
|---|---|
| Reconnaissance et enumeration | 30-40% |
| Exploitation et mouvement lateral | 30-40% |
| Documentation et reporting | 20-30% |

{% hint style="info" %}
Les scans lourds (Nessus, Nmap complet) doivent etre planifies en dehors des heures de production si le client le demande. Les ROE precisent generalement les plages horaires autorisees pour les tests intensifs.
{% endhint %}

## Pieges et galeres

- **Scope incomplet** : un scope qui mentionne "tout le reseau" sans lister les exclusions peut mener a des incidents. Toujours obtenir une liste d'IP/sous-reseaux explicite
- **Tunnel vision** : se focaliser sur un seul vecteur d'attaque pendant des heures. Si un chemin est bloque, revenir a l'enumeration et chercher une autre voie
- **Documentation negligee** : en pleine exploitation, la tentation est forte de reporter la documentation. Chaque etape non documentee est une etape perdue pour le rapport
- **Oublier le nettoyage** : chaque fichier depose, chaque compte modifie, chaque configuration changee doit etre note et restaure en fin d'engagement
- **Communication insuffisante** : les findings critiques doivent etre communiques immediatement au client, pas dans le rapport final

## Memo express

| Element | Detail |
|---|---|
| **Documents prealables** | SoW, ROE, NDA, scope detaille (tous signes) |
| **Notification** | Email de debut avec IP source, dates, contacts |
| **Approche** | Boite noire (realiste), grise (equilibree), blanche (exhaustive) |
| **Temps** | 30-40% enumeration, 30-40% exploitation, 20-30% reporting |
| **Communication** | Finding critique = alerte immediate au client |
| **Nettoyage** | Tout artefact depose doit etre supprime et documente |

***
