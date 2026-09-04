# Extensions

## Pourquoi

Les fonctionnalites de base de Burp et ZAP couvrent la majorite des besoins, mais les extensions permettent d'aller plus loin : decodeurs supplementaires, scanners specialises, detecteurs de vulnerabilites specifiques (deserialization, CSRF, JWT), ou integration avec d'autres outils. Les deux ecosystemes sont actifs et proposent des centaines d'extensions.

## Comment ca marche

### Burp - BApp Store

Accessible depuis l'onglet `Extensions > BApp Store`. Les extensions sont classees par popularite. Certaines sont reservees a la version Pro, mais la majorite fonctionne en Community.

**Installation :** selectionner une extension, cliquer sur `Install`. Certaines necessitent des dependances (Jython pour les extensions Python, JRuby pour Ruby). Les installer avant si necessaire.

**Extensions recommandees :**

| Extension | Usage |
|---|---|
| Active Scan++ | Checks supplementaires pour le scanner actif |
| Autorize | Test automatique d'escalade de privileges |
| JS Link Finder | Extraction de liens depuis les fichiers JavaScript |
| Retire.JS | Detection de librairies JS vulnerables |
| JWT Editor | Manipulation et attaque de tokens JWT |
| Logger++ | Logging avance avec filtres |
| Decoder Improved | Encodeurs/decodeurs supplementaires + hash |
| Backslash Powered Scanner | Detection de comportements inhabituels du serveur |

### ZAP - Marketplace

Accessible via le bouton `Manage Add-ons` dans la barre d'outils, onglet `Marketplace`. Les add-ons sont classes par statut : `Release` (stable), `Beta`, `Alpha`.

**Add-ons utiles :**

| Add-on | Usage |
|---|---|
| FuzzDB Files | Wordlists supplementaires pour le fuzzer |
| FuzzDB Offensive | Payloads d'attaque specifiques (SQLi, XSS, command injection) |
| Access Control Testing | Tests d'autorisation automatises |
| Token Generator | Generation de tokens pour le fuzzing |
| Wappalyzer | Detection des technologies web utilisees |

{% hint style="info" %}
Apres installation de FuzzDB, les wordlists sont disponibles dans le Fuzzer sous `File Fuzzers > fuzzdb`. Par exemple, `fuzzdb > attack > os-cmd-execution` contient des payloads d'injection de commande.
{% endhint %}

## En pratique

### Exemple : Decoder Improved (Burp)

Apres installation, un nouvel onglet `Decoder Improved` apparait dans Burp. Il offre les memes fonctionnalites que le Decoder natif, plus des encodages supplementaires et un editeur hexadecimal.

Pour hasher une chaine en MD5 : entrer le texte, puis `Hash With > MD5`.

### Exemple : FuzzDB (ZAP)

Apres installation de FuzzDB Offensive, lancer une attaque de fuzzing et selectionner `File Fuzzers > fuzzdb > attack > os-cmd-execution > command_execution-unix.txt` comme payload. Le fuzzer teste automatiquement tous les payloads d'injection de commande de la liste.

## Pieges et galeres

- Les extensions tierces peuvent introduire des bugs ou des instabilites. En cas de probleme, desactiver les extensions une par une pour identifier la cause.
- Certaines extensions Burp necessitent Jython (Python) ou JRuby (Ruby) qui doivent etre configures dans `Extensions > Extension Settings`.
- Les extensions en version Alpha/Beta dans ZAP peuvent ne pas fonctionner correctement. Privilegier les extensions en `Release`.

## Retour terrain

Les extensions transforment un proxy web en plateforme de test complete. Autorize pour les tests d'autorisation, JWT Editor pour les tokens, et Active Scan++ pour un scan plus exhaustif sont des incontournables. Les installer au debut de l'engagement et les configurer une fois evite de perdre du temps par la suite.

## Memo express

| Ecosysteme | Acces | Extensions cles |
|---|---|---|
| Burp BApp Store | `Extensions > BApp Store` | Active Scan++, Autorize, JS Link Finder, Retire.JS |
| ZAP Marketplace | `Manage Add-ons > Marketplace` | FuzzDB, Wappalyzer, Access Control Testing |

***
