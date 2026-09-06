# Redaction des findings

Les findings sont le coeur du rapport. Chaque vulnerabilite decouverte doit etre documentee de maniere a etre comprise, reproductible et actionnable. Cette page couvre la structure d'un finding, la presentation des preuves, les recommandations de remediation et les erreurs courantes.

## Pourquoi

Un finding mal redige ne sera pas corrige. Si l'equipe technique du client ne comprend pas la vulnerabilite, ne peut pas la reproduire ou ne sait pas comment la corriger, le rapport n'a pas rempli son objectif. La qualite de la redaction des findings determine directement l'impact du pentest.

## Comment ca marche

### Structure d'un finding

Chaque finding doit contenir au minimum les elements suivants.

| Element | Description |
|---|---|
| **Titre** | Nom clair et descriptif de la vulnerabilite |
| **Severite** | Critique, haute, moyenne, basse ou informationnelle |
| **CVSS** | Score optionnel mais recommande pour les clients soumis a la conformite |
| **Description** | Explication de la vulnerabilite, de sa cause et de son contexte |
| **Impact** | Consequences concretes si la vulnerabilite n'est pas corrigee |
| **Systemes affectes** | Liste des hotes, applications ou environnements concernes |
| **Etapes de reproduction** | Commandes et actions pour reproduire la vulnerabilite |
| **Preuves** | Captures d'ecran, sorties de commandes, extraits de configuration |
| **Recommandation** | Actions correctives specifiques et actionnables |
| **References** | Liens vers la documentation officielle, CVE, articles techniques |

Des champs supplementaires peuvent etre ajoutes selon le contexte.

- Identifiant CVE
- Identifiants OWASP, MITRE ATT&CK
- Facilite d'exploitation
- Probabilite d'attaque

### Presenter les preuves efficacement

Les preuves doivent etre claires, contextualisees et defensibles. Quelques regles essentielles.

{% tabs %}
{% tab title="Bonnes pratiques" %}
- **Une etape par capture** : ne pas combiner plusieurs actions dans une seule image. Le lecteur non specialiste ne saura pas ce qu'il regarde
- **Montrer la configuration avant l'exploitation** : pour un module Metasploit, capturer la configuration complete (`show options`) puis le resultat dans une seconde capture
- **Ecrire un narratif entre les captures** : decrire ce qui se passe et pourquoi, pas juste enchainer les images
- **Proposer des outils alternatifs** : mentionner d'autres outils capables de reproduire la vulnerabilite (juste le nom et une reference, pas une seconde demonstration)
{% endtab %}
{% tab title="Erreurs courantes" %}
- Capture sans URL ni prompt visible (aucune preuve du systeme cible)
- Terminal transparent montrant le bureau personnel
- Prompt non professionnel (eviter les noms provocants)
- Donnees sensibles non masquees (mots de passe en clair, hashes non rediges)
- 50 pages de sortie console copiees sans mise en contexte
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Une preuve doit etre incontestable. Par exemple, pour demontrer une transmission en clair de credentials via basic auth, un screenshot du popup de connexion ne suffit pas. Il faut montrer les credentials dans un packet capture (Wireshark) pour prouver la transmission non chiffree.
{% endhint %}

### Rediger des recommandations actionnables

La qualite des recommandations est ce qui differencie un rapport moyen d'un rapport excellent. Le client doit savoir exactement quoi faire.

{% tabs %}
{% tab title="Mauvaise recommandation" %}
"Reconfigurez les parametres du registre pour durcir le systeme contre cette attaque."

Probleme : vague, pas actionnable, le client doit chercher lui-meme quels parametres modifier.
{% endtab %}
{% tab title="Bonne recommandation" %}
"Pour corriger cette vulnerabilite, modifier les cles de registre suivantes :

`HKLM\System\CurrentControlSet\Control\Lsa\` : definir la valeur `RestrictAnonymous` a `1`.

Attention : les modifications du registre doivent etre testees sur un groupe pilote avant un deploiement large, car elles peuvent affecter la compatibilite avec certaines applications."

Points forts : specifique, actionnable, inclut un avertissement sur les risques de la remediation.
{% endtab %}
{% endtabs %}

| Bonne pratique | Mauvaise pratique |
|---|---|
| Donner les cles de registre exactes, les commandes, les parametres | "Durcissez le systeme" sans details |
| Proposer une alternative gratuite si un outil commercial est mentionne | Recommander uniquement un outil payant |
| Mentionner les effets secondaires potentiels de la remediation | Laisser le client decouvrir que la correction casse une application |
| Proposer des mesures palliatives en attendant la correction definitive | Ne donner qu'une seule option de remediation |

### Choisir des references pertinentes

Chaque finding doit etre accompagne de references externes de qualite.

| Critere | Explication |
|---|---|
| **Source neutre** | Privilegier les sources independantes (NIST, CIS, OWASP) plutot que les pages marketing d'un vendeur |
| **Contenu accessible** | Eviter les articles derriere un paywall ou les documents de 500 pages sans index |
| **Directement applicable** | La reference doit expliquer la vulnerabilite et/ou sa correction, pas juste la mentionner |
| **Perenne** | Privilegier les sites institutionnels qui ne disparaitront pas dans six mois |

{% hint style="success" %}
Developper ses propres articles de reference (blog technique, documentation interne) a un double avantage : la recherche approfondit la comprehension de la vulnerabilite, et le client reste sur les ressources du prestataire plutot que celles d'un concurrent.
{% endhint %}

## En pratique

### Exemple de finding : empoisonnement LLMNR/NBT-NS

**Titre** : Empoisonnement LLMNR/NBT-NS

**Severite** : Haute

**Description** : Les protocoles LLMNR (Link-Local Multicast Name Resolution) et NBT-NS (NetBIOS Name Service) sont actifs sur le reseau interne. Ces protocoles de resolution de noms en broadcast peuvent etre empoisonnes par un attaquant positionne sur le meme segment reseau. L'attaquant repond aux requetes de resolution et intercepte les hashes NTLMv2 des utilisateurs, qui peuvent ensuite etre casses hors ligne.

**Impact** : Un attaquant sur le reseau interne peut intercepter des credentials et obtenir un acces authentifie au domaine Active Directory. Si les mots de passe sont faibles, le craquage des hashes est rapide.

**Systemes affectes** : Ensemble du domaine `DOMAIN.LOCAL` (protocoles actifs sur tous les segments).

**Etapes de reproduction** :

```bash
# - Lancement de Responder sur l'interface reseau
sudo responder -I eth0 -wrfv
```

> Resultat : interception du hash NTLMv2 de l'utilisateur `jdupont` en moins de 5 minutes.

```bash
# - Craquage du hash avec Hashcat
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

> Resultat : mot de passe retrouve en 12 secondes.

**Recommandation** : Desactiver LLMNR et NBT-NS via GPO. Pour LLMNR : `Computer Configuration > Administrative Templates > Network > DNS Client > Turn Off Multicast Name Resolution` (activer). Pour NBT-NS : desactiver via DHCP ou directement dans les proprietes TCP/IP de chaque interface.

**References** :
- MITRE ATT&CK T1557.001
- Microsoft : desactivation de LLMNR via GPO

## Pieges et galeres

- **Finding generique copie-colle** : reutiliser un finding template sans l'adapter au contexte du client. Chaque finding doit mentionner les hotes specifiques et le contexte exact
- **Oublier l'impact business** : "un attaquant peut recuperer un hash" ne parle pas a la direction. "Un attaquant peut acceder aux donnees financieres de l'entreprise" a un impact bien plus concret
- **Preuves non reproductibles** : si l'equipe de remediation ne peut pas reproduire la vulnerabilite avec les etapes fournies, elle ne pourra pas verifier sa correction
- **Sortie brute sans contexte** : une sortie Hashcat de 50 lignes sans explication est inutile. Extraire les lignes significatives et expliquer ce qu'elles montrent
- **Mots de passe en clair dans le rapport** : toujours masquer les mots de passe et les hashes dans les captures et les sorties de commandes. Le rapport peut circuler largement

## Memo express

| Element | Detail |
|---|---|
| **Structure finding** | Titre, severite, description, impact, systemes, reproduction, preuves, recommandation, references |
| **Preuves** | Une etape par capture, narratif entre les images, contexte visible |
| **Recommandations** | Specifiques, actionnables, avec mesures palliatives et avertissements |
| **References** | Sources neutres, accessibles, perennes |
| **Credentials** | Toujours masquer dans le rapport |
| **Personnalisation** | Adapter chaque finding au contexte du client |

***
