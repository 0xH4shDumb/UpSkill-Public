# Prise de notes et organisation

La prise de notes est le fondement de tout rapport de pentest. Des notes bien structurees, accompagnees de logs d'outils et de captures d'ecran, constituent la matiere premiere du livrable final. Cette page couvre la structure recommandee, les outils adaptes et les techniques de logging.

## Pourquoi

Des notes desordonnees ralentissent la redaction du rapport, augmentent le risque d'oublier des findings et rendent la reproduction des vulnerabilites difficile. A l'inverse, des notes structurees permettent de generer un rapport rapidement, de repondre aux questions du client et de former d'autres membres de l'equipe si necessaire.

## Comment ca marche

### Structure recommandee des notes

Chaque engagement devrait suivre une arborescence similaire, adaptee au type de test.

| Section | Contenu |
|---|---|
| **Chaine d'attaque** | Chemin complet de l'acces initial a la compromission finale (captures + commandes) |
| **Credentials** | Tous les identifiants recuperes pendant le test (centralises) |
| **Findings** | Un sous-dossier par finding avec le narratif, les captures et les preuves |
| **Recherche vuln scan** | Notes sur les resultats de scans de vulnerabilites, faux positifs exclus |
| **Recherche services** | Services investigues, tentatives echouees, pistes prometteuses |
| **Recherche web** | Applications web identifiees, technologies, credentials par defaut testes |
| **Enumeration AD** | Etapes d'enumeration Active Directory, chemins d'attaque identifies |
| **OSINT** | Informations collectees en sources ouvertes (si applicable) |
| **Informations admin** | Contacts, ROE, objectifs, liste de taches |
| **Scope** | Plages IP, URLs, credentials fournis par le client |
| **Journal d'activite** | Suivi chronologique de toutes les actions |
| **Journal des payloads** | Payloads utilises, hash de fichiers uploades, emplacements de depot |

{% hint style="info" %}
Le journal des payloads est souvent neglige. En fin d'engagement, le pentester doit nettoyer tous les artefacts deposes. Sans journal precis des emplacements et des noms de fichiers, certains payloads peuvent rester sur les systemes du client.
{% endhint %}

### Outils de prise de notes

Le choix de l'outil est personnel, mais certains criteres sont importants pour un usage professionnel.

| Outil | Stockage | Points forts |
|---|---|---|
| **Obsidian** | Local | Markdown, liens bidirectionnels, plugins, export facile |
| **CherryTree** | Local | Hierarchie arborescente, captures integrees, chiffrement |
| **Notion** | Cloud | Collaboratif, bases de donnees, templates avances |
| **Outline** | Cloud / Self-hosted | Markdown, collaboratif, version auto-hebergeable |
| **GitBook** | Cloud | Documentation structuree, publication web integree |

{% hint style="danger" %}
Pour les engagements clients, privilegier un outil local ou auto-heberge. Un outil cloud synchronise les donnees vers des serveurs tiers, ce qui peut violer les obligations contractuelles ou les politiques de l'entreprise. Verifier les regles de stockage avec le responsable d'equipe avant de choisir.
{% endhint %}

### Logging terminal

Toutes les sessions de terminal doivent etre enregistrees. Le logging Tmux capture chaque commande et sa sortie dans un fichier texte, ce qui constitue une preuve irrefutable des actions realisees.

#### Configuration du logging Tmux

```bash
# - Installation du gestionnaire de plugins Tmux
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

```bash
# - Configuration dans ~/.tmux.conf
cat > ~/.tmux.conf << 'EOF'
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-logging'

run '~/.tmux/plugins/tpm/tpm'
EOF
```

```bash
# - Charger la configuration
tmux source ~/.tmux.conf
```

Une fois configure, demarrer l'enregistrement dans un pane Tmux avec `Ctrl+b` puis `Shift+P`. Le meme raccourci arrete l'enregistrement. Le fichier log est cree dans le repertoire courant.

{% hint style="success" %}
La commande `script` est une alternative rapide si Tmux n'est pas disponible. Elle enregistre tout ce qui se passe dans le terminal courant vers un fichier.

```bash
script -a pentest_$(date +%Y%m%d_%H%M).log
```
{% endhint %}

### Organisation des fichiers sur la VM

L'arborescence de fichiers sur la machine d'attaque doit refleter la structure des notes.

```
engagement/
├── evidence/
│   ├── credentials/
│   ├── data/
│   └── screenshots/
├── findings/
│   ├── 01-kerberoasting/
│   ├── 02-default-creds-tomcat/
│   └── 03-llmnr-poisoning/
├── logs/
│   ├── tmux-20250101.log
│   └── nmap-scans/
├── scans/
│   ├── nmap/
│   ├── nessus/
│   └── gobuster/
├── scope/
│   └── in-scope-ips.txt
└── tools/
    └── scripts-custom/
```

## En pratique

### Workflow de documentation pendant un test

1. **Debut de journee** : ouvrir une session Tmux nommee, activer le logging, verifier le scope
2. **Pendant le test** : noter chaque action dans l'outil de notes, capturer les screenshots des resultats significatifs, enregistrer les credentials dans la section dediee
3. **A chaque finding** : creer un dossier dedie, rediger le narratif initial, sauvegarder les preuves
4. **Fin de journee** : synchroniser les notes et les logs vers le stockage externe, verifier que les captures sont lisibles

### Captures d'ecran efficaces

| Bonne pratique | Mauvaise pratique |
|---|---|
| Inclure la barre d'URL ou le prompt shell | Screenshot sans contexte (quelle machine ?) |
| Encadrer les elements importants (fleches, cadres) | Screenshot brut sans annotation |
| Utiliser un fond opaque (pas transparent) | Terminal transparent montrant le bureau |
| Prompt professionnel (`pentester@kali`) | Prompt type `hackerman@pwn3d` |
| Masquer les donnees sensibles | Credentials en clair dans les captures |

## Pieges et galeres

- **Notes dans un seul fichier** : un fichier de 200 pages devient impossible a naviguer. Separer par section et par finding
- **Screenshots sans contexte** : une capture d'ecran sans l'URL, le hostname ou le timestamp ne prouve rien. Toujours inclure un element identifiant
- **Oublier les tentatives echouees** : documenter ce qui n'a PAS fonctionne est aussi important que ce qui a fonctionne. Ca montre au client que le test a ete exhaustif
- **Pas de journal de payloads** : sans liste des fichiers deposes sur les systemes du client, le nettoyage de fin d'engagement est incomplet
- **Notes synchronisees vers le cloud sans autorisation** : verifier les obligations contractuelles avant d'utiliser un outil cloud pour les donnees client

## Memo express

| Element | Detail |
|---|---|
| **Structure notes** | Findings, credentials, journal d'activite, scope, logs |
| **Outil recommande** | Obsidian (local) ou Outline (self-hosted) pour les engagements clients |
| **Logging terminal** | Tmux logging ou `script` pour enregistrer toutes les sessions |
| **Arborescence VM** | evidence/, findings/, logs/, scans/, scope/, tools/ |
| **Screenshots** | Annotees, avec contexte (URL, hostname), prompt professionnel |
| **Sauvegarde** | Quotidienne vers un stockage externe |

***
