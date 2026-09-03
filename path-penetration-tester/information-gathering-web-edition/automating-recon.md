# Automatisation de la reconnaissance

## Pourquoi

La reconnaissance manuelle est efficace mais chronophage et sujette à l'erreur humaine. Quand on a plusieurs cibles à analyser ou qu'on veut garantir une couverture exhaustive, l'automatisation devient indispensable. Les frameworks de reconnaissance regroupent plusieurs techniques (DNS, WHOIS, crawling, scan de ports) dans un seul outil, permettant de gagner du temps tout en maintenant une cohérence dans les résultats.

## Comment ça marche

Les frameworks d'automatisation s'appuient sur des modules spécialisés qui exécutent chacun une technique de reconnaissance spécifique. Ils agrègent ensuite les résultats dans un format exploitable. Les avantages sont clairs :

- **Efficacité** : les tâches répétitives sont exécutées bien plus vite qu'à la main.
- **Scalabilité** : on peut lancer la reconnaissance sur un grand nombre de domaines simultanément.
- **Cohérence** : des procédures standardisées réduisent le risque d'oubli.
- **Couverture** : chaque module couvre un angle différent, la combinaison offre une vue globale.
- **Intégration** : les résultats peuvent alimenter d'autres outils de la chaîne d'attaque.

## En pratique

### Frameworks principaux

| Framework | Langage | Points forts |
|---|---|---|
| **FinalRecon** | Python | Modulaire, couvre headers/WHOIS/SSL/crawling/DNS/sous-domaines/Wayback |
| **Recon-ng** | Python | Architecture modulaire, large bibliothèque de modules, persistance des données |
| **theHarvester** | Python | Spécialisé dans la collecte d'emails, sous-domaines, hôtes et noms via sources publiques |
| **SpiderFoot** | Python | Intègre de nombreuses sources de données, interface web, automatisation poussée |
| **OSINT Framework** | Web | Collection d'outils et de ressources classés par catégorie, point de départ pour l'OSINT |

### FinalRecon en détail

FinalRecon est un framework tout-en-un qui couvre les aspects suivants :

| Module | Ce qu'il collecte |
|---|---|
| `--headers` | Informations de headers HTTP (serveur, technologies, configurations) |
| `--whois` | Détails d'enregistrement du domaine |
| `--sslinfo` | Informations sur le certificat SSL/TLS |
| `--crawl` | Liens (internes, externes), scripts JS, images, robots.txt, sitemap.xml |
| `--dns` | Plus de 40 types d'enregistrements DNS, dont les records DMARC |
| `--sub` | Sous-domaines via crt.sh, AnubisDB, ThreatMiner, CertSpotter et autres |
| `--dir` | Énumération de répertoires par wordlist |
| `--wayback` | URLs historiques des 5 dernières années via le Wayback Machine |
| `--ps` | Scan de ports rapide |
| `--full` | Exécute tous les modules |

#### Installation

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
```

#### Utilisation

```bash
# - Reconnaissance complète
./finalrecon.py --full --url http://<DOMAINE_CIBLE>

# - Headers + WHOIS uniquement
./finalrecon.py --headers --whois --url http://<DOMAINE_CIBLE>

# - Sous-domaines + DNS
./finalrecon.py --sub --dns --url http://<DOMAINE_CIBLE>
```

#### Options utiles

| Option | Rôle |
|---|---|
| `--url` | URL de la cible |
| `-T` | Timeout des requêtes (défaut : 30s) |
| `-w` | Wordlist personnalisée pour l'énumération de répertoires |
| `-e` | Extensions de fichiers à chercher (`txt,xml,php`) |
| `-o` | Format d'export (`txt` par défaut) |
| `-d` | Serveur DNS personnalisé (défaut : 1.1.1.1) |
| `-k` | Ajouter une clé API (Shodan, VirusTotal, etc.) |

### theHarvester

Outil spécialisé dans la collecte d'emails, sous-domaines et noms d'hôtes à partir de sources publiques (moteurs de recherche, serveurs PGP, Shodan) :

```bash
theHarvester -d <DOMAINE_CIBLE> -b google,bing,linkedin
```

### Recon-ng

Framework modulaire qui fonctionne comme un mini-Metasploit dédié à la reconnaissance. Il dispose d'un système de marketplace pour installer des modules communautaires et permet de persister les résultats dans une base de données locale.

### SpiderFoot

Outil d'automatisation OSINT avec interface web intégrée. Il interroge un grand nombre de sources de données et présente les résultats sous forme de graphe de relations entre les entités découvertes (domaines, IPs, emails, organisations).

## Pièges et galères

- **Bruit dans les résultats** : les outils automatisés génèrent beaucoup de données, dont une partie peut être des faux positifs. Le tri manuel reste indispensable.
- **Rate limiting et blocage** : les scans automatisés peuvent être détectés et bloqués par les WAF, les IDS ou les services interrogés (Google, crt.sh). Configurer des délais raisonnables entre les requêtes.
- **Clés API** : certains modules nécessitent des clés API (Shodan, VirusTotal, etc.) pour fonctionner. Sans clé, les résultats seront partiels.
- **Périmètre** : un outil lancé sans restriction peut explorer au-delà du périmètre autorisé. Toujours vérifier la configuration avant de lancer un scan.

## Retour terrain

L'automatisation ne remplace pas l'analyse humaine, elle la complète. Le workflow typique consiste à lancer un framework pour collecter un maximum de données brutes, puis à analyser manuellement les résultats pour identifier les pistes les plus prometteuses. Les résultats les plus intéressants sont souvent ceux qui semblent anodins individuellement mais prennent du sens quand on les croise.

## Mémo express

| Besoin | Commande |
|---|---|
| Recon complète (FinalRecon) | `./finalrecon.py --full --url http://<cible>` |
| Collecte d'emails (theHarvester) | `theHarvester -d <domaine> -b google,bing` |
| Scan modulaire (Recon-ng) | `recon-ng` puis `marketplace search`, `modules load` |

***
